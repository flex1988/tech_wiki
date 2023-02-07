# virtio: Towards a De-Facto Standard For Virtual I/O Devices

#### 摘要

Linux内核现在支持了最少8种虚拟化系统：Xen，KVM，VMware's VMI，IBM的System p，IBM System z，用户态Linux，lguest和IBM iSeries。看起来会出现更多类似的系统，每个系统都有它独立的块设备，网络，终端以及不同功能和优化的驱动。

一个尝试是virtio：一系列高效、维护很好的Linux驱动，用一层很薄的抽象就可以被多种hypervisor所使用。对于每个驱动它包含了一个简单的可扩展的机制。我们还提供了一个ring buffer传输的实现，和KVM，lguest用的ring buffer一样。对于新的hypervisor来说这种方法副作用很小，只要支持了这种高效的传输机制就能立马减少大量的工作量。最后，我们提供了一个把vring和设备配置成PCI设备的实现：这意味着guest os不需要增加一个新的PCI驱动，hypervisor只需要在他们实现的虚拟设备中支持vring就可以了（现在只有KVM这么做了）。

本文将会说明Linux中vritio的API层的实现，vring的实现，最后在全虚拟化guest作为PCI设备的体验。我们将会包含一些其他工作把这种I/O机制深入集成到Linux host kernel中。

#### 1. 介绍
