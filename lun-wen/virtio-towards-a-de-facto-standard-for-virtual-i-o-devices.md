# virtio: Towards a De-Facto Standard For Virtual I/O Devices

#### 摘要

Linux内核现在支持了最少8种虚拟化系统：Xen，KVM，VMware's VMI，IBM的System p，IBM System z，用户态Linux，lguest和IBM iSeries。看起来会出现更多类似的系统，每个系统都有它独立的块设备，网络，终端以及不同功能和优化的驱动。

一个尝试是virtio：一系列高效、维护很好的Linux驱动，用一层很薄的抽象就可以被多种hypervisor所使用。对于每个驱动它包含了一个简单的可扩展的机制。我们还提供了一个ring buffer传输的实现，和KVM，lguest用的ring buffer一样。对于新的hypervisor来说这种方法副作用很小，只要支持了这种高效的传输机制就能立马减少大量的工作量。最后，我们提供了一个把vring和设备配置成PCI设备的实现：这意味着guest os不需要增加一个新的PCI驱动，hypervisor只需要在他们实现的虚拟设备中支持vring就可以了（现在只有KVM这么做了）。

本文将会说明Linux中vritio的API层的实现，vring的实现，最后在全虚拟化guest作为PCI设备的体验。我们将会包含一些其他工作把这种I/O机制深入集成到Linux host kernel中。

#### 1. 介绍

Linux kernel被移植到了很多平台；官方的kernel树包含了24种架构相关的目录和大约200万行架构相关的代码。这些代码大部分都是在支持不同的平台变体。不幸的是，我们发现只有一种架构从内核中删掉了，但同时新增加的硬件变种像雨后的春笋一样多。每天大约有1万行的代码改动，这是你能想象的事。

如果我们把Linux看做一个虚拟化下的guest，我们会特别幸运：支持IBM的 System p，System z和遗产iSeries。用户态的Linux早就被包含在内了，就是在Power,IA64 和 32位,64位的x86机器上把Linux运行为一个用户态的进程。过去两年，x86被证明非常有竞争力，有了来自于XenSource的Xen的支持，VMware VMI的支持以及Qumranet KVM的支持。最后，我要提一下我自己的一个贡献，lguest：一个玩具hypervisor，可以用来开发或者教学，也悄悄的溜到主干代码里了。

8个平台的每一个都想要他们自己的块，网络和终端驱动，有时还有framebuffer，USB控制器，host文件系统，虚拟的kitchen sink控制器等。他们很少会去优化他们的驱动，又提供了基本重叠但有微小区别的功能。重要的是，没有人乐意去管他们的驱动，或者维护驱动。

这个问题在KVM项目里越来越明显，当它在2006年突然跳入Linux的视野中时，仍然没有一个paravirtual设备模型。模拟设备的性能限制编的越来越明确，不管是移植Xen的驱动模型还是开发另一个驱动模型都不是很容易实现。在Xen设备模型的工作过程中，我们相信为各种类型的hypervisor和平台创建一个通用的虚拟I/O机制会很有效率，也是对于把Xen的设备配置系统引入的一种赎罪。

#### 2. VIRTIO: THE THREE GOALS

我们最初始的目标非常明确：所有的工作都在Linux kernel内部，不需要其他人在做一些事情。如果嵌入式虚拟I/O机制的开发者对Linux熟悉，它可能会引导他们映射Linux的API到他们自己的ABI。但是『如果』和『可能』是不充分的：我们对这些情况会更有耐心。

经验显示高级的传输机制更倾向于不止针对一个hypervisor和架构，而是各个种类的设备。所以接下来一个明确的目标就是提供一个公共的ABI和buffer的使用。更加小心的是，我们的virtio\_ring实现并没有什么技术革新：开发者看下代码就会发现没有什么不喜欢的。

最后，我们提供了两种ABI的实现，对虚拟I/O设备来说用virtio\_ring设施和Linux API。这些实现了虚拟I/O的最后一部分：设备探测和配置。重要的是，他们证明了用Linux 虚拟I/O API来提供功能以及向前向后兼容是多么的容易，所以未来的Linux 驱动功能都可以被探测和使用对任何host实现来说。

明确的驱动分离，传输和配置代表了我们从当前实现的一些思考。比如，你不能把Xen的Linux网络驱动用在一个新的hypervisor上除非你支持了Xen-Bus的探测和配置系统。



#### 3. VIRTIO: A LINUX-INTERNAL ABSTRACTION API

如果我们想在虚拟驱动设备上减少重复的工作，我们需要一个直接的抽象层，这样驱动就可以共享代码了。一种方法是提供一套给虚拟驱动使用的公共函数，另一种更耗费精力的方法是用一个公共的驱动和一套操作结构：一系列函数指针用于实际的传输实现的接口。任务是为了创建一个所有虚拟设备的传输层抽象，能很好去优化传输，也允许已一个较小的代价从已有的传输过渡过来。

当前的结果就是virtio驱动把他们注册到了一个32位的设备类型，可以选择性的拒绝一个32位的vendor。合适的virtio设备被发现时会调用驱动的probe函数：传入的结构体virtio\_device有一个virtio\_config\_ops的指针，可以被用来解析设备配置。

配置操作可以被分为4个部分：读和写功能bits，读和写配置的空间，读和写状态bits，设备重置。设备查找device-type-specific feature bits符合他想要的功能，比如VIRTIO\_NET\_F\_CSUM feature bit表明是否支持网络设备checksum的offload。Feature bits是非常明确的：host知道guest返回的feature bits，然后可以决定driver需要理解哪些feature。

第二部分是配置空间：这是一个和虚拟设备规格信息相关联的结构。他们可以被guest读或者写。比如，网络设备有一个VIRTIO\_NET\_F\_MAC的feature bit，表明host想要设备有一个独立的MAC地址，然后配置空间包含了这个值。

