# 内存分页

#### 内存管理-内存分页

一直觉得内存管理是操作系统最复杂的模块，没有之一。之前对于内存管理的了解仅限于上学时学过的堆，栈，内存映射，页表，内存分配器等概念层面，对于如何实现的有点模糊。去年想从0写个操作系统，结果写了个hello world就扔在那了。

前几天兴趣重燃，找了好多资料和文章终于拼凑出一个能启动的玩具内核了。话不多说先放个链接：[Github](https://github.com/flex1988/phenix)

**1.为何要内存分页**

内存管理其实是一个非常大的topic，这篇文章我只想写内存映射这一块，任何其他的东西都忽略掉。

内存管理对于C/C++程序员来说其实是个非常非常重要的一块东西，当然其他程序员了解一下也不错。

很久之前的机器是没有分页的，当时加载一个程序直接在内存上跑，跑完了换下一个，但是如果同时让两个程序跑的话，内存就有点捉襟见肘了。后来出现了分段管理，一个逻辑地址由segment+offset组成，可能是分段仍然不够灵活，后来又出现了分页管理。

在了解分页之前，我们先思考一下为什么要分页？

我们都知道每个进程都有一个自己的虚拟进程空间，如果每个进程的内存都是一对一映射话，物理内存是肯定不够的，而且很多时候虚拟进程空间的内存只有一小部分在被使用，所以内存分页可以有效的节约物理内存，同时虚拟内存空间也带来了程序代码地址无关的好处，隔离了不同进程，避免互相影响。

这边文章只讲32位操作系统的内存分页，因为64位的原理大体相同，只是多了一级页表。

**2.页表**

说到内存分页，最核心的部分是页表，页表分成两级，第一级页表是page directory，第二级page table，每级页表大小都是4K，都分为1024个entry，每个entry大小为4个字节。

page directiry每个entry是4个字节，其中0-9是标志位，9-11是预留位，12-31是逻辑地址的前十位。

每个标记位的含义：

* P `present` 代表映射的物理页在内存中，如果为false，代表映射的物理页被swap到了磁盘，会发生缺页中断，由中断处理函数将物理页加载回内存
* R `read/write` 代表本页能否被修改
* U `User/Supervisor` 代表页的访问权限是内核态还是用户态
* W `Write-Through`
* D `Cache Disable` 是否不允许cache
* A `Accessed` 该页是否被访问过
* S `Page Size` 表示该页为4MB还是4KB

![img](http://ww1.sinaimg.cn/large/7cb11947ly1fiva7ystjrj20c0073t96.jpg)

page table的entry跟page directory大致一样，只是有些标记位的含义不同。

![img](http://ww1.sinaimg.cn/large/7cb11947ly1fiva8egq7mj20c00740t3.jpg)

CR3寄存器中存着指向第一级页表的地址，所以分页整体结构如下图所示：

![img](http://ww1.sinaimg.cn/large/7cb11947ly1fiva7e1qecg20m70gn3zm.gif)

**3.地址转换**

MMU是Intel CPU中具体处理内存分页的单元，它是如何寻址的呢？

假设我们有一个虚拟内存地址p=0x12345678，4个字节的地址，换成2进制就是0001001000 1101000101 011001111000

1. 首先地址的前十位用来取出第一级页表的entry，也就是0001001000，以这个值为index，在第一级页表中找到entry1，找到entry1后根据entry1的标记位看是否有权限或者该entry是否已经map等
2. 取entry1的前20位地址（物理地址），并将后12位全部置0，找到指向的4K物理内存为第二级页表
3. 取内存地址p的中间10位地址为index，从第二级页表中取出entry2
4. 取entry2的前20位加上内存地址p的后12位组成的物理地址为真正的物理地址，内存地址转换完成

```c
uint32_t virt_to_phys(uint32_t virtualaddr) {
    int pdidx = virtualaddr >> 22;
    int ptidx = virtualaddr >> 12 & 0x03ff;
    int offset = virtualaddr & 0xfff;

    page_tabl_t* tabl = _kernel_pd->tabls[pdidx];
    page_t* page = &tabl->pages[ptidx];

    return page->addr << 12 + offset;
}
```

**4.Page Faults**

当进程访问到被swap出去的内存或者写了只读的内存或者用户态的程序试图写内核态的内存的时候都会发生页错误中断，同时CPU会将错误码PUSH到栈中，然后将由中断处理函数来具体处理剩下的逻辑。
