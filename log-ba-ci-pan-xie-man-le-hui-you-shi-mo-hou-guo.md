# Log把磁盘写满了会有什么后果



#### 背景

最近在测试环境灰度时，业务反应服务无响应，然后马上摘除节点恢复业务。

到测试机器检查，发现进程存在，日志正常。

`printf "stats\r\n"|nc localhost 10101`

测试端口发现命令卡住，大约10s才回复。

#### syscall:write耗时

df -h发现磁盘写满，然而磁盘满会使写磁盘这么慢吗？

用strace查看每次write耗时

![img](http://ww1.sinaimg.cn/large/7cb11947ly1fgt4c2zmn9j213o0psgx3.jpg)

每次write耗时250ms+...有点长。

修改代码，把\_log函数体注释，重新编译发现服务恢复正常。

在印象中，感觉磁盘满不可能会导致写文件这么慢啊，有啥问题呢？

又找了一台新的机器，写了小程序先把磁盘写满，然后启动同样的程序，测试发现一切正常

![img](http://ww1.sinaimg.cn/large/7cb11947ly1fgt4fv7poyj20jm0aotcw.jpg)

这就很有趣了，那么问题在哪呢？

#### ext4

uname -r发现没有问题的内核版本是3.10.0-229.el7.x86\_64，有问题的版本是2.6.32-431.11.2.el6.toa.2.x86\_64

df -T发现文件系统都是ext4，怀疑可能是ext4文件系统某些机制导致的

使用perf stat查看所有ext4文件系统的trace event

`sudo perf stat -e "ext4:*" -p 23488 sleep 10`

1.  有问题的机器

    ![img](http://ww1.sinaimg.cn/large/7cb11947ly1fgt4gd89grj20cl0gxjsw.jpg)
2.  没问题的机器

    ![img](http://ww1.sinaimg.cn/large/7cb11947ly1fgt4gm73kxj20bw0jsjtb.jpg)

发现ext4的执行路径差异很大，然后同时trace ext4和syscall:write

`sudo perf stat -e "syscalls:sys_enter_write,ext4:*" -p 23488 sleep 10`

![img](http://ww1.sinaimg.cn/large/7cb11947ly1fgt4gtu8bqj20bk0h6gn6.jpg)

试了几次发现syscall:write和ext4:ext4\_da\_write\_begin次数完全一样，这说明每次write都会调用ext4的ext4\_da\_write\_begin，但是接下来的执行逻辑就不一样了。

#### 内核源码

翻内核的源码发现，函数ext4\_da\_write\_begin在3.10和2.6差别很多，用systemtap脚本调试N久，尝试复原ext4\_da\_write\_begin不同的路径

```c
static int ext4_da_write_begin(){
    /*省略*/
	if (ret == -ENOSPC && ext4_should_retry_alloc(inode->i_sb, &retries))
		goto retry;
out:
	return ret;
}
```

看到在写满ENOSPC的错误发生时，ext4\_should\_retry\_alloc函数的嫌疑很大，怀疑是触发了不同的逻辑导致低版本内核频繁的重试

用脚本trace发现在函数ext4\_should\_retry\_alloc里有差异，有问题的机器会去频繁jbd2\_journal\_force\_commit\_nested，而没问题的机器直接return 0

```c
int ext4_should_retry_alloc(struct super_block *sb, int *retries)
{
	if (!ext4_has_free_blocks(EXT4_SB(sb), 1) ||
	    (*retries)++ > 3 ||
	    !EXT4_SB(sb)->s_journal)
		return 0;

	jbd_debug(1, "%s: retrying operation after ENOSPC\n", sb->s_id);

	return jbd2_journal_force_commit_nested(EXT4_SB(sb)->s_journal);
}
```

stap脚本

```
global joural
global ext4

probe module("ext4").function("ext4_da_write_begin").call {
    ext4++;
}

probe module("jbd2").function("jbd2_journal_force_commit_nested").call {
    joural++;
}

probe module("ext4").function("ext4_has_free_clusters").return {
   printf("%x\n",@cast($sb->s_fs_info,"ext4_sb_info")->s_journal);
}

probe timer.s(1), end {
    ansi_clear_screen();
    printf("%10s %10s\n","ext4_write_begin","jbd2_journal_force_commit_nested");
    printf("%10d %10d\n",ext4,joural);
}
```

1.  有问题的机器

    ![img](http://ww1.sinaimg.cn/large/7cb11947ly1fgt4hu91xjj209y00yweg.jpg)
2.  没问题的机器

    ![img](http://ww1.sinaimg.cn/large/7cb11947ly1fgt4i3pmzzj20a200w748.jpg)

继续往下看

```c
int ext4_has_free_blocks(struct ext4_sb_info *sbi, s64 nblocks)
{
    /* 省略 */
	/* Hm, nope.  Are (enough) root reserved blocks available? */
	if (sbi->s_resuid == current_fsuid() ||
	    ((sbi->s_resgid != 0) && in_group_p(sbi->s_resgid)) ||
	    capable(CAP_SYS_RESOURCE)) {
		if (free_blocks >= (nblocks + dirty_blocks))
			return 1;
	}

	return 0;
}
```

发现在函数ext4\_has\_free\_blocks中会判断是否会有空闲块，决定是不是重试，对比3.10的代码，在3.10中的函数是

```c
static int ext4_has_free_clusters(struct ext4_sb_info *sbi,
				  s64 nclusters, unsigned int flags)
{
    /* 省略 */
	/* No free blocks. Let's see if we can dip into reserved pool */
	if (flags & EXT4_MB_USE_RESERVED) {
		if (free_clusters >= (nclusters + dirty_clusters))
			return 1;
	}

	return 0;
}
```

发现在判断逻辑上的不同，在3.10里，函数ext4\_has\_free\_clusters调用时，flags传入0，所以不考虑root用户的预留空间，而在2.6里会判断用户是否是root用户，假如是root用户，那么在判断空闲块是否够用时会加上root用户的预留空间。

写个脚本看一下free\_blocks，nblocks，dirty\_blocks,root\_blocks分别是多少

```
probe module("ext4").function("ext4_has_free_blocks") {
  printf("free: %d dirty: %x nblocks: %d root: %d\n",$sbi->s_freeblocks_counter->count,$sbi->s_dirtyblocks_counter->count,$nblocks,$sbi->s_es->s_r_blocks_count_hi<<32|$sbi->s_es->s_free_blocks_count_lo);
}
```

![img](http://ww1.sinaimg.cn/large/7cb11947ly1fgtxoumbg3j20bv05fjsm.jpg)

root用户的判断逻辑是free > nblocks + dirty，是true，所以一直在重试

非root用户的判断逻辑是free > nblocks + dirty + root，是false，所以不会重试

#### 验证

这应该是问题所在了，那么也很好验证，在低内核版本上只有root用户才有权限使用预留空间，那么我们用其他用户启动程序，应该就没有这个问题了。

![img](http://ww1.sinaimg.cn/large/7cb11947ly1fgt4h0iadkj20jp092q5y.jpg)

重新用非root用户启动程序，strace查看，完全正常。

#### 总结

那么这个问题就是由于低版本的内核文件系统某些缺陷导致的，但是线上的内核版本不一，假如由于某些问题或者其他进程把磁盘刷爆，那么很有可能导致服务不可用。

预防措施：

1. 对日志增加监控，防止出现日志把磁盘刷爆的情况
2. 尽量用非root用户启动进程
