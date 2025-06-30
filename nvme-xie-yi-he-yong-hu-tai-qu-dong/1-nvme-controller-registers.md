# 1 NVMe – Controller Registers

NVMe协议中NVMe SSD的一些寄存器记录了盘上的各种配置参数以及功能等信息，通过把PCI 的resouce mmap 到 host 内存中，就可以解析这些寄存器的信息。

在 linux 中，pci 的 resouce 位于/sys/bus/pci/devices/10001:02:00.0/resource0，通过把这个 文件 mmap 到内存中就可以直接访问寄存器的内容从而得到 SSD 的信息。

<figure><img src="../.gitbook/assets/image (4) (1).png" alt=""><figcaption><p>NVMe Express 1.1</p></figcaption></figure>

Controller registers位于 PCIe Conf space 里的 MLBAR/MUBAR register(BAR0/BAR1)，以 MMIO 的方式 mapping 到 host memory，所以 host software 可以通过 memory read/write 的方式来访问 controller register

#### 1 Offset 00h: CAP – Controller Capabilities

根据 NVMe 协议，第一个64bits寄存器是CAP 寄存器，记录了 SSD 控制器一些基本的信息。

<figure><img src="../.gitbook/assets/image (1) (1) (1) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

通过nvme show-regs -H /dev/nvme0 也可以得到这些信息

#### 2 Offset 08h: VS – Version

记录了 NVMe 协议的版本号

#### 3 Offset 14h: CC – Controller Configuration

controller configuration 寄存器

#### 4 Offset 1Ch: CSTS – Controller Status

#### 寄存器的定义在 spdk 的源码include/spdk/nvme\_spec.h 中也有完整的定义，类型spdk\_nvme\_cap\_register

