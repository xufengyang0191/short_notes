# lesson1: Abstraction

## 1. File Abstractions

在 UNIX 下面，File = bytes：Up to the user to interpret its content

10 个 essential syscalls:

- open/close/(creat)
- read/write/lseek(long seek)/(tell)
- fstat(file status)/ftruncate(file truncate)
- unlink/mkdir/dup

## 2. File System Abstraction

```
Map<string, string> // from filename to content
```

最核心的就是映射关系，上述 MAP 是抽象的映射，文件名对文件内容的映射，但是上述映射关系过于抽象，那么：

```
/* or less-abstract */

struct File {
    mode (permissions, type), size, user, group,
    timestamps (atime, mtime, ctime),
    content (bytes)
}
```

那么上面的抽象就可以具象一点：

```
Map<string, File> // from file path to File
```

可以把文件系统当成磁盘上的数据结构，它实现的就是文件路径到文件的映射，这也是 File system 的最核心的最核心的抽象

Unix 中使用 inode 把目录和文件联系起来

```
typedef uint32-t inode_num_t; // unique within a file system

Map<string, inode_num_t> dirs; // map file path to inode number

Map<inode_num_t, string> files; // map inode number to connect
/* or */
Map<inode_num_t, File> files; // each inode is a file

```

根据 key-value 的特性可以知道：Two or more directory entries could point to the same inode.

## 3. Disk Abstraction: block device

```
// Physical
class Disk // read-write in sectors
{
    // Logical block addressing (LBA)
    readSector(long n, void* buf);
    writeSector(long n, const void* buf);
};
```

Typical sector is 512B in old times, usually 4096B for new disks since 2010s.(Enterprise SAS disksmay have 520/528 "fat" sectors.)

Older disks use cylinder/head/sector (CHS). 3D cylindrical coordinate system.

下面对比逻辑抽象

```
// Logical
class BlockDevice
{
    // Linear space of contiguous blocks
    readBlock(long n, void* buf);
    writeBlock(long n, const void* buf);
};
```

Typical block is 1KiB in old times. Default for all slides.

Can also be 2KiB or 4KiB.4KiB is more common recently.

8KiB blocks are possible on some platforms (DEC Alpha).

上面我们讲述了文件系统可以抽象成为一个块设备，那么文件系统的作用呢，就是把一个块设备编程一个文件的 API。(Filesystem turns a block device into files)

```
class Filesystem
{
    explicit FileSystem(BlockDevice* dev); // 构造函数，把一个 BlockDevice 构造成一个 FileSystem

    File* Open(string filename, int mode);
    // delete File* means close it;

    read(File*, void*buf, int len);
    write(File*, const void* buf, int len);
    lseek(File*, offset，whence);
    
    unlink(string filename );
    mkdir(string path);
}
```
