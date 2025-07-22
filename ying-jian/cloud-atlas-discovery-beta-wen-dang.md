# Cloud Atlas: Discovery beta 文档

### RDT管控L2/L3缓存和内存带宽[](cloud-atlas-discovery-beta-wen-dang.md#rdtl2-l3)

RDT技术，全称为Resource Director Technology，提供了两种能力：监控和分配。该技术旨在通过一系列的CPU指令从而允许用户直接对每个CPU核心（附加了HT技术后为每个逻辑核心）的L2缓存、L3缓存（LLC）以及内存带宽进行监控和分配。

Linux Kernel 4.10引入了Intel RDT实现架构，基于 `resctrl` 文件系统提供了 L3 CAT (Cache Allocation Technology)，L3 CDP(Code and Data Prioritization)，以及L2 CAT。并且Linux Kernel 4.12进一步实现支持了MBA(Memory Bandwidth Allocation)内存带宽分配技术。

Intel RDT提供了一系列分配(资源控制)能力，包括缓存分配技术(Cache Allocation Technology, CAT)，代码和数据优先级(Code and Data Prioritization, CDP) 以及 内存带宽分配(Memory Bandwidth Allocation, MBA)。

Intel志强处理器 E5-xxxx v4系列(即Broadwell)提供了L3缓存的配置以及CAT机制，其中部分通讯相关功能在 E5-xxxx v3系列(即Haswell)引入。一些Intel处理器系列(例如Intel Atom处理器系列)可能支持对L2缓存的控制。此外，MBA共功能提供了相应的处理器核心级别的内存带宽管理。

为了能够在Linux中使用资源分配技术，需要在内核和用户空间引入 `resctl` 接口。从Linux Kernel 4.10开始，可以使用 L3 CAT, L3 CDP 和 L2 CAT 以及 `resctrl` 架构。从Linux Kernel 4.12开始，开始引入并正在开发MBA技术。

### RDT技术架构[](cloud-atlas-discovery-beta-wen-dang.md#rdt)

缓存分配技术CAT(Cache Allocation Technology)的核心目标是基于服务级别(Class of Service, COS 或 CLOS)来实现资源分配。应用程序或者独立线程可以按照处理器提供的一系列服务级别来标记。这样就会按照应用程序和线程的服务分类来限制和分配其使用的缓存。每个CLOS可以使用能力掩码(capacity bitmasks, CBMs)来标志并在服务分类中指定覆盖(overlap)或隔离(isolation)的程度。

对于每个逻辑处理器，都有一个寄存器(被称为 `IA32_PQR_ASSOC` MSR或PQR)来允许操作系统(OS)或虚拟机管理器(VMM)在应用程序、线程、虚拟机(VM)调度(scheduled)的时候指定它的CLOS。

RDT分为5个功能模块：

* Cache Monitoring Technology (CMT) 缓存检测技术
* Cache Allocation Technology (CAT) 缓存分配技术
* Memory Bandwidth Monitoring (MBM) 内存带宽监测
* Memory Bandwidth Allocation (MBA) 内存带宽分配
* Code and Data Prioritization (CDP) 代码和数据优先级

备注

RDT技术针对的是缓存和内存带宽，分别又分为监控和控制，就形成了4个功能模块，再加上代码和数据优先级(控制技术)，合起来形成5个功能模块。

RDT允许OS或VMM监控线程、应用或VM使用的cache/内存带宽，通过分析cache/内存带宽使用率，OS或VMM可以优化调度策略提高效能，使得高级优化技术可以实现。

备注

实时监控缓存和内存带宽，处理器可以把资源分配给虚拟机的最重要、最紧迫的任务，或者分配给最重要的容器。例如在线和离线应用混布的场景就非常依赖RDT技术，在发生缓存和内存带宽竞争时优先把资源分配给在线应用。

Intel RDT功能随着Intel几代服务器芯片发展而不断扩展，在Haswell上只能简单在几个预设定的LLC (末级高速缓存，Last Level Cache) partition之间切换，而到最近的Skylake上已经实现了自主编程的LLC和内存带宽分配（全面控制）。

#### RDT术语[](cloud-atlas-discovery-beta-wen-dang.md#id2)

* RMID

OS或VMM会给每个应用或虚拟机标记一个软件定义的ID，叫做RMID（Resource Monitoring ID），通过RMID可以同时监控运行在多处理器上相互独立的线程，注意这里是指应用线程而是不是硬件的core。每个处理器可用的RMIDs数量是不一样的，这个可以通过CPUID指令获取得到，RMID数量不一样意味着可以监控独立线程的数量会有差异，如果一个系统上运行过多的线程可能会导致不能监控到所有线程的资源使用。

此外线程可以被独立监控，也可以按组的方式进行监控：

* 多个线程可以标记为相同的RMID
* 同一个虚拟机下的所有线程可以标记为相同的RMID
* 同样一个应用下的所有线程可以标记为相同的RMID

绑定RMID到线程的动作由OS/VMM来完成。见下文的使用 `IA32_PQR_ASSOC` MSR 来关联 RMID。

备注

每个CPUcore上存在一个 `IA32_PQR_ASSOC` MSR

获取监控数据也是通过MSR来实现的: `IA32_QM_EVTSEL` 设置RMID和Event ID，硬件就会查看特定数据，并通过 `IA32_QM_CTR` MSR返回结果。这个 `IA32_QM_CTR` MSR的 `E/U` 位表示 `Error` 和 `Unavailable` ，如果数据合法就不会设置这两个位，则数据就可以被软件使用。

* CLOS

CAT中引入了一个中间结构叫做 `CLOS` (Class of Service)，可以理解为资源控制标签。此外每个CLOS定义了CBM（capacity bitmasks），CLOS和CBM一起，确定有多少cache可以被这个CLOS使用。

### 高速缓存分配技术(CAT)[](cloud-atlas-discovery-beta-wen-dang.md#cat)

在云计算环境，多租户虚拟机会运行多种不同类型的应用，所以确保一致的性能和优先级划分确保重要应用运行是巨大的挑战。在多核处理器系统中，共享资源，例如末级高速缓存(LLC，Last Level Cache)、共享IO设备、共享内存带宽的分配和使用是关系到应用性能的关键。



一些应用（如后台视频流和转码应用）会过度使用高速缓存，导致降低更重要应用的性能。例如下图中 `Noisy neighbor` 的 `App[0]` （运行在CPU核心0上）消耗了过多的末级高速缓存，影响了CPU核心1上运行的 `App[1]` 。这是因为通常根据先到先得的分配原则：



高速缓存分配技术（CAT）提供了软件可编程控制，以控制特定线程、应用、虚拟机或容器等消耗的高速缓存空间。可支持操作系统保护重要的进程，支持管理程序即使在 `noisy` 环境中也可以对重要虚拟机进行优先级划分。

CAT基本机制：

* 通过CPUID枚举CAT功能和相关LLC分配支持的能力
* 支持操作系统/管理程序将应用划分成不同服务类(CLOS)并为不同CLOS指定可用末级高速缓存量的接口。这些接口都基于特定型号寄存器（MSR）

备注

Intel在Haswell志强处理器首次引入CAT L3功能，并且在后续的Broadwell和Skylake系列上得到增强改进。未来x86处理器还将引入CAT L2功能，对共享的L2缓存进行类似的分配管理技术。

#### CAT硬件架构[](cloud-atlas-discovery-beta-wen-dang.md#id3)

#### CAT技术的关键概念[](cloud-atlas-discovery-beta-wen-dang.md#id4)

* CLOS

高速缓存分配技术引入一种名为服务类(CLOS)的中间接口，可以为资源控制标记，线程/应用/虚拟机/容器在该标记内进行分组。CLOS包含相关资源容量位掩码(CBM)，来说明特定的CLOS能够使用多少高速缓存。



* 使用模式

通过高速缓存分配技术（Cache Allocation Technology, CAT) 功能提供的可伸缩接口可以创建出大量的 使用模式，包括对重要应用程序的优先级以及隔离应用程序降低干扰。

在使用CAT功能的底层软件，例如OS或VMM使用如下步骤实现:

* 通过CPUID检查CPU是否支持CAT: CPUID 的leaf （最末端值？）0x10 提供了CAT功能的能力的详细信息
* 配置服务分类（the class of service, CLOS）定义了通过MSRs可提供的资源范围（缓存空间）
* 每一个逻辑线程都有响应的一个可用的逻辑CLOS
* 当OS/VMM将一个线程或VCPU加载到CPU核心中，将通过 `IA32_PQR_ASSOC` MSR来更新CPU核心的CLOS，以确保这个资源是通过步骤2（配置的服务分类）来控制其使用的资源

更高层次的软件，例如一个调度框架（Kubernetes）或者管理员层次的工具可以通过 `OS/VMM` 激活硬件能力，可以参考 [Software Enabling for Cache Allocation Technology in the Intel® Xeon® Processor E5 v4 Family](https://software.intel.com/en-us/articles/software-enabling-for-cache-allocation-technology) 文档的设置方法。

对于给定应用程序指定可用的缓存是通过MSR所包含 `能力掩码` ( capacity bitmasks, CBMs )来设置的：

在CBMs中的值表示了可用缓存量以及重叠或隔离程度。例如下图CLOS\[1]的可用缓存小于CLOS\[3]，即优先级较低：

在末端高速缓存（LLC）中，如果应用程序没有相互覆盖或者VM没有竞争缓存空间的情况下，系统不会使用独立的缓存分区，而是可以动态更新任何需要修改的资源。

使用覆盖位掩码（overlapping bitmasks）（在上图Figure 2中的CLOS\[2]和CLOS\[3]）通常可能比隔离情况更能达到较高的带宽，并且依然具备了相关优先级：因为比使用完全隔离的分区，可以动态按需更新资源可以获得更大的LLC。这可能是适合很多线程/应用/VM并发运行的模型。

关联软件线程和CLOS是通过 `IA32_PQR_ASSOC` MSR实现的，为每个硬件线程做了定义：

另一种可选的方法是不激活操作系统和VMM的CLOS直接pin到硬件线程的方式，而是采用软件线程pin到硬件线程上；不过建议激活OS/VMM的方式避免需要pin应用线程。

在开始评估截断，pin模式可以通过 [RDT工具](https://github.com/01org/intel-cmt-cat) 来实现，这个工具提供了Linux系统的线程监控和通过关联 [Resource Monitoring IDs (RMIDs)](https://software.intel.com/en-us/blogs/2014/12/11/intel-s-cache-monitoring-technology-software-visible-interfaces) 和 每个硬件线程的 [CLOS](https://software.intel.com/en-us/articles/introduction-to-cache-allocation-technology) 控制资源使用。

### CAT技术的应用场景[](cloud-atlas-discovery-beta-wen-dang.md#id6)

CAT缓存分配技术在很多领域有广泛适应性，具备动态更新的伸缩和重叠(overlapped)、隔离(isolated)配置，可以将一个设备在不同应用领域轮转共享使用：

* 数据中心的云计算主机 - 在同时运行着 `noisy neighbors` 的主机上保障重要虚拟机或容器的资源使用
* 公有/私有云 - 保护重要的基础架构VM（例如VPN to bridge连接私有和公有云）能够提供稳定的网络服务
* 数据中心基础架构 - 确保虚拟交换机能够稳定服务
* 通讯 - 确保网络应用的性能和后台任务稳定运行
* 内容分发（CDN） - 提供内容分发应用的带宽稳定
* 网络 - 基于 [DPDK](http://dpdk.org/) 的高性能应用能够不受 `noisy neighbor` 干扰
* 工业控制 - 实时环境确保重要代码部分能够符合要求稳定运行

### 代码和数据优先级(CDP)[](cloud-atlas-discovery-beta-wen-dang.md#cdp)

代码和数据优先级(Code and Data Prioritization, CDP)技术是CAT技术的扩展，提供了代码和数据的隔离以及区分优先级，也是通过CLOSID来实现代码和数据掩码的隔离。CDP技术最早从Broadwell系列志强处理器引入。

### RDT实战[](cloud-atlas-discovery-beta-wen-dang.md#id7)

RDT使用分为两种方式:

* 直接将RMID绑定到硬件线程，然后将应用绑定到这些线程
* 使能OS/VMM调度（需要内核支持），在进程切换时候会自动将RMID进行更新，能够支持线程迁移。

使用Intel开源工具 [intel-cmt-cat](https://github.com/intel/intel-cmt-cat) 可以不需要内核支持，直接使用 CAT,CMT,MBM,CDP功能。

#### Intel开源RDT工具intel-cmt-cat[](cloud-atlas-discovery-beta-wen-dang.md#intelrdtintel-cmt-cat)

```
make && make install
```

如果找不到动态链接库，则指定: `export LD_LIBRARY_PATH=/usr/local/lib`

RDT工具 `pqos` ，运行在用户层，通过标准Linux工具访问MSR寄存器，需要root用户权限。支持在每个core或线程上提供CMT和MBM，其中MBM包括本地和异地内存。目前在 RHEL 7操作系统，通过 `intel-cmt-cat` 软件包提供了 `pqos` 工具，用于控制Intel处理器CPU缓存和内存带宽。( [Red Hat Enterprise Linux 7 Performance Tuning Guide 2.14. pqos](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-performance_monitoring_tools-pqos) ) 。根据 [pqos manual(8)](https://manpages.debian.org/unstable/intel-cmt-cat/pqos.8.en.html) 可以看到， `pqos` 工具同时支持 Intel RDT 和 AMD PQoS。

#### 内核方式使用RDT[](cloud-atlas-discovery-beta-wen-dang.md#id8)

虽然传统上通过Intel开源RDT工具intel-cmt-cat来直接访问MSR设置RDT，但是也可以通过操作系统内核方式使用RDT，需要内核支持。

*   确认kernel和CPU均支持CAT:

    ```
    cat /proc/cpuinfo | grep cat_l3
    ```

### AMD PQoS 和ARM MPAM[](cloud-atlas-discovery-beta-wen-dang.md#amd-pqos-arm-mpam)

AMD在Zen处理器二代架构支持和RDT对等技术PQoS，并且已经被内核支持 - [AMD Publishes Platform QoS Patches For Next-Gen Processors](https://www.phoronix.com/scan.php?page=news_item\&px=AMD-Platform-QoS-RFC-Patches) 。如上文所述， `pqos` 工具是同时支持Intel RDT和AMD PQoS技术的，两者兼容。

ARM架构处理器对应有MPAM技术(Memory Partitioning and Monitoring)，不过该技术起步较晚，目前尚未有完善的用户空间管控工具( `resctrl` 还不支持ARM架构 )。

### 参考[](cloud-atlas-discovery-beta-wen-dang.md#id9)

* [Resource Allocation: Intel Resource Director Technology (RDT) by Fenghua Yu, Intel](https://www.youtube.com/watch?v=rKe5_xWpH8o) - Intel的FengHua Yu在Linux Foundation上演讲介绍RDT视频，可参考
* [Resource Allocation: Intel Resource Director Technology (RDT)](https://events.static.linuxfound.org/sites/events/files/slides/cat8.pdf) - Intel的FengHua Yu的演讲PPT，举了不少形象的例子
* [Intel RDT 三级缓存管理技术](https://zhuanlan.zhihu.com/p/29432536)
* Intel RDT特性详解
* [英特尔® 资源调配技术 (英特尔® RDT)](https://www.intel.cn/content/www/cn/zh/architecture-and-technology/resource-director-technology.html)
* [英特尔® 至强™ 处理器 E5 v4 产品家族的高速缓存分配技术简介](https://software.intel.com/zh-cn/articles/introduction-to-cache-allocation-technology?_ga=2.21223931.1997524624.1555917558-1194351684.1555901060)
* [Intel CMT & CAT & CDP 技术](https://blog.csdn.net/force_eagle/article/details/77197833) 这篇blog提供了很多技术文档参考链接
* [AMD64 Technology Platform Quality of Service Extensions](https://developer.amd.com/wp-content/resources/56375.pdf)
* [Kernel 4.14+ Intel RDT / AMD PQOS配置](https://zhuanlan.zhihu.com/p/92125001)
