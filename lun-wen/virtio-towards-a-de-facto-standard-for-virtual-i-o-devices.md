# virtio: Towards a De-Facto Standard For Virtual I/O Devices

### 摘要

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

![](<../.gitbook/assets/image (6).png>)

add\_buf是用来添加一个新的buffer到队列中；data参数是由驱动提供的非空token，当buffer被消费时返回。kick通知对端（比如host）在buffer增加以后；一个kick之前可以添加多个buffer。这对通知来说很重要，因为通常都会引起guest昂贵的exit。

get\_buf得到一个用过的buffer：返回对端写到buffer的长度（我们可以看到为什么在讨论guest间的通信）。它返回了cookie到add\_buf或者NULL：buffer不是顺序被用的。

disable\_cb是一个hint，当guest不想知道一个buffer被用了：这和禁止设备的中断是等效的。driver初始化的时候在virtqueue上注册了一个回调，virtqueue的回调可能会禁止更多的一些回调在唤醒service线程前。在这之后无法保证回调仍然会被调用，因为这会需要昂贵的SMP系统上的同步开销。实际上，这是一个减少和host或者VMM不必要开销的优化。

enable\_cb是和disable\_cb相反的。driver会经常重新启用回调，只要它处理完了队列里所有的待处理buffer。在一些virtio传输上仍然有竞争：buffer可能会在get\_buf返回空和enable\_cb之间用，也不会有回调被调用。水平触发的中断实现就不会有这个问题，但enable\_cb也会返回false去说明窗口出现了更多的工作，在回调被禁止的地方。

所有的这些调用对于Linux上下文来说都是可用的，是否要确保调用同步取决于调用者。唯一的异常是disable\_cb-它经常被自己调用，也用来去禁止回调，但是它是不可信的，所以它随时可以发生。

### 4. VIRTIO\_RING：A TRANSPORT IMPLEMENTATION FOR VIRTIO

尽管我们相信任何优化传输会共享类似的特性，Linux virtio/virtqueue API 更偏向于我们特殊的传输机制实现，被称为virtio\_ring。

先于virtio创建之前，lguest已经有一套虚拟I/O系统：一个通用的，多路I/O机制，一个可以用到guest之间的网络，基于Xen的一个前期版本。但是由于大部分系统的复杂度来自于他自己的N路广播，让它看起来像只是针对特殊场景的一个实现。跟这个功能一样有趣，牺牲它使我们有了一个简单的scheme包括ringbuffers，一个标准的高速I/O方法。经过几次简单的实现的迭代，我们有了一个KVM和lguest共同使用的virtio\_ring scheme。

virtio\_ring包含了3个部分：描述符数组-guest保存了length和地址对，可用的ring-guest用来表明哪些描述符可以被使用，和使用的ring-host用来表明它用了哪些描述符。ring的大小可变，但必须是2的整数幂。

![](<../.gitbook/assets/image (2) (3) (1).png>)

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

这个类型表明不管它是一个读还是写还是通用的SCSI命令，或是一个写barrier应该先于这个命令。IO的优先级允许guest提示请求的相对优先级，现在的实现基本都会忽略这个，读写sector是512字节的offset。

差不多描述符的字节不是只读就是只写，依赖于请求的类型，总长度决定了请求的大小。最后一个字节是只写的，表明了请求是成功（0）还是失败（1），亦或是不支持（2）。

块设备可以支持barrier，简单的SCSI命令（大部分用于弹出虚拟的CDROM）。对于更精确的用途来说，可以实现一个virtio上的SCSI HBA。

#### 5.1.1 Virtio Block Mechanics

为了聚焦于这些机制，让我们过一下virtio block驱动做一个单个block读的概念路径，用virtio\_ring作为传输例子。开始时，guest有一个空的buffer，数据可以被读进去。我们分配了一个struct virtio\_blk\_outhdr和请求metadata，和一个单独的字节去接受状态（成功或失败）像表2那样。

![](<../.gitbook/assets/image (3).png>)

我们把请求的这三个部分放置到描述符表三个空闲的entry里，然后把他们链接起来。在这个例子里，我们读的buffer是物理连续的：如果不是，我们要用到多个描述符表的entry。header是只读的，空的buffer和状态字节是只写的，像表3这样。

![](<../.gitbook/assets/image (1) (5).png>)

![](<../.gitbook/assets/image (2) (3).png>)

一旦做了这些以后，描述符就可以被标记为available像图4一样。这通过把描述符的索引头放到available ring里来做到，发起一个memory barrier，然后增长索引。kick用来通知host一个请求正在排队（实际上，我们的driver会把所有的请求都放到ring后，在发起一个kick）。

在未来的一个时刻，请求像图5一样完成：buffer被填上了，状态字节也被更新成成功状态。这时，描述符头被从used ring返回，guest会得到通知。块驱动回调会重复的调用get\_buf来检查哪些请求完成了，知道get\_buf返回NULL。

![](<../.gitbook/assets/image (1).png>)

#### 5.2 Virtio Network Driver

网络设备用了两个队列：一个是发送，一个是接收。像块驱动一样，每个网络buffer以一个header起始，允许checksum offload和TCP/UDP段的offload。segmentation的offload被开发用于网络硬件，用于解决大的MTU和1500字节的packet错配的问题；少的packet意味着更少的PCI传输。在一个虚拟的环境里，它意味着更少的虚拟机调用和性能的提升。

```
struct virtio_net_hdr
{
// Use csum_start, csum_offset
#define VIRTIO_NET_HDR_F_NEEDS_CSUM 1
    __u8 flags;
#define VIRTIO_NET_HDR_GSO_NONE     0
#define VIRTIO_NET_HDR_GSO_TCPV4    1
#define VIRTIO_NET_HDR_GSO_UDP      3
#define VIRTIO_NET_HDR_GSO_TCPV6    4
#define VIRTIO_NET_HDR_GSO_ECN      0x80
    __u8 gso_type;
    __u16 hdr_len;
    __u16 gso_size;
    __u16 csum_start;
    __u16 csum_offset;
};
```

2.6.24的virtio网络驱动有一些基础设施TSO在packet上，但由于他不会申请大的接收buffer，所以也不能用。我们会看下怎么把这个变更用上。

一个有趣的点是网络驱动会把回调放到传输virtqueue上：跟块驱动不同，它不关注packet什么时候完成。异常点是队列满的时候：驱动会重新启用回调，当buffer可用的时候可以尽快传输。

### 6. VIRTIO\_PCI: A PCI IMPLEMENTATION OF VRING AND VIRTIO

目前为止我们已经两种可以统一虚拟I/O的方法。首先，用Linux内核中的virtio驱动然后提供适当的ops操作去支持特定的传输工作。其次，用virtio\_ring的框架，实现传输。我们现在说明设备探测和完整virtual I/O ABI的配置问题。

大部分全虚拟化hosts已经有了一些形式的PCI模拟，以及大部分guests有方法增加第三方的PCI驱动，很明显我们应该提供一个标准的virtio-over-PCI的定义，这样能给hosts和guests带来最大的便利。这是一个相对直接的vring实现，加上用一个I/O region的配置。比如，virtio\_pci 网络设备用了Linux内核中virtio\_net API和ABI中的结构体virtio\_net\_hdr。这些结构是专门被设计来这么用的，它也让这种透传的方式更加简单。

启动了KVM项目的Qumranet，贡献了他们的设备ID（vendor ID 0x1AF4）从0x1000到0x10FF。PCI设备子系统vendor和设备ids变成了virtio类型和vendor字段，所以PCI驱动不需要知道virtio类型的意义；在Linux里，这意味着它创建了一个virtio\_device的结构体，并把它注册到了virtio的总线上，virtio驱动可以来找到它。

I/O空间可能会需要特别的访问符，根据不通的平台，一般看起来像下面的结构体：

```
struct virtio_pci_io
{
    __u32 host_features;
    __u32 guest_features;
    __u32 vring_page_num;
    __u16 vring_ring_size;
    __u16 vring_queue_selector;
    __u16 vring_queue_notifier;
    __u8 status;
    __u8 pci_isr;
    __u8 config[];
}
```

发布和接收的bits是前2个32位的字段在I/O空间里：最后一个bit可以用来扩展它什么时候可用。vring\_queue\_selector用来得到设备的virtqueues：如果vring\_ring\_size是0的话，queue不存在。否则，guest期望写到它申请的queue的page地址，根据vring\_page\_size：这不像lguest的实现，host为ring申请空间。

vring\_queue\_notifier用来告诉host当一个queue有新buffer时，status字节用来写标准的virtio状态位，0用来重置设备。pci\_isr字段有一个副作用就是清空在读上的中断；非0意味着设备的一个virtqueue在等待了，第二个bit意味着设备的配置发生了变更。

ring自己包含了『guest-physical』的地址：不管是KVM还是lguest，都有一个简单的偏移从他们的地址到host进程的虚拟内存。所以host只需要检查地址是否超过了guest的内存大小就好了，然后就把地址对应的的内容转给readv和writev：如果碰到了一个内存空洞，就会返回一个-EFAULT，然后host可以给guest返一个错。

最后，实现一个PCI virtio驱动是一个相对简单的问题；它大概有450行代码，或者170个分号在Linux 2.6.25里。

### 7. PERFORMANCE

不好意思，当前阶段关于性能现在只有很少的数据，除了一些简单的测试确保我们的性能不是很差。对于我们把把目标设置成虚拟I/O在Linux guest跑在一个Linux host上来说没有任何阻碍：因为硬件的支持也提升了，我们期望接近于裸的速度。

当前网络性能非常受关注：支持多TSO的配置是当务之急，这可以去除掉KVM QEMU框架的一些复制。一旦完成了这个，我们期望探索控制通知来得到更多的注意：在接收的负载下我们的Linux驱动将会进入polling模式，这在标准的高性能NIC来说通常做法，但在包的传输上减少通知的方法仍然是很原始的。

### 8. ADOPTION

现在KVM和lguest都把virtio作为他们原生的传输层；对于KVM意味着，在32位和64位 x86，System z, IA64和PowerPC上支持Linux virtio驱动。lguest只是32位的x86，但已有patch将它扩展到了64位，这或许会在明年发布。

Qumranet对于Windows guest发布了beta版本的virtio\_pci驱动。KVM用QEMU模拟器去支持模拟设备，KVM版本的QEMU有virtio的host支持，但没有过多的优化；这是一个需要大量工作的领域。lguest的启动器也支持了一个很小的virtio实现。我们还不清楚其他的host实现，目前没有在内核的host实现，用来得到最后几个百分点性能的提升。

我们期望看到更多操作系统的virtio guest驱动，因为virtio驱动很容易写。

#### 8.1 Adaption Existing Transports to Use Virtio Drivers

你会注意到所有的例子驱动都写了一个定义的header在buffer的前面：这像一个ABI，对于KVM和lguest来说确实如此，就像他们被直接传到host一样。

然而，virtio驱动一个主要的目的是让他们在不同的传输层也能工作，通过配置不通的ops和feature bit就能实现。比如，如果网络驱动告诉host不支持TSO和checksum offload，整个网络header在发送时就可以忽略了，也能在接收时实现零拷贝。这会在add\_buf里面实现。如果header的格式不同，或者等价的信息要其他方式发送，也可以用这种观点解释。

我们面前的一个任务是为其他hypervisor创建一个shim层，可以跑测试和benchmark，来说服那些维护者原因切换到virtio的驱动上来。Xen的驱动是最有挑战性的：不只是他们被优化到了一个相当的程度，他们也有着相当多的功能还支持了一个断链和重连的模型，现在virtio的驱动还不能支持。

替换掉现有的驱动是一个微不足道的收益，但有两个场景我们认为virtio设施是非常有竞争力的。第一个是当虚拟化技术增加一个新的virtio已支持的虚拟设备类型时，适配的工作比从头重写一个简单太多了。比如，现在有一个可以给guest提供随机数的virtio entropy驱动，已经merge到了Linux 2.6.27版本了。

第二个场景是当新的虚拟化传输想要支持Linux；我们认识他们只要用了vring就可以了，或者至少能随便得到现有的驱动，至少比在他们的只是领域范围外实现和支持Linux驱动要简单的多。

### 9. FUTURE WORK

virtio和驱动仍在开发中；ABI在2.6.25中已经官方发布了，未来仍会做一些优化等工作。怎么在保证兼容性的前提下增加新功能是一个值得考虑的问题，现在进行的一些试验在未来可能也会增加进去。

#### 9.1 Feature Bits and Forward Compatibility

当然不是所有的host会支持所有的功能，不管是他们太老了还是不支持一些加速方法，比如checksum offload和TSO。

我们已经了解过了feature bit的机制，实现更值得提起：lguest和virtio\_pci用了两个bitmap，一个用来做host的feature表示，另一个用来表示driver接受的feature。当VIRTIO\_CONFIG\_S\_DRIVER\_OK状态位被设置了的时候，host可以检查接受了的feature集合，就可以看到guest驱动可以适配什么feature了。

目前为止所有定义的feature特定于一个设备类型，比如host在块设备里支持barrier。我们也为设备无关的feature预留了几个bits。因为我们不想随机的增加feature bits，对于复杂的guest和host交互场景来说，我们允许试验。

#### 9.2 Inter-guest Communication

让host支持guest间的交互非常简单；有一个试验的lguest的patch就是做这个事情。每个guest的启动进程映射了其他guest和自己的内存，然后用一个管道去通知其他启动guest间的I/O。guest协商用哪个virtqueue来加入，然后就很简单了：从guest的virtqueue中得到一个buffer，其他guest的virtqueue，根据buffer的flag是读还是写在它们之间拷贝数据。这个机制完全独立于virtqueue的用途；virtio\_net协议是对称的，所以不用guest的变动就可以在guest间点对点的网络通信。这个代码可以允许一个guest作为另一个guest的一个块设备或者终端设备，如果另外一个guest有对应的驱动的话。

对于不受信任传输的效率有一个细节但重要的协议考虑。设想下这个guest间的网络协议场景，guest收到一个声称有1514字节的包。如果所有的1514字节没有都被拷贝进来，接收的guest会把buffer里的老数据当成包剩余的部分。这个数据可能会泄露了用户的数据，或者来自于guest外部。为了防止这个问题，guest会清理所有的接收buffer，即使是麻烦和低效。

这也是为什么vring里的used ring包含了一个长度的字段。只要拷贝进来的数据是从一个被信任的数据源，我们可以避免做这件事。在lguest的原型里，启动器做了拷贝，所以可以信任它。一个virtio传输的实现只要连接到了host或者另一个可信任的数据源也可以只提供长度字段。如果传输层连接到了一个不被信任的数据源，也不能确保拷贝进来的数据长度是多少，那么他就必须在把buffer暴露到对端之前重置整块写buffer。这远比要求驱动来做安全的多，特别是传输层不会有这种问题。

当一个host为了guest间的I/O可以方便的联合起来两个vring，协商feature更麻烦一些。我们想要给每个guest提供所有的功能，这样它们可以享受到便利，比如TSO和checksum offload，但如果一个guest关闭了我们已经提供给其他guest的feature，我们要不就在每个I/O间做一些转换，要不就热插拔一下设备，改变一下提供的feature。

我们思考的方案是增加一个多轮的协商功能：如果guest知晓了功能，在驱动通过设置状态字段去通知feature后，它会希望feature被重新加上，知道最后多轮feature消失。我们会首先公布一个最小的feature集合，通过多轮的bit：如果所有的guest都知晓了，我们会公布所有的feature，然后依次的去掉有个别guest不接受的feature，直到大家都满意为止。这可能会是我们第一个非设备的feature bit。

#### 9.3 Tun Device  Vring Support

QEMU/KVM和lguest用户层的virtio host设备用的是Linux tun设备；这是一个用户层的网络接口，驱动读写来自于ethernet的网络包。我们提了一个简单的patch去使这个设备支持virtio\_ring header，这样我们可以测试vritio网络驱动的GSO的支持，Anthony Liguori和Herbert Xu把它引入到了KVM里。根据Herbert提供的数据，从guest到host的TCP流可以和Xen这种全虚拟化相比较，速度是guest回环设备的一半。

这个版本在传输时数据拷贝了两次：一次是在QEMU内部，一次在kernel里。前者是QEMU内部的架构限制，后者更难以解决。如果我们想要避免拷贝数据，我们在写操作时必须把用户层的页面pin住即使他们不被任何包引用。考虑到本地socket接收或者对于非GSO适配的设备有大包的拆分，创建一个析构回调不太可行。特别是，它会耗费非常长的时间来完成；如果没人来读一个包甚至会一直卡在本地socket里。

幸运的是，我们有一个方案来解决这个问题：vring！它可以处理乱序的buffer结束，很有效率，有一个设计的很好的ABI。因此我们有一个'/dev/vring'的Patch创建了一个文件描述符关联了一个vring ringbuffer到用户的内存中。这个文件描述符可以用来轮询（是否有buffer被用了），读（更新最后看见的index和清理轮询的flag）和写（告诉vringfd的另一端有新的buffer可以写了）。最后一个小patch增加了一些方法把vringfd附加到tun设备的接收和传输上。

我们也实现了一个ioctl去设置vring可以访问的offset和bounds；有了这个guest的网络vring可以直接暴露给host kernel的tap设备。终极的效率试验会是避免用户空间的操作，这会非常简单一但其他方面具备了的话。

### 10. CONCLUSIONS

就我们现在看到的这些虚拟I/O解决方案来说，通常的形式也清晰了：一个ring buffer，通知机制，一些feature bits。事实上，他们看起来更像高速物理设备：你可以认为他们可以用DMA的方式读或者写你的内存，你几乎不怎么和它们交互。这些实现的质量看起来更有趣，这意味着时间才是检验的唯一标准。

你将会确实的注意到virtio：它有中断，设备features，配置和DMA rings。有很多类似的概念比如driver作者，一个操作系统的设备架构用来去容纳他们。如果virtio驱动变成了Linux虚拟化环境下的标准，它会栓住了虚拟I/O设计的进程。没有这些指引就会变得非常混乱，集成到guest os变成了一个事后的想法，结果就是驱动看起来像是掉到了一个外星宇宙，代码也不容易看懂像是没有经过很好的训练。

在虚拟I/O里还有很多事情要做，但如果我们发布了一个坚固的基础，创新的过程就会加速，不用为了重新发明那些不感兴趣的配置和bit发布等功能而深陷泥潭。我们希望virtio会使人们相信他们可以完成一个虚拟I/O系统，可以支持未来的虚拟设备。

### 11. ACKNOWLEDGMENTS

作者感谢Muli Ben-Yehuda和Eric Van Hensbergen的参与，virtio值得一篇论文，Anthony Ligruori的校阅，操作系统reviewer的校阅。特别感谢所有的KVM开发者背后默默的支持。
