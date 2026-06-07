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


Operations of superblock `struct super_operations`: alloc_inode, destroy_inode, put_super, statfs, remount_fs, etc


### Example of structure `struct superblock`:
```c
struct super_block {
    dev_t           s_dev;        // Device identifier (e.g., /dev/sda1)
    unsigned long   s_blocksize;  // Block size in bytes
    unsigned char   s_blocksize_bits;
    loff_t          s_maxbytes;   // Max file size
    struct file_system_type *s_type; // Points to "ext4", "tmpfs" etc.
    const struct super_operations *s_op; // FS-specific callbacks
    struct dentry   *s_root;      // Root dentry of this mount
    struct list_head s_inodes;    // All inodes of this FS
    void            *s_fs_info;   // FS-private data (ext4_sb_info*)
    // ... more fields
};
```

## 2. INODE : `struct inode`

An inode represents a file or directory as a kernel object - it's existence, it's metadata, it's location.

`Note:` The inode knows nothing about the filename.


`Analogy:` A file is like a person. The inode is the birth certificate which has the person's identity,attributes and vital info. The filename is name tag on a desk which points to the person, but the person exists independently of what's written on tag. Likewise the file exists irrespective of names given to it, actual file is the inode.


### Example structure of `struct inode`:

```c
struct inode {
    umode_t         i_mode;     // Permissions + type (file/dir/symlink)
    uid_t           i_uid;      // Owner user ID
    gid_t           i_gid;      // Owner group ID
    loff_t          i_size;     // File size in bytes
    struct timespec i_atime;    // Last access time
    struct timespec i_mtime;    // Last modification time
    struct timespec i_ctime;    // Last status change time
    unsigned long   i_ino;      // Inode number
    unsigned int    i_nlink;    // Number of hard links
    dev_t           i_rdev;     // Device ID (for device files)
    const struct inode_operations *i_op;  // FS callbacks for inode ops
    const struct file_operations  *i_fop; // FS callbacks for file ops
    struct super_block *i_sb;   // Which superblock (FS) owns this
    struct address_space *i_mapping; // Page cache mapping
    void            *i_private; // FS-private data (ext4_inode_info*)
    // ...
};
```

We can also view the inode number of any file using the command:
`ls -li example.txt`

### So how does inode get's created? - The connection between ext4's on-disk inode and VFS inode structure.

When a process opens or queries a file via system calls like read(), write(), open(), stat() or any, the VFS copies the on-disk metadata stored in inode table of the file-system (ext4) and loads it into the memory(RAM). Here the kernel creates the VFS inode, which wraps the filesystem specific inode. The object stays alive in the kernel's **inode cache** so subsequent lookups do not need slow disk access.


### Why does linux not store filename in the inode structure?

An inode does not store the filename because linux separate's a file's identity from it's name. The inode stores metadata such as permissions, ownership, size, timestamps, and data block pointers, while directory entries store mapping from filename to inode number. This design allows multiple filenames(hard links) to point the same inode. If filenames were stored inside the inode, there would be no single filename to store. Separating names from the inodes also makes rename operations very efficient because only the directory entry changes while the inode remains unchanged.

`Quick try:` Execute this on your terminal.

```bash
mkdir folderA
mkdir folderB

touch folderA/example.txt

ln folderA/example.txt folderB/report.txt

ls -li folderA/
ls -li folderB/

```

Above execution will create 2 folders, One file inside the folderA. The `ln` command will create a hard link between the specified files(The folderB/report.txt will be newly created).

On exectuing `ls -li folderA/` and `ls -li folderB/` you can see both the files have same inode number.

This is called `hard link` between 2 files pointing to same inode block.

`Note` : Change in either file will change have changes in other file because it is actually a pointer to same inode. This means copying operation is different the hard links.


## 3. Dentry (Directory Entry) : `struct dentry`

Dentry connects a file name to its underlying data, represented by an inode.

It acts as a bridge to translate human-readable file paths into the system's internal file identifiers.

Unlike inodes or file data, dentries are strictly memory based and created on-the-fly when a file path is being resolved and destroyed when no longer in use.

### The problem dentry solves"

Consider the example below:

Every time you access `/home/docs/file.txt`, the kernel needs to :

1. Find the inode for `/`
2. Look inside it for `home`, find inode for `home`
3. Look inside `home` for `docs`, find inode for `docs`
4. Look inside `docs` for `file.txt`, find inode for `file.txt`

Each "look inside" step potentially means a disk read (reading directory block from ext4).

Let's visualize the **look inside**.

Starting off with `/`, inside the inode of `/` (each inode which is of type directory) there is a block haveing the `directory entries` or the mappings (directory name -> inode).

So if the dentry for /home is not inside the dcache (cache miss),then VFS asks the ext4 is asked to lookup (parent dentry, d_name), essentially doing `inode->i_op->lookup(...)`, in our case (/, home) and a d_inode or the pointer will be returned to the inode and this inode will be loaded in-memory.

```
How ext4 looks up for this?

Each directory inode contains essentially :

inode 18278239
--------------
type = directory

i_block[]
|
|
+----> block 5000

Now this block 5000 contains all the mappings inside '/' (struct ext4_dir_entry_2)

Like:

Block 5000

inode=100
name="home"

inode=101
name="etc"

inode=102
name="var"
.
.
.

In this block, ext4 finds the name=home and inode=100 and returns this inode to the kernel and kernel will load that inode.

And VFS will now start with the /home inode and further lookup happens.

```

This way pathname resolution happens for each component in the path with the help of dentries.

### Example structure of `struct dentry`:
```c
struct dentry {
    struct inode        *d_inode;    // The inode this name maps to
    struct dentry       *d_parent;   // Parent dentry
    struct qstr          d_name;     // The name component ("harsh")
    struct super_block  *d_sb;       // Which FS this belongs to
    const struct dentry_operations *d_op; // FS callbacks
    struct list_head     d_child;    // Siblings in parent
    struct list_head     d_subdirs;  // Children
    // ...
};
```


## 4. File Object: `struct file`

The VFS file object `struct file` represents an open instance of a file.

It is created by open() and contains the states specific to that open operation, such as current file offset(f_pos), open flags (O_RDONLY, O_APPEND, etc) and pointers to file operations.

Unlike inode, which represents the file itself and is shared system-wide, each open call typically creates a separate file object.

File descriptors in a process point to file objects, and system calls like read(), write(), and lseek() operate through the file object.


### Why can't we use inode directly, instead of file objects?

The soul reason behind file object is that it is for each seaparate process, while the inodes are system wide maintained and accessible by each process, while this file object is only for respective process.

Imagine two opens, opening same file and reading.

```c

fd1 = open("notes.txt", O_RDONLY);
fd2 = open("notes.txt", O_RDONLY);

read(fd1, buf, 5);

//kernel reads : Hello and f_pos for fd1 is now 5
//But the f_pos for fd2 still remains unchanged i.e. 0

```

So that each process can access the file in shared manner simultaenously without duplication, file objects are useful in such cases.

### Example structure of `struct file`:

```c
struct file {
    struct path         f_path;      // Contains dentry + mount info
    struct inode        *f_inode;    // Pointer to inode
    const struct file_operations *f_op; // FS callbacks for this file
    loff_t              f_pos;       // Current offset (seek position)
    unsigned int        f_flags;     // O_RDONLY, O_NONBLOCK, etc.
    fmode_t             f_mode;      // FMODE_READ, FMODE_WRITE
    atomic_long_t       f_count;     // Reference count
    struct fown_struct  f_owner;     // For signals (async I/O)
    // ...
};
```