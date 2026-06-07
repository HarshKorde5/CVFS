# NOTES

## What is a virtual file system in linux?

- The Virtual File System, also known as Virtual Filesystem Switch, is the software layer in the Linux Kernel that provides a uniform interface to the userspace programs to interact with different types of file systems.
- Think of it as a "translator" that allows applications to use standard commands like open(), read(), or write() without needing to know if the file is stored on a local disk(ext4), a Windows-formatted drive(NTFS), or even a remote network server(NFS).
- Virtual filesystems are the magic abstraction that makes the "everything is a file" philosophy of Linux possible.

**Core purpose and benefits** :
- It hides the complex implementation details of specific file systems from applications.
- It provides single set of system calls that work across all supported file systems.
- It allows multiple different file systems to exist simultaneously in a namespace.


**Quick example** : Format a pendrive in Linux,plug it in Windows and view the contents of the pendrive, Windows won't show anything. Now Format the pendrive on Windows and plug it in Linux, you can see contents. This is power of VFS, it can read any supported filesystem contents.

### Virtual File System(VFS) VS File-System(FS) :
- The core difference is that a Virtual File System(VFS) is an abstract software layer that manages multiple file systems, while a Filesystem(like NTFS or ext4) is the specific method used to organize and store data on a physical or logical device.


## How VFS and FS works together?
1. When a program calls a command like read() or write(), it talks to the VFS. The VFS doesn't know where the data is on the disk; it just knows which specific Filesystem(FS) is responsible for that path.
2. The VFS translates the generic system call into a specific operation that the underlying filesystem (eg. ext4) understands.

![VFS architecture](/cvfs_01.png)

## Components of VFS:

1. SUPERBLOCK
2. INODE
3. DENTRY
4. FILE OBJECT 

------------

## 1. SUPERBLOCK : `struct super_block`

A superblock represents state and global attributes of a mounted filesystem instance.

It holds filesystem wide data like block sizes, maximum file size, and flags.

When you mount ext4 at /home, VFS creates a superblock object in memory for that mount.

`Analogy` : If we consider the file system as a library, the superblock is the library cataloge system - it knows how many books exist, where to find them, the library rules, etc.

**Are there 2 superblocks? The VFS one and the FS one?**

Yes, there are these 2 superblocks one belongs to the actual filesystem (eg.ext4) and another is the VFS struct super_block.

How these two superblocks relate is; The VFS superblock resides on RAM throughout its lifetime as it is a part of the VFS, and the FS superblock resides on the disk.

When you run mount() system call, the actual FS reads its on-disk superblock, then populates the VFS struch super_block in RAM. From this point onwards the VFS uses only the RAM version.


`Note` : Whenever the kernel needs to read or write a file, it resolves the path to locate the target inode, and every inode contains a pointer back to it's corresponding superblock. This ensures the kernel knows exactly which file system driver to route the I/O requests through.


Operations of superblock (s_op):alloc_inode, destroy_inode, put_super, statfs, remount_fs, etc




