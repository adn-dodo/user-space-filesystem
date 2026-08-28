# UserFS Design

## 1. Purpose and scope

UserFS is a user-space filesystem implemented as a C library. It stores the
entire filesystem—metadata, allocation maps, directories, file contents and
journal records—inside one regular Linux file used as a disk image. Programs
call the API declared in `include/userfs.h`; UserFS does not modify the Linux
kernel and does not use FUSE or host directories to represent individual
UserFS objects.

The implementation provides formatting, mounting, persistent nested
directories, regular-file I/O, seeking, truncation, metadata, hard and symbolic
links, a redo journal, and a copy-backed memory-mapping interface.

## 2. Main design decisions

| Property | Value |
|---|---:|
| Image size | 16 MiB (`16,777,216` bytes) |
| Block size | 512 bytes |
| Total blocks | 32,768 |
| Maximum inodes | 512 |
| Inode size | 128 bytes |
| Directory-entry size | 64 bytes |
| Open descriptors per process | 32 |
| Active mappings per process | 32 |
| Maximum name | 31 characters |
| Maximum path | 255 characters |

The format has a fixed geometry. `ufs_format()` accepts exactly the configured
16 MiB size, and `ufs_mount()` rejects images whose size or superblock geometry
does not match this version. Fixed geometry keeps validation and block
addressing simple and makes corrupt or incompatible images easier to detect.

## 3. On-disk layout

| Block range | Count | Size | Purpose |
|---|---:|---:|---|
| `0` | 1 | 512 B | Superblock |
| `1` | 1 | 512 B | Inode bitmap |
| `2–9` | 8 | 4 KiB | Whole-image block bitmap |
| `10–137` | 128 | 64 KiB | Inode table |
| `138–2185` | 2,048 | 1 MiB | Redo journal |
| `2186–32767` | 30,582 | 14.93 MiB | File, directory and pointer blocks |

The block bitmap contains 32,768 bits—one bit per image block. During format,
blocks `0–2185` are marked reserved, so allocation can return only blocks in
the data region. The inode bitmap uses one bit per inode; inode `0` is reserved
for the root directory, leaving 511 initially free inodes.

The root inode initially has size zero and no data block. Its first directory
block is allocated when the first root entry is created.

## 4. Persistent structures

### 4.1 Superblock

The superblock occupies exactly one 512-byte block. It stores:

- magic value `0x55465331` (`UFS1`) and on-disk format identifier `2` (the final submission format);
- block size and total image blocks;
- the start and length of every disk region;
- total/free inode and data-block counters;
- the root inode number;
- journal location and sequence information;
- clean/dirty mount state;
- feature flags for journaling, metadata and links.

Mount validates the magic, version, image size and every fixed layout field
before accepting the image. The filesystem is marked dirty while mounted and
clean after a successful unmount.

### 4.2 Inodes

Each inode is exactly 128 bytes, so four inodes fit in each inode-table block.
An inode stores:

- object type: regular file, directory or symbolic link;
- flags, permission mode, owner UID and owner GID;
- persistent hard-link count;
- 64-bit file size and logical data-block count;
- eight direct block pointers;
- one single-indirect and one double-indirect pointer;
- an inode generation number;
- access, modification and status-change timestamps.

Every unused block pointer is `UFS_INVALID_BLOCK` (`UINT32_MAX`), not zero,
because block zero is a valid image block occupied by the superblock. The inode
generation changes when an inode slot is reused, allowing descriptors and
mappings to detect stale references.

### 4.3 Directory entries

A directory's data blocks contain fixed-size 64-byte records. Each record
stores a used flag, inode number, object type and a null-terminated name of up
to 31 characters. One 512-byte block therefore contains eight entries.

The current directory implementation uses the inode's eight direct pointers,
giving a maximum of 64 entries in one directory. Deleted entries are reusable.
Directory `size` is the allocated directory capacity in bytes, so it is a
multiple of 512 and is not the sum of filename lengths.

### 4.4 Timestamps

Each persistent timestamp is 16 bytes: signed seconds, nanoseconds and a
reserved field. This fixed-width representation avoids depending on the host
layout of `struct timespec` inside the disk image.

Compile-time `_Static_assert` checks guarantee the intended sizes of the
superblock, inode, timestamp and directory-entry structures.

## 5. Allocation strategy

### 5.1 Inode allocation

The allocator scans the inode bitmap for a clear bit, reserves that inode,
initializes it with invalid pointers, and decreases `free_inodes`. Releasing an
inode clears its bitmap bit and makes the slot reusable. Inode `0` is never
released.

### 5.2 Block allocation

The data-block allocator performs first-fit scanning from block `2186`. A new
block is marked allocated, zeroed before use, and deducted from `free_blocks`.
Freeing reverses the bitmap and counter changes. Metadata and journal blocks
are outside the allocator's valid range and can never be returned.

### 5.3 Logical-to-physical mapping

Regular files use three addressing levels:

| Logical range | Mechanism | Capacity |
|---|---|---:|
| `0–7` | Eight direct pointers | 8 blocks |
| `8–135` | Single-indirect block | 128 blocks |
| `136–16519` | Double-indirect tree | 16,384 blocks |

Each pointer is 32 bits, so one 512-byte pointer block stores 128 addresses.
The theoretical maximum file size is therefore:

```text
(8 + 128 + 128 × 128) × 512 = 8,458,240 bytes
```

Indirect pointer blocks are themselves allocated from the data region and are
tracked by the block bitmap. Shrinking a file releases unnecessary data blocks
and then releases empty indirect blocks. New file ranges are zero-filled.

## 6. Namespace and path resolution

The public API uses absolute paths. Path processing enforces the configured
path and component limits and rejects malformed components. The root path `/`
resolves directly to inode `0`. Resolution walks each directory entry until it
reaches the requested object; attempting to traverse through a non-directory
returns `ENOTDIR`.

Namespace operations enforce the following rules:

- duplicate names return `EEXIST`;
- missing paths return `ENOENT`;
- `/` cannot be removed;
- `ufs_unlink()` rejects directories;
- `ufs_rmdir()` rejects regular files and non-empty directories;
- an open or actively mapped inode cannot be removed;
- allocation failures do not leave a successfully visible partial object.

### 6.1 Hard links

`ufs_link()` creates another directory entry referring to the same inode and
increments its `link_count`. `ufs_unlink()` removes one name and decrements the
count. File data and the inode are released only when the count reaches zero.
Directory hard links are prohibited to prevent directory cycles.

### 6.2 Symbolic links

A symbolic link has type `UFS_TYPE_SYMLINK`; its target text is stored in a
UserFS data block. Targets may be absolute or relative to the link's parent.
Normal path resolution follows symbolic links, while `ufs_lstat()` and
`ufs_readlink()` inspect the final link itself. Resolution permits at most 40
link expansions and returns `ELOOP` for a cycle or excessive chain.

## 7. File descriptors and I/O

Open descriptors exist only in process memory; they are not part of the disk
format. Each of the 32 slots records whether it is active, its inode number and
generation, its independent byte offset, and its access/append flags.

- `ufs_open()` validates the path, type, flags and requested permissions.
- `ufs_read()` stops at EOF, reads across block boundaries and advances the
  descriptor offset.
- `ufs_write()` overwrites or extends a file, allocates blocks when necessary,
  handles block boundaries, zero-fills a seek-created hole and advances the
  offset.
- append mode chooses the current EOF for every write, not only at open time.
- `ufs_seek()` supports `SEEK_SET`, `SEEK_CUR` and `SEEK_END`; seeking beyond
  EOF does not allocate blocks or change file size.
- `ufs_truncate()` grows with zeros or shrinks while releasing blocks.

The implementation reports Linux-style errors through `errno`, including
`ENOENT`, `EEXIST`, `ENOSPC`, `ENOTDIR`, `EISDIR`, `ENOTEMPTY`, `EINVAL`,
`EBADF`, `EACCES`, `EMFILE`, `EBUSY` and `ELOOP`.

## 8. Permissions, ownership and metadata

New regular files use `0666 & ~umask`; new directories use
`0777 & ~umask`. The initial UserFS umask is `0022`. Access selection checks
owner bits first, then group bits—including supplementary groups—and finally
other bits. Effective UID `0` bypasses permission checks as a project
simplification.

The public metadata API includes `ufs_chmod()`, `ufs_chown()`, `ufs_access()`
and `ufs_utimens()`. Creation initializes all three timestamps. Reads apply a
relatime-style access-time update. Writes and truncation update modification
and change times; metadata changes update change time. `ufs_stat()` follows a
final symbolic link, whereas `ufs_lstat()` reports the link itself.

Permission enforcement is implemented for file open/truncate and explicit
metadata access operations. This version does not claim complete POSIX
permission enforcement on every intermediate directory traversal and every
namespace mutation.

## 9. Redo journal and recovery

The journal uses physical-block after-images. A transaction can stage up to
124 unique target blocks; staging the same target again replaces its earlier
after-image. The commit protocol is:

1. write the journal header and complete 512-byte after-images;
2. call `fsync()`;
3. write the commit record and call `fsync()`;
4. copy the after-images to their home blocks and call `fsync()`;
5. clear the header and commit blocks and call `fsync()`.

At mount, UserFS checks the journal header, commit sequence, target ranges and
payload checksum. A valid committed transaction is replayed. An incomplete,
invalid or checksum-failing transaction is discarded. Replaying full block
after-images is idempotent, so recovery is safe if a crash happens during an
earlier replay.

The journal primitives and inode-metadata updates use this mechanism. The
allocator and some multi-block namespace/file operations retain explicit
rollback/direct-write logic, so this submission does not claim that every
public mutation is one fully atomic journal transaction.

## 10. Copy-backed memory mapping

UserFS descriptors are library handles rather than Linux kernel file
descriptors, and a UserFS file may occupy non-contiguous image blocks. They
therefore cannot be passed directly to the operating system's file-backed
`mmap()`.

`ufs_mmap()` instead creates anonymous Linux memory and copies the requested
UserFS range into it without changing the descriptor offset:

- a private mapping never modifies the file;
- `ufs_msync()` copies a writable shared mapping back to the same inode range;
- synchronizing past EOF extends the file;
- `ufs_munmap()` synchronizes a writable shared mapping before releasing it;
- an inode-generation check prevents a stale mapping from writing to a reused
  inode;
- unmounting with active mappings, or unlinking a mapped inode, returns
  `EBUSY`.

This is an explicit, copy-backed UserFS mapping abstraction. It deliberately
does not claim kernel page-fault-backed POSIX `mmap`, automatic dirty-page
tracking, or automatic coherence with unrelated writes.

## 11. Synchronization model

The synchronization module combines a process-local POSIX read/write lock with
whole-image advisory `fcntl` locks. The metadata API uses these locks, and the
mapping table has its own mutex. The library maintains one global mounted
filesystem context per process.

These mechanisms are a foundation for coordinated access, but the current
submission does not claim complete multi-threaded or multi-process POSIX
semantics across every public operation. In particular, descriptors are
process-local and not stored in the image.

## 12. Important invariants

1. Block `0` contains a valid final-format superblock for every mountable image.
2. Inode `0` always represents `/` and is never placed on the free list.
3. Blocks below `2186` are permanently reserved and never returned by the data
   allocator.
4. A set inode-bitmap bit means that its inode slot is allocated.
5. A set block-bitmap bit means that the corresponding block is reserved or
   allocated.
6. Every live data or pointer address lies inside the data region.
7. Every unused inode pointer equals `UFS_INVALID_BLOCK`.
8. A regular file's logical data-block count equals `ceil(size / 512)`; pointer
   blocks are additional allocation metadata.
9. A directory entry marked used refers to an allocated inode.
10. An inode remains allocated while its hard-link count is nonzero.
11. Descriptor and mapping generations must match the current inode generation.
12. Successful operations remain visible after unmounting and remounting the
    same image.
13. Newly allocated blocks and newly exposed file ranges read as zero.
14. Capacity-growing operations return `ENOSPC` without intentionally exposing
    a partially created namespace object.

## 13. Testing strategy

The test suite is divided by subsystem:

| Test | Main coverage |
|---|---|
| `test_integration.c` | Lifecycle, namespace, I/O, seek, truncate, errors and persistence |
| `test_storage_v2.c` | Direct, single-indirect and double-indirect allocation |
| `test_journal.c` | Commit, replay, incomplete transactions and persistence |
| `test_metadata.c` | Permissions, ownership, timestamps and metadata persistence |
| `test_links.c` | Hard links, symbolic links, loops and remount behavior |
| `test_mmap.c` | Shared/private mappings, synchronization, errors and persistence |
| `test_permissions_namespace.c` | Permission enforcement and namespace access errors |
| `test_crash_recovery.c` | Journal recovery after simulated interruption |
| `test_threads.c` | Concurrent access from multiple threads |
| `test_processes.c` | Advisory locking and access from multiple processes |

The project is compiled with strict warnings (`-Wall -Wextra -Werror
-pedantic`) and is also tested with AddressSanitizer and UndefinedBehaviorSanitizer.

## 14. Deliberate limits

- One fixed 16 MiB image geometry and one mounted image per process.
- At most 512 allocated inodes, 32 open descriptors and 32 active mappings.
- At most 64 entries in any single directory in the current implementation.
- No native kernel-backed `mmap`; UserFS mapping is copy-backed.
- No transparent sparse-file representation: gaps are physically zero-filled
  when a later write extends the file.
- No full POSIX multi-process descriptor or cache-coherency model.
- Journal transactions hold at most 124 unique block after-images, and full
  atomic coverage of every allocator/namespace mutation is not claimed.

These limits keep the design understandable while preserving the core
filesystem concepts required by the project: persistent metadata, allocation,
directories, block mapping, descriptors, offsets, error handling and recovery.
