# 1 NVMe – Controller Capabilities

NVMe协议中NVMe SSD的一些寄存器记录了盘上的各种配置参数以及功能等信息，通过把PCI 的resouce mmap 到 host 内存中，就可以解析这些寄存器的信息。

在 linux 中，pci 的 resouce 位于/sys/bus/pci/devices/10001:02:00.0/resource0，通过把这个 文件 mmap 到内存中就可以直接访问寄存器的内容从而得到 SSD 的信息。

<figure><img src="../.gitbook/assets/image.png" alt=""><figcaption><p>NVMe Express 1.1</p></figcaption></figure>

根据 NVMe 协议，第一个64bits寄存器是CAP 寄存器，记录了 SSD 控制器一些基本的信息。

<figure><img src="../.gitbook/assets/image (1).png" alt=""><figcaption></figcaption></figure>

通过nvme show-regs -H /dev/nvme0 也可以得到这些信息。
