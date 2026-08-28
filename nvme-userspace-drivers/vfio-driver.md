# vfio 驱动

vfio 驱动是一个新的用户态驱动框架，vfio本身作为一个内核模块，可以为用户态驱动提供 DMA 和 interrupt 等特性，一般用在虚拟化等场景，可以实现设备直通和设备隔离。

在底层存储领域，也可以使用 vfio 来实现驱动的中断功能。

在 nvme协议中，中断主要有三种方式：

1. intx 中断，性能很差
2. msi 中断，性能好，使用限制大，只支持32个中断，中断号必须连续等
3. msix 中断，作为 msi 中断的增强，性能好，限制小，支持最多1024个中断，中断号可不连续

用户态驱动如果使用了 vfio-pci 驱动就可以使用 msix 中断了，msix 中断是目前性能最好的中断功能。

内核 vfio驱动目录结构：

```
drivers/vfio/
├── Kconfig
├── Makefile
├── mdev
│   ├── Kconfig
│   ├── Makefile
│   ├── mdev_core.c
│   ├── mdev_driver.c
│   ├── mdev_private.h
│   ├── mdev_sysfs.c
│   └── vfio_mdev.c
├── pci
│   ├── Kconfig
│   ├── Makefile
│   ├── trace.h
│   ├── vfio_pci.c
│   ├── vfio_pci_config.c
│   ├── vfio_pci_igd.c
│   ├── vfio_pci_intrs.c
│   ├── vfio_pci_nvlink2.c
│   ├── vfio_pci_private.h
│   └── vfio_pci_rdwr.c
├── platform
│   ├── Kconfig
│   ├── Makefile
│   ├── reset
│   │   ├── Kconfig
│   │   ├── Makefile
│   │   ├── vfio_platform_amdxgbe.c
│   │   ├── vfio_platform_bcmflexrm.c
│   │   └── vfio_platform_calxedaxgmac.c
│   ├── vfio_amba.c
│   ├── vfio_platform.c
│   ├── vfio_platform_common.c
│   ├── vfio_platform_irq.c
│   └── vfio_platform_private.h
├── vfio.c
├── vfio_iommu_spapr_tce.c
├── vfio_iommu_type1.c
├── vfio_spapr_eeh.c
└── virqfd.c
```

mdev 目录是 mediated device 功能，
