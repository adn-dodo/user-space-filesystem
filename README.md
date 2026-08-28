# UserFS — User-Space Filesystem

A filesystem implemented in C that runs entirely in user space and stores its data and metadata inside a disk image.

UserFS was developed to explore how filesystems work internally, including block allocation, inodes, directories, path resolution, file descriptors, persistent file I/O, permissions, links, and consistency.

Unlike FUSE-based filesystems, UserFS is implemented as a C library. Applications interact directly with the filesystem through the UserFS API.

## Features

### Files and Directories
- Create and remove files and directories
- Open and close files
- Read and write file contents
- Seek within files
- Truncate files
- List directory contents
- Query file metadata

### Filesystem Internals
- Disk-image backed storage
- Block allocation using bitmaps
- Inode allocation and management
- Path traversal and resolution
- File descriptor management
- Persistent metadata across mount/unmount
- Direct, single-indirect, and double-indirect block addressing

### Extended Functionality
- Hard links
- Symbolic links
- File permissions and ownership
- Timestamps
- Journaling and crash recovery
- Copy-backed memory mapping
- Synchronization support

## Architecture

UserFS is divided into several components:

```text
Application / Tests
        |
        v
   UserFS Public API
        |
        +--------------------+
        |                    |
        v                    v
   File Operations      Namespace Operations
        |                    |
        +---------+----------+
                  |
                  v
           Inode Management
                  |
                  v
           Block Allocation
                  |
                  v
             Disk Layer
                  |
                  v
              Disk Image
```

The filesystem is accessed through a public C API rather than through the Linux kernel or FUSE.

## Repository Structure

```text
user-space-filesystem/
├── include/        Public and internal headers
├── src/            Filesystem implementation
├── apps/           Example/test applications
├── tests/          Test programs
├── DESIGN.md       Detailed filesystem design
├── Makefile        Build configuration
└── README.md
```

## Core Concepts Practiced

This project provided practical experience with:

- Filesystem architecture
- Block-based storage
- Inodes
- Allocation bitmaps
- File descriptors
- File offsets
- Directory entries
- Path traversal
- Persistent metadata
- Hard and symbolic links
- File permissions
- Journaling
- Crash consistency
- Synchronization
- C systems programming

## Building

Clone the repository and build it using:

```bash
make
```

The included Makefile builds the filesystem implementation and its associated programs/tests.

## Design Documentation

A more detailed explanation of the filesystem layout, data structures, allocation strategy, file I/O behavior, and implementation decisions is available in:

[`DESIGN.md`](DESIGN.md)


## Acknowledgments

UserFS was developed as a team project as part of the **Introduction to Linux and Embedded Systems** training at STMicroelectronics, within the **Linux System Programming** track, under the guidance of **Eng. Reda Maher**.
## Purpose

UserFS was built as a hands-on systems programming project to understand what happens internally when applications create directories, open files, read or write data, allocate storage, resolve paths, and persist filesystem metadata.

The project focuses on implementing these mechanisms directly rather than relying on an existing filesystem framework.
