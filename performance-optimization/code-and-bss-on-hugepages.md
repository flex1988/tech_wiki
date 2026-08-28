# 把程序的代码段和BSS段放到大页内存

在 Linux服务器里，一般内存的页大小是4KB，进程的页表一般有5级，从虚拟内存地址到实际物理地址的转换是需要经过页表来解析的。

但对于高性能的计算机CPU系统架构来讲，这肯定是跟不上 CPU 的速度的，所以在 CPU 里有一个叫 TLB 的硬件存了虚拟地址到实际物理地址的映射，当 TLB 命中时就不需要走进程的页表来寻找物理地址了，但 TLB 的大小是有限的，当 TLB miss 了后，还是需要查找页表。

所以如果把系统默认的4KB 内存页大小改成2MB，那么 TLB 需要保存的映射项就会降低512倍，能够优化性能。

1 HugeTLB pages

{% embed url="https://www.kernel.org/doc/html/v4.18/admin-guide/mm/hugetlbpage.html" %}
Linux HugeTLB pages
{% endembed %}

在 Linux 中支持多种大小的 HugePage，根据 CPU 架构的不同，比如x86支持4KB 2MB 1GB 的大页。

首先需要在系统中配置一些大页内存，如下：

![](<../.gitbook/assets/image (17).png>)

```
 echo 2048 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```

2 libhugetlbfs

Linux 里 libhugetlbfs 可以在不用改源码的前提下，把程序的代码段数据段放到大页内存中，从而提高性能。

{% embed url="https://learn.arm.com/learning-paths/servers-and-cloud-computing/libhugetlbfs/libhugetlbfs_general/" %}

编译程序带上编译选项：

```
-B /usr/share/libhugetlbfs -Wl,--hugetlbfs-align -Wl,--no-as-needed
```

运行程序需要设置环境变量：

```
HUGETLB_ELFMAP=RW
```

```
cat /proc/89612/smaps|less 可以看到进程的代码段已经放到了大页中
```

<figure><img src="../.gitbook/assets/image (19).png" alt=""><figcaption></figcaption></figure>

#### 3 perf  cache miss

{% embed url="https://perf.wiki.kernel.org/index.php/Tutorial" %}

通过 perf 工具观察 cache miss 事件来评估优化效果：

```
perf stat -e dTLB-loads,dTLB-load-misses,iTLB-loads,iTLB-load-misses,L1-icache-loads,L1-icache-loads-misses ./application
```

<figure><img src="../.gitbook/assets/image (22).png" alt=""><figcaption><p>优化前</p></figcaption></figure>

<figure><img src="../.gitbook/assets/image (20).png" alt=""><figcaption><p>优化后</p></figcaption></figure>

可以看到优化后，iTLB-loads 减小显著。

iTLB-loads 意思是没命中L1-itlb，但是命中 unified tlb

iTLB-load-misses 意思是既没命中 L1-itlb，也没命中 unified tlb

但程序整体优化结果有限，原因是这个程序的 itlb 本来命中就不错所以，整体优化效果不明显，但对于二进制体积很大的程序来说或许会有更好的优化效果。
