# Ext2 Filesystem

### 1. Ext2 Disk Layout

<figure><img src="../.gitbook/assets/ext2_layout.jpg" alt=""><figcaption></figcaption></figure>

ext2文件系统的磁盘layout如上图所示：

其中前1024字节是启动block，给系统引导扇区预留的，ext2不会去使用这一块数据。剩下的空间被分成了N个block group，其中每个block group的结构都是一样的。

superblock的定义如下：

```clike
struct ext2_superblock {
    uint32_t s_inodes_count;       // total inodes count
    uint32_t s_blocks_count;       // total blocks count
    uint32_t s_r_blocks_count;     // root reserved blocks
    uint32_t s_free_blocks_count;  // free blocks count
    uint32_t s_free_inodes_count;  // free inodes count
    uint32_t s_first_data_block;   // Block number of the block containing the superblock
    uint32_t s_log_block_size;
    uint32_t s_log_frag_size;
    uint32_t s_blocks_per_group;
    uint32_t s_frags_per_group;
    uint32_t s_inodes_per_group;
    uint32_t s_mtime;  // Last mount time
    uint32_t s_wtime;  // Last written time

    uint16_t s_mnt_count;        // Number of times the volume has been mounted since its last consistency check
    uint16_t s_max_mnt_count;    // Number of mounts allowed before a consistency check must be done
    uint16_t s_magic;            // Ext2 signature (0xef53)
    uint16_t s_state;            // File system state
    uint16_t s_errors;           // What to do when an error is detected
    uint16_t s_minor_rev_level;  // Minor portion of version

    uint32_t s_lastcheck;      // POSIX time of last consistency check
    uint32_t s_checkinterval;  // Interval (in POSIX time) between forced consistency checks
    uint32_t s_creator_os;     // Operating system ID from which the filesystem on this volume was created
    uint32_t s_rev_level;      // Major portion of version

    uint16_t s_def_resuid;  // User ID that can use reserved blocks
    uint16_t s_def_resgid;  // Group ID that can use reserved blocks

    uint32_t s_first_ino;
    uint16_t s_inode_size;
}
```

\
block group起始都是一个superblock，记录了ext2自己的一些信息，比如block size，inode size等等信息。ext2通过读取superblock计算blocksize大小，blocksize=1024 << superblock->s\_log\_block\_size。

接着是group descriptor列表，group descriptor记载了block group元信息，superblock和group descriptor除了第一个block group有之外，其他block group也可能会有一份冗余，包括block bitmap,inode bitmap,inodetablet等结构。

```clike
struct ext2_group_desc {
    uint32_t bg_block_bitmap;       // Block address of block usage bitmap
    uint32_t bg_inode_bitmap;       // Block address of inode usage bitmap
    uint32_t bg_inode_table;        // Starting block address of inode table
    uint16_t bg_free_blocks_count;  // Number of unallocated blocks in group
    uint16_t bg_free_inodes_count;  // Number of unallocated inodes in group
    uint16_t bg_used_dirs_count;    // Number of directories in group
    uint16_t bg_pad;
    uint8_t  bg_reserved[12];
};
```

group descriptor后面是Reserved GDT blocks，是用来文件系统在线resize的，不是所有的block group都有。

然后后面分别是block\_bitmap和inode\_bitmap，block\_bitmap和inode\_bitmap分别表示本block group的block和inode使用情况，0代表block或者inode未被使用，1代表已经使用了，他们的长度分别是1k，这样一个block group最多有8\*1024个block或者inode。

接着是inodetable，inodetable记录了block group预定义的所有的inode，而block\_bitmap，inode\_bitmap，inodetable的offset记录在group descriptor里。

最后面是datablocks，用来存储实际的数据。

我们可以用dumpe2fs来解析一份实际的文件系统layout，如下所示

```
Filesystem volume name:   <none>
Last mounted on:          /root/disk_1
Filesystem UUID:          11befedf-951f-4041-99b2-02eff14583fb
Filesystem magic number:  0xEF53
Filesystem revision #:    1 (dynamic)
Filesystem features:      ext_attr resize_inode dir_index filetype sparse_super
Filesystem flags:         signed_directory_hash 
Default mount options:    user_xattr acl
Filesystem state:         not clean
Errors behavior:          Continue
Filesystem OS type:       Linux
Inode count:              32768
Block count:              131072
Reserved block count:     6553
Free blocks:              125377
Free inodes:              32753
First block:              1
Block size:               1024
Fragment size:            1024
Reserved GDT blocks:      256
Blocks per group:         8192
Fragments per group:      8192
Inodes per group:         2048
Inode blocks per group:   256
Filesystem created:       Sat Mar 11 12:43:07 2023
Last mount time:          Tue Mar 28 18:50:46 2023
Last write time:          Tue Mar 28 18:50:46 2023
Mount count:              5
Maximum mount count:      -1
Last checked:             Sat Mar 11 12:43:07 2023
Check interval:           0 (<none>)
Reserved blocks uid:      0 (user root)
Reserved blocks gid:      0 (group root)
First inode:              11
Inode size:               128
Default directory hash:   half_md4
Directory Hash Seed:      4d11bca9-0b5e-4645-96cc-355af63698fd


Group 0: (Blocks 1-8192)
  Primary superblock at 1, Group descriptors at 2-2
  Reserved GDT blocks at 3-258
  Block bitmap at 259 (+258), Inode bitmap at 260 (+259)
  Inode table at 261-516 (+260)
  7662 free blocks, 2037 free inodes, 2 directories
  Free blocks: 531-8192
  Free inodes: 12-2048
Group 1: (Blocks 8193-16384)
  Backup superblock at 8193, Group descriptors at 8194-8194
  Reserved GDT blocks at 8195-8450
  Block bitmap at 8451 (+258), Inode bitmap at 8452 (+259)
  Inode table at 8453-8708 (+260)
  7676 free blocks, 2048 free inodes, 0 directories
  Free blocks: 8709-16384
  Free inodes: 2049-4096
Group 2: (Blocks 16385-24576)
  Block bitmap at 16385 (+0), Inode bitmap at 16386 (+1)
  Inode table at 16387-16642 (+2)
  7934 free blocks, 2048 free inodes, 0 directories
  Free blocks: 16643-24576
  Free inodes: 4097-6144
Group 3: (Blocks 24577-32768)
  Backup superblock at 24577, Group descriptors at 24578-24578
  Reserved GDT blocks at 24579-24834
  Block bitmap at 24835 (+258), Inode bitmap at 24836 (+259)
  Inode table at 24837-25092 (+260)
  7676 free blocks, 2048 free inodes, 0 directories
  Free blocks: 25093-32768
  Free inodes: 6145-8192
Group 4: (Blocks 32769-40960)
  Block bitmap at 32769 (+0), Inode bitmap at 32770 (+1)
  Inode table at 32771-33026 (+2)
  7933 free blocks, 2047 free inodes, 1 directories
  Free blocks: 33028-40960
  Free inodes: 8194-10240
Group 5: (Blocks 40961-49152)
  Backup superblock at 40961, Group descriptors at 40962-40962
  Reserved GDT blocks at 40963-41218
  Block bitmap at 41219 (+258), Inode bitmap at 41220 (+259)
  Inode table at 41221-41476 (+260)
  7676 free blocks, 2048 free inodes, 0 directories
  Free blocks: 41477-49152
  Free inodes: 10241-12288
Group 6: (Blocks 49153-57344)
  Block bitmap at 49153 (+0), Inode bitmap at 49154 (+1)
  Inode table at 49155-49410 (+2)
  7933 free blocks, 2047 free inodes, 1 directories
  Free blocks: 49412-57344
  Free inodes: 12290-14336
Group 7: (Blocks 57345-65536)
  Backup superblock at 57345, Group descriptors at 57346-57346
  Reserved GDT blocks at 57347-57602
  Block bitmap at 57603 (+258), Inode bitmap at 57604 (+259)
  Inode table at 57605-57860 (+260)
  7676 free blocks, 2048 free inodes, 0 directories
  Free blocks: 57861-65536
  Free inodes: 14337-16384
Group 8: (Blocks 65537-73728)
  Block bitmap at 65537 (+0), Inode bitmap at 65538 (+1)
  Inode table at 65539-65794 (+2)
  7934 free blocks, 2048 free inodes, 0 directories
  Free blocks: 65795-73728
  Free inodes: 16385-18432
Group 9: (Blocks 73729-81920)
  Backup superblock at 73729, Group descriptors at 73730-73730
  Reserved GDT blocks at 73731-73986
  Block bitmap at 73987 (+258), Inode bitmap at 73988 (+259)
  Inode table at 73989-74244 (+260)
  7676 free blocks, 2048 free inodes, 0 directories
  Free blocks: 74245-81920
  Free inodes: 18433-20480
Group 10: (Blocks 81921-90112)
  Block bitmap at 81921 (+0), Inode bitmap at 81922 (+1)
  Inode table at 81923-82178 (+2)
  7934 free blocks, 2048 free inodes, 0 directories
  Free blocks: 82179-90112
  Free inodes: 20481-22528
Group 11: (Blocks 90113-98304)
  Block bitmap at 90113 (+0), Inode bitmap at 90114 (+1)
  Inode table at 90115-90370 (+2)
  7934 free blocks, 2048 free inodes, 0 directories
  Free blocks: 90371-98304
  Free inodes: 22529-24576
Group 12: (Blocks 98305-106496)
  Block bitmap at 98305 (+0), Inode bitmap at 98306 (+1)
  Inode table at 98307-98562 (+2)
  7933 free blocks, 2047 free inodes, 1 directories
  Free blocks: 98564-106496
  Free inodes: 24578-26624
Group 13: (Blocks 106497-114688)
  Block bitmap at 106497 (+0), Inode bitmap at 106498 (+1)
  Inode table at 106499-106754 (+2)
  7934 free blocks, 2048 free inodes, 0 directories
  Free blocks: 106755-114688
  Free inodes: 26625-28672
Group 14: (Blocks 114689-122880)
  Block bitmap at 114689 (+0), Inode bitmap at 114690 (+1)
  Inode table at 114691-114946 (+2)
  7933 free blocks, 2047 free inodes, 1 directories
  Free blocks: 114948-122880
  Free inodes: 28674-30720
Group 15: (Blocks 122881-131071)
  Block bitmap at 122881 (+0), Inode bitmap at 122882 (+1)
  Inode table at 122883-123138 (+2)
  7933 free blocks, 2048 free inodes, 0 directories
  Free blocks: 123139-131071
  Free inodes: 30721-32768
```







