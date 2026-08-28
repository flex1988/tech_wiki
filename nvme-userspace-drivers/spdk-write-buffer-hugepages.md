# SPDK 的写 buffer 为什么必须是来自于大页内存

{% embed url="https://spdk.io/doc/memory.html" %}

一般的计算机系统都把物理内存分成了一段段的 4kb 大小的page，然后每个进程的页表做了4kb 虚存到4kb物理内存的映射。

物理内存是和内存 channel 有关系的，每个内存 channel 会提供定量的内存带宽，为了优化内存带宽的使用，物理地址的分配会根据 channel 自动编排，比如 page0在 channel0，page1在 channel1，page2在 channel2，page3在 channel3，这样顺序写内存就自动利用了所有 channel 的带宽。实际上，自动编排在比 page 更粗的粒度上来做的。

现代的硬件平台的内存管理模块 MMU都支持虚存到实存转换的硬件加速，MMU 会支持多种不同的 page size。在最近的 x86系统上，支持4KB 2MB 1GB 的 page size，操作系统默认使用4kb 的 page size。

NVMe 设备和 host memory 传输数据一般使用 DMA，一般他们首先通过 PCI总线发送数据传输请求，没有 IOMMU 的时候，这些请求包含了物理内存地址。然后数据传输就在没有 CPU 参与的情况下发生了，MMU 负责内存数据的一致性。

NVMe 设备可能对于这些数据传输的物理内存 latyout 还有特殊的要求，NVMe1.0 SPEC 里要求所有的物理内存都能用一个 PRP列表的形式表示。

为了达到可以用 PRP 列表描述，内存必须有这些属性：

* 内存分裂为4kb 的物理page，我们称之为device page
* 第一个 device page 可以是一个从4B 对齐开始的不完全的 page
* 如果有多个 device page，第一个 page 必须在4kb 物理page 边界上结束
* 最后一个 device page 必须在4kb 的物理 page 边界上开始，但不要求在4kb 物理 page 边界上结束

协议允许 device page 比4kb 大，但已知的所有设备的 device page 都是4kb

NVMe 1.1协议里增加了对于弹性的 Scatter gatter list 的支持，但这个 feature 是可选的，当前仍有很多设备不支持

用户态的驱动在一个进程里运行可以访问到虚存，为了能够使用物理内存驱动设备，所以需要有一套虚存到物理内存的转换能力。

在 linux 上最简单的方式就是查询进程的`/proc/self/pagemap` 文件，这个文件里保存了虚存到实存的映射关系。在 linux4.0里，访问这些映射关系需要 root 权限。然而操作系统并不保证这个映射关系是固定的。操作系统不知道什么时候 PCI 的设备会通过 DMA 的方式传输数据，所以协调 DMA 和 page 的移动会非常难。当操作系统把虚存到物理内存的映射标记为无法更改，这称之为 page 的pinning。

虚存到实存的映射会变可能有这么几个原因，最常见的就是 page 到磁盘的 swap。在compaction 的过程中也有可能会移动 page，这个过程会把相同的虚存指向同一个实存以节省内存空间。一些操作系统还会做内存的透明压缩，这样由于新增的内存，会引起物理内存在多个 channel 上的 rebalancing。

posix 提供了 mlock 的系统调用，强制一个虚拟内存页使用物理内存页，但swapping 就无法运转了。然而这也无法保证虚存到物理内存的映射是静态的。mlock 的使用不能和 pin混用，posix没有定义 pin memory 的接口。所以申请 pin 内存是操作系统特有的。

SPDK 依赖于 DPDK 申请 pined memory，在 linux 上 DPDK 都是申请2MB 的大页内存。linux 处理大页内存和普通的4kb 内存不一样。特别的，操作系统从来不会更改它们的物理地址，当然这没有正式声明，而且在未来的版本有可能会变，但在当前和之前的数年都是这个样子。

所以上面就解释了为什么 SPDK 使用的内存 buffer 必须通过大页内存获得。

#### IOMMU

很多平台包含了一个特殊的硬件称为 IOMMU，它更像是一个 MMU，除了它向外围设备提供虚拟地址映射。MMU知道系统上每个进程虚存到实存的映射，由于 IOMMU 知道了这些映射关系后，就可以允许用户在虚存里映射任意的 bus 空间。所有的 DMA 操作都可以通过 IOMMU 先把 bus 地址转为虚存地址，然后把虚存地址转为物理地址。这也可以让操作系统更自由的更改虚存到实存的映射关系而且不影响正在进行的 DMA 操作。linux 提供了一个设备驱动，vfio-pci，允许一个进程配置它的 iommu。

