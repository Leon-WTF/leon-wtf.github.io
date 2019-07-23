---
title: "Linux EXT系列文件系统格式"
category: Linux
tag: linux
---
### Linux文件系统 ###
![computer_hard_drive](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/computer_hard_drive.png)

常见的硬盘如上图所示，每个盘片分多个磁道，每个磁道分多个扇区，每个扇区512字节，是硬盘的最小存储单元，但是在操作系统层面会将多个扇区组成块（block），是操作系统存储数据的最小单元，通常是8个扇区组成4K字节的块。
对于Linux文件系统，需要考虑以下几点：
- 文件系统需要有严格的组织形式，使文件能够以块为单位存储
- 文件系统需要有索引区，方便查找一个文件分成的多个块存在了什么位置
- 如果有文件近期经常被读写，需要有缓存层
- 文件应该用文件夹的形式组织起来方便管理和查询
- Linux内核要在自己的内存里维护一套数据结构，保持哪些文件被哪些进程打开和使用
Linux里面一切皆文件，都有以下几种文件（从***ls -l***结果的第一位标识位可以看出来）：
- \- 表示普通文件
- d 表示文件夹
- c 表示字符设备文件
- b 表示块设备文件
- s 表示套接字socket文件
- l 表示软链接
### Inode和块存储 ###
下面就以EXT系列格式为例来看一下文件是如果存在硬盘上的。首先文件会被分成一个个的块，分散得存在硬盘上，就需要一个索引结构来帮助我们找到这些块以及记录文件的一些元信息，这就是inode，其中i代表index。inode数据结构如下：
```C
struct ext4_inode {
        __le16  i_mode;         /* File mode */
        __le16  i_uid;          /* Low 16 bits of Owner Uid */
        __le32  i_size_lo;      /* Size in bytes */
        __le32  i_atime;        /* Access time */
        __le32  i_ctime;        /* Inode Change time */
        __le32  i_mtime;        /* Modification time */
        __le32  i_dtime;        /* Deletion Time */
        __le16  i_gid;          /* Low 16 bits of Group Id */
        __le16  i_links_count;  /* Links count */
        __le32  i_blocks_lo;    /* Blocks count */
        __le32  i_flags;        /* File flags */
        union {
                struct {
                        __le32  l_i_version;
                } linux1;
                struct {
                        __u32  h_i_translator;
                } hurd1;
                struct {
                        __u32  m_i_reserved1;
                } masix1;
        } osd1;                         /* OS dependent 1 */
        __le32  i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
        __le32  i_generation;   /* File version (for NFS) */
        __le32  i_file_acl_lo;  /* File ACL */
        __le32  i_size_high;
        __le32  i_obso_faddr;   /* Obsoleted fragment address */
        union {
                struct {
                        __le16  l_i_blocks_high; /* were l_i_reserved1 */
                        __le16  l_i_file_acl_high;
                        __le16  l_i_uid_high;   /* these 2 fields */
                        __le16  l_i_gid_high;   /* were reserved2[0] */
                        __le16  l_i_checksum_lo;/* crc32c(uuid+inum+inode) LE */
                        __le16  l_i_reserved;
                } linux2;
                struct {
                        __le16  h_i_reserved1;  /* Obsoleted fragment number/size which are removed in ext4 */
                        __u16   h_i_mode_high;
                        __u16   h_i_uid_high;
                        __u16   h_i_gid_high;
                        __u32   h_i_author;
                } hurd2;
                struct {
                        __le16  h_i_reserved1;  /* Obsoleted fragment number/size which are removed in ext4 */
                        __le16  m_i_file_acl_high;
                        __u32   m_i_reserved2[2];
                } masix2;
        } osd2;                         /* OS dependent 2 */
        __le16  i_extra_isize;
        __le16  i_checksum_hi;  /* crc32c(uuid+inum+inode) BE */
        __le32  i_ctime_extra;  /* extra Change time      (nsec << 2 | epoch) */
        __le32  i_mtime_extra;  /* extra Modification time(nsec << 2 | epoch) */
        __le32  i_atime_extra;  /* extra Access time      (nsec << 2 | epoch) */
        __le32  i_crtime;       /* File Creation time */
        __le32  i_crtime_extra; /* extra FileCreationtime (nsec << 2 | epoch) */
        __le32  i_version_hi;   /* high 32 bits for 64-bit version */
        __le32  i_projid;       /* Project ID */
};
```
其中**__le32  i_block[EXT4_N_BLOCKS]**存储了到数据块的引用，EXT4_N_BLOCKS定义如下：
```C
#define	EXT4_NDIR_BLOCKS	12
#define	EXT4_IND_BLOCK	EXT4_NDIR_BLOCKS
#define	EXT4_DIND_BLOCK	(EXT4_IND_BLOCK	+ 1)
#define	EXT4_TIND_BLOCK	(EXT4_DIND_BLOCK + 1)
#define	EXT4_N_BLOCKS	(EXT4_TIND_BLOCK + 1)
```
在ext2和ext3中i_block前12项存储了直接到数据块的引用，第13项存储的是到间接块的引用，在间接块里存储着数据块的位置，以此类推，第14项里存储着二次间接快的位置，第15项里存储着三次间接块的位置,如下图所示：

![ext2_3_i_block](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/ext2_3_i_block.png)

不难看出，对于大文件，需要多次读取硬盘才能找到相应的块，在ext4中就提出了Extents Tree来解决这一问题，其核心思想就是把连续的块用开始位置加块的个数来表示，不再是一个一个去记录每一个块的位置，这样就能节约存储空间。首先，它将i_block中原来4*15=60字节的空间换成了一个extent header（ext4_extent_header）加4个extent entry（ext4_extent），因为ext4_extent_header和ext4_extent都是占用了12字节。ee_len中的第一个bit用来判断是否初始化，所以它还能存储最大32K个数，所以一个extent entry里最大可以存32K*4K=128M的数据，如果一个文件大于4*128M=512M或者这个文件被分散到多于4个不连续的块中存储，我们就需要扩展inode中的i_block结构。它的extent entry就要从ext4_extent被换成ext4_extent_idx结构体，它所指向的是一个块，有4K字节，除去header占用的12字节，还能存340个ext4_extent，最大可以存340*128M=42.5G的数据。可以看出这种索引结构在文件用连续的块存储时非常高效。
```C
struct ext4_extent_header {
    __le16	eh_magic;    /* ext4 extents标识：0xF30A */
    __le16	eh_entries;  /* 当前层级中有效节点的数目 */
    __le16	eh_max;      /* 当前层级中最大节点的数目 */
    __le16	eh_depth;    /* 当前层级在树中的深度，0为叶子节点，即数据节点，>0代表索引节点 */
    __le32	eh_generation; 
}
struct ext4_extent {
     __le32   ee_block;    /* extent的起始block逻辑序号 */
     __le16   ee_len;      /* extent包含的block个数 */
     __le16   ee_start_hi; /*extent起始block的物理地址的高16位 */
     __le32   ee_start_lo; /*extent起始block的物理地址的低32位 */
};//数据节点中的extent_body格式
struct ext4_extent_idx {
    __le32   ei_block;    /* 索引所覆盖的文件范围的起始block的逻辑序号 */
    __le32   ei_leaf_lo;  /* 存放下一级extents的block的物理地址的低32位 */     
    __le16   ei_leaf_hi;  /* 存放下一级extents的block的物理地址的高16位 */
     __u16   ei_unused;

};//索引节点中的extent_body格式
```
举一个/var/log/messages文件的例子如下图所示：

![ext4_extent_entry](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/ext4_extent_entry.png)

### inode位图和块位图 ###
硬盘上会有专门存放块数据的区域也会有存放inode的区域，但是当我们要新建一个文件时，就需要知道哪个inode区域和哪个块是空的，这就需要分别用一个块来存储inode位图和一个块来存储块位图，每一个bit为1表示占用，为0表示未占用。但是一个块最多有4K*8=32K个位，也就最多能表示32K个块的状态，所以需要让这些块组成一个块组，来搭出更大的系统。

### 硬链接和软链接 ###
硬链接与原文件共用一个inode，且inode不能跨文件系统，所以硬链接也不能跨文件系统。

![hard_link](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/hard_link.png)

软链接有自己inode，只是打开文件时是指向另外一个文件，所以可以跨文件系统且当原文件被删除后仍存在。

![soft_link](https://raw.githubusercontent.com/Leon-WTF/leon.github.io/master/img/soft_link.png)
