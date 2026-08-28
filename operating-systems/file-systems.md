# 文件系统



太久没有写文章了，有时候是因为懒，有时候是觉得理解的不够透彻，更多的是因为感觉文笔太差。。

今天就来聊聊文件系统，这也是个庞然大物，先说说现在流行的分布式存储把，现在分布式存储主要分为对象存储，块存储，和文件存储，这些存储其实单机本来就有的，不过是云计算厂商在云上提供了这些服务，把单机的存储转到了云上，准确的说其实应该叫云上的分布式对象存储、块存储、文件存储。

块存储要从Linux系统设备说起，这是Unix系统遗留下来的传统，也是一种优秀的设计思想，把所有的东西都看成文件，设备当然也是一种文件咯，根据读写方式的不同分为：块设备（按照Block为单位读写），字符设备（可以按照offset以字节为单位读写），以及其他设备。在虚拟文件系统里，下一层是就是块设备层，块设备层会调用驱动程序对硬盘进行读写，而对VFS提供的这个块设备的服务抽出来，拿到云上去做就叫块存储。

对象存储其实就是KV存储，因为在比如Java这种语言里，value存储的往往是一个对象的序列化后的数据，所以叫对象存储（理解的不对请告诉我！）。

文件存储指的是提供符合POSIX标准的存储服务，在Linux用户态下，想要对文件进行读写操作，必须通过系统提供的接口，比如read,write,open等，而POSIX标准就是一个针对操作系统提供的接口的标准，也就说说能够提供read,write,open等接口的存储服务就是文件存储。

#### 文件系统

其实文件系统茫茫多，ext,ext2,ext3,ext4等等等，看看linux/fs下面的模块就知道了，为了统一这些模块，linux抽象出了虚拟文件系统VFS，VFS把文件抽象成了inode，每个inode都会关联一组函数指针，这一组函数指针就是不用的文件系统具体的读写操作。

Linux启动时会挂载一个根目录文件系统，这个由你启动参数指定，启动过程中还有一些特殊的文件系统，比如sockfs,tmpfs,procfs等等，这些文件系统都会被挂载到一个挂载树上，挂载树的根就是`/`，他是这个系统这次启动的根目录，你可以在这棵树上继续挂载新的节点，每个节点都会针对一个具体的文件系统。

当你做一个操作时，比如open("/foo/bar")，VFS首先会在挂载树中查找挂载点，然后从挂载点找到对应的inode，从这个inode开始对剩余的路径继续进行相应的操作。

#### EXT2文件系统

EXT2和EXT3是兼容的是个简单的文件系统，很适合来学习，下面我们来剖析一下EXT2文件系统。

上面说到了，块设备就是按照块来读写的设备，那么EXT2就是决定如何将数据存储到对应的块设备上去的系统。

文件系统其实很简单，只有文件和目录，目录里可能会包含目录和文件，是一个树状的结构，而文件可能会存储数据，数据从几KB到几GB不等，所以要选择合适的文件系统，就首先要确定你的存储场景。而这么多的文件系统其实就是针对不同的场景经过权衡取舍的结果。

EXT2文件系统将许多Blocks分成多个Block Group，每个Block Group都有一个Group Descriptor，这些Group Descriptor都在一起，位于SuperBlock的后面，而每个Group Descriptor都含有block bitmap，inode bitmap，inode table等。

**SuperBlock**

对于EXT2文件系统来说，首先就是SuperBlock，SuperBlock是一个特殊的块，它记载着本文件系统全局的属性信息，位于磁盘起始的1024字节处，假如BlockSize是1024字节，它就位于第二个Block，假如BlockSize是4096字节，那它位于第一个Block，BlockSize在创建具体的文件系统时指定。

```
/*
 * Structure of the super block
 */
struct ext2_super_block {
	unsigned long  s_inodes_count;	/* Inodes count */
	unsigned long  s_blocks_count;	/* Blocks count */
	unsigned long  s_r_blocks_count;/* Reserved blocks count */
	unsigned long  s_free_blocks_count;/* Free blocks count */
	unsigned long  s_free_inodes_count;/* Free inodes count */
	unsigned long  s_first_data_block;/* First Data Block */
	unsigned long  s_log_block_size;/* Block size */
	long           s_log_frag_size;	/* Fragment size */
	unsigned long  s_blocks_per_group;/* # Blocks per group */
	unsigned long  s_frags_per_group;/* # Fragments per group */
	unsigned long  s_inodes_per_group;/* # Inodes per group */
	unsigned long  s_mtime;		/* Mount time */
	unsigned long  s_wtime;		/* Write time */
	unsigned short s_mnt_count;	/* Mount count */
	short          s_max_mnt_count;	/* Maximal mount count */
	unsigned short s_magic;		/* Magic signature */
	unsigned short s_state;		/* File system state */
	unsigned short s_errors;	/* Behaviour when detecting errors */
	unsigned short s_pad;
	unsigned long  s_lastcheck;	/* time of last check */
	unsigned long  s_checkinterval;	/* max. time between checks */
	unsigned long  s_reserved[238];	/* Padding to the end of the block */
};
```

比较关键的有s\_blocks\_per\_group和s\_log\_block\_size，这就确定了磁盘分区上有多少个block group。

**BlockDescriptor**

Block Descriptor紧跟SuperBlock，Block Descripotr的数量可以通过公式 `(s_blocks_count - s_first_data_block - 1) / s_blocks_per_group + 1`算出，而且每个BlockDescriptor的大小必须不能超过BlockSize，所以我们可能一下将所有的Block Descriptor读出来。

```
struct ext2_group_desc
{
	unsigned long  bg_block_bitmap;		/* Blocks bitmap block */
	unsigned long  bg_inode_bitmap;		/* Inodes bitmap block */
	unsigned long  bg_inode_table;		/* Inodes table block */
	unsigned short bg_free_blocks_count;	/* Free blocks count */
	unsigned short bg_free_inodes_count;	/* Free inodes count */
	unsigned short bg_used_dirs_count;	/* Directories count */
	unsigned short bg_pad;
	unsigned long  bg_reserved[3];
};
```

**Inode**

虚拟文件系统我们就提到Inode了，这里的Inode其实像是虚拟文件系统的Inode序列化后存放到磁盘中的Inode，每一个目录或者文件都会对应一个Inode。

根目录的Inode是固定的，从根目录开始对于特定Inode的读取，首先要知道Inode的inode号，然后用inode号除以SuperBlock中的s\_inodes\_per\_group得到在哪个Block Descriptor，通过余数得到在这个BlockDescriptor里的哪个inode，从inode table中读取对应的inode。

```
/*
 * Structure of an inode on the disk
 */
struct ext2_inode {
	unsigned short i_mode;		/* File mode */
	unsigned short i_uid;		/* Owner Uid */
	unsigned long  i_size;		/* Size in bytes */
	unsigned long  i_atime;		/* Access time */
	unsigned long  i_ctime;		/* Creation time */
	unsigned long  i_mtime;		/* Modification time */
	unsigned long  i_dtime;		/* Deletion Time */
	unsigned short i_gid;		/* Group Id */
	unsigned short i_links_count;	/* Links count */
	unsigned long  i_blocks;	/* Blocks count */
	unsigned long  i_flags;		/* File flags */
	unsigned long  i_reserved1;
	unsigned long  i_block[EXT2_N_BLOCKS];/* Pointers to blocks */
	unsigned long  i_version;	/* File version (for NFS) */
	unsigned long  i_file_acl;	/* File ACL */
	unsigned long  i_dir_acl;	/* Directory ACL */
	unsigned long  i_faddr;		/* Fragment address */
	unsigned char  i_frag;		/* Fragment number */
	unsigned char  i_fsize;		/* Fragment size */
	unsigned short i_pad1;
	unsigned long  i_reserved2[2];
};
```

对文件的读写其实就是对于inode的读写，inode是如何存放连续的数据的呢，答案就在i\_block里。

首先每个inode都有15个block指针，前12个指针是direct pointer，直接指向一个具体的block。

假设这些block用完了之后，第13个指针是indirect pointer，它指向的是block的指针。

第十四个指针是double indirect pointer，它指向的是block指针的指针。

第十五个指针是triple indirect pointer，它指向的是block指针的指针的指针。

通过这样多级指针的设计我们可以把某个文件扩展的很大，而在文件较小时也不影响它的性能。

**direntry**

direntry是一种固定大小的存储格式，它代表一个具体的目录，它的inode内容就是它所包含的子目录或者文件。

```
struct ext2_dir_entry {
	unsigned long  inode;			/* Inode number */
	unsigned short rec_len;			/* Directory entry length */
	unsigned short name_len;		/* Name length */
	char           name[EXT2_NAME_LEN];	/* File name */
};
```
