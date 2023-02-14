# virtio: Towards a De-Facto Standard For Virtual I/O Devices

#### 摘要

Linux内核现在支持了最少8种虚拟化系统：Xen，KVM，VMware's VMI，IBM的System p，IBM System z，用户态Linux，lguest和IBM iSeries。看起来会出现更多类似的系统，每个系统都有它独立的块设备，网络，终端以及不同功能和优化的驱动。

一个尝试是virtio：一系列高效、维护很好的Linux驱动，用一层很薄的抽象就可以被多种hypervisor所使用。对于每个驱动它包含了一个简单的可扩展的机制。我们还提供了一个ring buffer传输的实现，和KVM，lguest用的ring buffer一样。对于新的hypervisor来说这种方法副作用很小，只要支持了这种高效的传输机制就能立马减少大量的工作量。最后，我们提供了一个把vring和设备配置成PCI设备的实现：这意味着guest os不需要增加一个新的PCI驱动，hypervisor只需要在他们实现的虚拟设备中支持vring就可以了（现在只有KVM这么做了）。

本文将会说明Linux中vritio的API层的实现，vring的实现，最后在全虚拟化guest作为PCI设备的体验。我们将会包含一些其他工作把这种I/O机制深入集成到Linux host kernel中。

### 1. 介绍

Linux kernel被移植到了很多平台；官方的kernel树包含了24种架构相关的目录和大约200万行架构相关的代码。这些代码大部分都是在支持不同的平台变体。不幸的是，我们发现只有一种架构从内核中删掉了，但同时新增加的硬件变种像雨后的春笋一样多。每天大约有1万行的代码改动，这是你能想象的事。

如果我们把Linux看做一个虚拟化下的guest，我们会特别幸运：支持IBM的 System p，System z和遗产iSeries。用户态的Linux早就被包含在内了，就是在Power,IA64 和 32位,64位的x86机器上把Linux运行为一个用户态的进程。过去两年，x86被证明非常有竞争力，有了来自于XenSource的Xen的支持，VMware VMI的支持以及Qumranet KVM的支持。最后，我要提一下我自己的一个贡献，lguest：一个玩具hypervisor，可以用来开发或者教学，也悄悄的溜到主干代码里了。

8个平台的每一个都想要他们自己的块，网络和终端驱动，有时还有framebuffer，USB控制器，host文件系统，虚拟的kitchen sink控制器等。他们很少会去优化他们的驱动，又提供了基本重叠但有微小区别的功能。重要的是，没有人乐意去管他们的驱动，或者维护驱动。

这个问题在KVM项目里越来越明显，当它在2006年突然跳入Linux的视野中时，仍然没有一个paravirtual设备模型。模拟设备的性能限制编的越来越明确，不管是移植Xen的驱动模型还是开发另一个驱动模型都不是很容易实现。在Xen设备模型的工作过程中，我们相信为各种类型的hypervisor和平台创建一个通用的虚拟I/O机制会很有效率，也是对于把Xen的设备配置系统引入的一种赎罪。

### 2. VIRTIO: THE THREE GOALS

我们最初始的目标非常明确：所有的工作都在Linux kernel内部，不需要其他人在做一些事情。如果嵌入式虚拟I/O机制的开发者对Linux熟悉，它可能会引导他们映射Linux的API到他们自己的ABI。但是『如果』和『可能』是不充分的：我们对这些情况会更有耐心。

经验显示高级的传输机制更倾向于不止针对一个hypervisor和架构，而是各个种类的设备。所以接下来一个明确的目标就是提供一个公共的ABI和buffer的使用。更加小心的是，我们的virtio\_ring实现并没有什么技术革新：开发者看下代码就会发现没有什么不喜欢的。

最后，我们提供了两种ABI的实现，对虚拟I/O设备来说用virtio\_ring设施和Linux API。这些实现了虚拟I/O的最后一部分：设备探测和配置。重要的是，他们证明了用Linux 虚拟I/O API来提供功能以及向前向后兼容是多么的容易，所以未来的Linux 驱动功能都可以被探测和使用对任何host实现来说。

明确的驱动分离，传输和配置代表了我们从当前实现的一些思考。比如，你不能把Xen的Linux网络驱动用在一个新的hypervisor上除非你支持了Xen-Bus的探测和配置系统。

### 3. VIRTIO: A LINUX-INTERNAL ABSTRACTION API

如果我们想在虚拟驱动设备上减少重复的工作，我们需要一个直接的抽象层，这样驱动就可以共享代码了。一种方法是提供一套给虚拟驱动使用的公共函数，另一种更耗费精力的方法是用一个公共的驱动和一套操作结构：一系列函数指针用于实际的传输实现的接口。任务是为了创建一个所有虚拟设备的传输层抽象，能很好去优化传输，也允许已一个较小的代价从已有的传输过渡过来。

当前的结果就是virtio驱动把他们注册到了一个32位的设备类型，可以选择性的拒绝一个32位的vendor。合适的virtio设备被发现时会调用驱动的probe函数：传入的结构体virtio\_device有一个virtio\_config\_ops的指针，可以被用来解析设备配置。

配置操作可以被分为4个部分：读和写功能bits，读和写配置的空间，读和写状态bits，设备重置。设备查找device-type-specific feature bits符合他想要的功能，比如VIRTIO\_NET\_F\_CSUM feature bit表明是否支持网络设备checksum的offload。Feature bits是非常明确的：host知道guest返回的feature bits，然后可以决定driver需要理解哪些feature。

第二部分是配置空间：这是一个和虚拟设备规格信息相关联的结构。他们可以被guest读或者写。比如，网络设备有一个VIRTIO\_NET\_F\_MAC的feature bit，表明host想要设备有一个独立的MAC地址，然后配置空间包含了这个值。

这些机制在未来给了我们很大空间，也方便hosts给设备增加功能，只要在feature bits数目和配置空间的layout上达成一致。

也有一些操作去读取或者设置8位的设备状态字段，用于guest表明设备的状态；当VIRTIO\_CONFIG\_S\_DRIVER\_OK设置了的时候，就知道guest驱动已经完成了feature探测。此时host也知道了他有什么功能和想用什么功能。

最后，重置操作期望就是重置设备，它的配置和状态位。对于可插拔的模块化驱动来说是必备的。它也避免了在驱动停止的时候移除Buffer的问题：重置后，buffer只有明确知道设备不会修改了以后才能被释放。它也可以用于尝试guest上设备的恢复。

#### 3.1 Virtqueues: A Transport Abstraction

我们的配置API非常重要，但是性能相关的部分API就是I/O机制。我们最这块的抽象是一个virtqueue：对应的配置函数有一个find\_vq，它会返回queue的一个移植结构，加上一个virtio设备和index数字。有些设备只有一个queue，比如virtio block设备，但是其他设备像网络和中断设备有一个queue输入一个queue输出。

一个virtqueue是一个简单的队列，里面是guest生产的buffer，消费的是host。每个buffer都是一个分散聚集的数组，包含了可读和可写的部分：数据的结构依赖于设备的类型。virtqueue的操作接口如下：

![](../.gitbook/assets/image.png)

add\_buf是用来添加一个新的buffer到队列中；data参数是由驱动提供的非空token，当buffer被消费时返回。kick通知对端（比如host）在buffer增加以后；一个kick之前可以添加多个buffer。这对通知来说很重要，因为通常都会引起guest昂贵的exit。

get\_buf得到一个用过的buffer：返回对端写到buffer的长度（我们可以看到为什么在讨论guest间的通信）。它返回了cookie到add\_buf或者NULL：buffer不是顺序被用的。

disable\_cb是一个hint，当guest不想知道一个buffer被用了：这和禁止设备的中断是等效的。driver初始化的时候在virtqueue上注册了一个回调，virtqueue的回调可能会禁止更多的一些回调在唤醒service线程前。在这之后无法保证回调仍然会被调用，因为这会需要昂贵的SMP系统上的同步开销。实际上，这是一个减少和host或者VMM不必要开销的优化。

enable\_cb是和disable\_cb相反的。driver会经常重新启用回调，只要它处理完了队列里所有的待处理buffer。在一些virtio传输上仍然有竞争：buffer可能会在get\_buf返回空和enable\_cb之间用，也不会有回调被调用。水平触发的中断实现就不会有这个问题，但enable\_cb也会返回false去说明窗口出现了更多的工作，在回调被禁止的地方。

所有的这些调用对于Linux上下文来说都是可用的，是否要确保调用同步取决于调用者。唯一的异常是disable\_cb-它经常被自己调用，也用来去禁止回调，但是它是不可信的，所以它随时可以发生。

### 4. VIRTIO\_RING：A TRANSPORT IMPLEMENTATION FOR VIRTIO

尽管我们相信任何优化传输会共享类似的特性，Linux virtio/virtqueue API 更偏向于我们特殊的传输机制实现，被称为virtio\_ring。

先于virtio创建之前，lguest已经有一套虚拟I/O系统：一个通用的，多路I/O机制，一个可以用到guest之间的网络，基于Xen的一个前期版本。但是由于大部分系统的复杂度来自于他自己的N路广播，让它看起来像只是针对特殊场景的一个实现。跟这个功能一样有趣，牺牲它使我们有了一个简单的scheme包括ringbuffers，一个标准的高速I/O方法。经过几次简单的实现的迭代，我们有了一个KVM和lguest共同使用的virtio\_ring scheme。

virtio\_ring包含了3个部分：描述符数组-guest保存了length和地址对，可用的ring-guest用来表明哪些描述符可以被使用，和使用的ring-host用来表明它用了哪些描述符。ring的大小可变，但必须是2的整数幂。

![](<../.gitbook/assets/image (2).png>)

```
struct vring_desc
{
    __u64 addr;
    __u32 len;
    __u16 flags;
    __u16 next;
};
```

每个描述符包含了buffer guest的物理地址，它的长度，可选的next可以用来链接，和两个flag：一个用来表明next字段是不是有效的，另一个控制buffer是否是只读的还是只写的。这可以允许一个list的buffer包含可读和可写的副本，对于实现一个块设备来说非常有用。按照习俗，可读的buffer优先于可写的buffer。

64位的地址使用尽管在32位的系统上也是个tradeoff：他允许一个通用的格式在老的平台上只用32位。所有的结构被设计成避免对齐，但是我们没有定义一个特定的字节序格式：guest定义了它天然的字节序。

```
struct vring_avail
{
    __u16 flags;
    __u16 idx;
    __u16 ring[NUM];
};
```

可用的vring包含了一个运行的index，一个中断压缩flag，一组描述符表的表头。描述符到ring的分离是因为virtqueue天然异步的特性：快速的描述符可能已经循环了很多次，而慢的描述符仍然在等待完成。这对于实现一个块设备来说非常有用，而且对于zero-copy的网络也很有用。

```
struct vring_used_elem
{
    __u32 id;
    __u32 len;
};

struct vring_used
{
    __u16 flags;
    __u16 idx;
    struct vring_used_elem ring[];
};
```

used ring和available ring差不多，但是被host像描述符列表消费。注意到把这种结构放到一个单独的page里的时候会有对齐：这带来了足够好的cache行为和承认了每端只需要写virtqueue结构的一部分即可。

注意到vring\_used flags和vring\_avail flags：这些现在用于压缩通知。比如，used flags字段被host用来告诉guest，不要kick当它在添加buffer时：因为kick需要一个vmexit，这是一个重要的优化，KVM的实现用它和一个timer来做网络层传输退出的迁移。相似的，avail flags字段被guest网络驱动用来建议一些更多的中断不需要（比如disable\_cb和enable\_cb 设置和取消这个bit）。

最后，我们不去关注guest的停止和启动的设计是有意义的；我们不需要他们，因为我们发布我们自己的buffer。实际上，host里KVM的停止和启动的实现已经证明了相当不重要了。

#### 4.1 A Note on Zero-Copy And Religion of Page Flipping

在设计效率I/O的时候，我们考虑到了两件事：每个操作需要的通知的数量，访问到的cache冷数据的规模。前者被virtio\_ring中断压缩flags处理的很好了。cache冷数据的处理更值得讨论。

在KVM和lguest模型里，guest内存表现为host进程虚拟地址空间的一部分：对于host OS来说，进程就是guest。然后I/O从guest到host应该像I/O从任何正常host进程一样快，即使加上guest和host间额外的开销。这也是为什么virtio专注于发布buffer，假设I/O target可以访问内存。

Xen没有如此天然的访问模型：没有host可以访问到其他guest的内存，但是所有的域名都有peer。这和KVM以及lguest里guest间的通信是一样的，映射一个guest的buffer到另一个对于允许guest间的zero-copy是必须的。

数据复制的时间和冷cache的数据量成正比：不管是guest还是host接触到了多大的数据量，复制的时间都会被偿还。page映射的开销独立于数据的大小，但它只工作在页对齐的数据上。因为这个原因，他只是对于大块数据有吸引力。

在通用的翻页scheme上，每个guest内部的I/O包含了两个分离的页表修改：一个是map，一个是unmap。我们必须确保buffer在释放前被unmap了，否则有可能page被回收的时候还有另外一个guest仍然去访问它。代码可以用batching和完成的通知来分摊，但在SMP系统上这种操作仍然很昂贵。

永久的共享一个定长的内存区域可以避免这种翻页的需求，但是不适用于一个通用的guest OS，比如Linux：如果我们复制定长的内存区域这样另外一段可以访问数据，我们可以简单的在guests间复制就可以了。

大块的guest间的复制相对来说比较少：virtio\_net被限制到了64k packets，因为TSO的实现，guest内部的块设备看起来是一个复杂的使用场景。不管怎么样，证明page-flipping的价值是一段简单的代码就能做到的，同时我们猜测结果会比较无关紧要，我们也希望会有人来证明我们是错的。

一个这些工作不会做的原因是DMA引擎的使用，可以用来复制大量的数据。它们优化于类似的场景，page-flipping可能会提供一些便利：大块的冷cache传输。

### 5. CURRENT VIRTIO DRIVERS

现在我们熟悉了Linux virtio和virtqueue的概念以及API，也看到了传输的实现，这些存在的virtio驱动看起来是很有意义的。我们有一个lguest简单的和非常哑的终端驱动，同时KVM用了一些终端的模拟；终端的性能和接收注意不太一样，直到有人释放了一些东西，比如一个virtcon的benchmark。

我们也有一个简单的balloon驱动，可以让host指定它想从guest解压的page的数量。guest传送了多组page数量到host；host允许去unmap这些页面以及用空页来替换他们当他们被访问到时。

对于块和网络驱动，我们仍然会深入更多细节。

#### 5.1 Virtio Block Driver

对于块设备，我们有一个简单的请求队列。队列里每个buffer的前16字节是一个只读的描述符：

```
struct virtio_blk_outhdr
{
    __u32 type;
    __u32 ioprio;
    __u64 sector;
};
```

