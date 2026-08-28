# 通过sysfs 访问 PCI 设备资源

{% embed url="https://docs.kernel.org/PCI/sysfs-pci.html" %}

sysfs 通常挂在到/sys 目录，提供了平台对于 PCI 资源的访问，一个服务器上的 PCI 长这个样子，路径里的10000:01:00.0就是 pci 设备的地址。

```
[root@TENCENT64 ~]# ll /sys/bus/pci/devices/10000:01:00.0/
total 0
-r--r--r-- 1 root root  4096 May 20 19:26 aer_dev_correctable
-r--r--r-- 1 root root  4096 May 20 19:26 aer_dev_fatal
-r--r--r-- 1 root root  4096 May 20 19:26 aer_dev_nonfatal
-r--r--r-- 1 root root  4096 May 20 19:26 ari_enabled
-rw-r--r-- 1 root root  4096 May 20 19:26 broken_parity_status
-r--r--r-- 1 root root  4096 May 11 16:06 class
-rw-r--r-- 1 root root  4096 May 11 16:06 config
-r--r--r-- 1 root root  4096 May 20 19:26 consistent_dma_mask_bits
-r--r--r-- 1 root root  4096 May 20 19:26 current_link_speed
-r--r--r-- 1 root root  4096 May 20 19:26 current_link_width
-rw-r--r-- 1 root root  4096 May 20 19:26 d3cold_allowed
-r--r--r-- 1 root root  4096 May 11 16:06 device
-r--r--r-- 1 root root  4096 May 20 19:26 dma_mask_bits
lrwxrwxrwx 1 root root     0 May 12 02:04 driver -> ../../../../../../bus/pci/drivers/nvme
-rw-r--r-- 1 root root  4096 May 20 19:26 driver_override
-rw-r--r-- 1 root root  4096 May 20 19:26 enable
-r--r--r-- 1 root root  4096 May 11 16:06 irq
-r--r--r-- 1 root root  4096 May 20 19:26 local_cpulist
-r--r--r-- 1 root root  4096 May 20 19:26 local_cpus
-r--r--r-- 1 root root  4096 May 20 19:26 max_link_speed
-r--r--r-- 1 root root  4096 May 20 19:26 max_link_width
-r--r--r-- 1 root root  4096 May 12 02:04 modalias
-rw-r--r-- 1 root root  4096 May 20 19:26 msi_bus
drwxr-xr-x 2 root root     0 May 20 19:26 msi_irqs
-rw-r--r-- 1 root root  4096 May 12 02:04 numa_node
drwxr-xr-x 3 root root     0 May 11 16:11 nvme
-r--r--r-- 1 root root  4096 May 20 19:26 pools
drwxr-xr-x 2 root root     0 May 20 19:26 power
--w--w---- 1 root root  4096 May 20 19:26 remove
--w------- 1 root root  4096 May 20 19:26 rescan
--w------- 1 root root  4096 May 20 19:26 reset
-r--r--r-- 1 root root  4096 May 11 16:06 resource
-rw------- 1 root root 32768 May 20 14:48 resource0
-r--r--r-- 1 root root  4096 May 20 19:26 revision
-rw------- 1 root root 65536 May 20 19:26 rom
-rw-r--r-- 1 root root  4096 May 20 19:26 sriov_drivers_autoprobe
-rw-r--r-- 1 root root  4096 May 20 19:26 sriov_numvfs
-r--r--r-- 1 root root  4096 May 20 19:26 sriov_offset
-r--r--r-- 1 root root  4096 May 20 19:26 sriov_stride
-r--r--r-- 1 root root  4096 May 20 19:26 sriov_totalvfs
-r--r--r-- 1 root root  4096 May 20 19:26 sriov_vf_device
lrwxrwxrwx 1 root root     0 May 11 16:45 subsystem -> ../../../../../../bus/pci
-r--r--r-- 1 root root  4096 May 20 19:26 subsystem_device
-r--r--r-- 1 root root  4096 May 20 19:26 subsystem_vendor
-rw-r--r-- 1 root root  4096 May 20 19:26 uevent
-r--r--r-- 1 root root  4096 May 11 16:06 vendor
```

目录里面的每个文件都对应他们自己的 function



| file                 | function                                                                                         |
| -------------------- | ------------------------------------------------------------------------------------------------ |
| class                | PCI class (ascii, ro)                                                                            |
| config               | PCI config space (binary, rw)                                                                    |
| device               | PCI device (ascii, ro)                                                                           |
| enable               | Whether the device is enabled (ascii, rw)                                                        |
| irq                  | IRQ number (ascii, ro)                                                                           |
| local\_cpus          | nearby CPU mask (cpumask, ro)                                                                    |
| remove               | remove device from kernel’s list (ascii, wo)                                                     |
| resource             | PCI resource host addresses (ascii, ro)                                                          |
| resource0..N         | PCI resource N, if present (binary, mmap, rw[1](https://docs.kernel.org/PCI/sysfs-pci.html#id2)) |
| resource0\_wc..N\_wc | PCI WC map resource N, if prefetchable (binary, mmap)                                            |
| revision             | PCI revision (ascii, ro)                                                                         |
| rom                  | PCI ROM resource, if present (binary, ro)                                                        |
| subsystem\_device    | PCI subsystem device (ascii, ro)                                                                 |
| subsystem\_vendor    | PCI subsystem vendor (ascii, ro)                                                                 |
| vendor               | PCI vendor (ascii, ro)                                                                           |

```
ro - read only file
rw - file is readable and writable
wo - write only file
mmap - file is mmapable
ascii - file contains ascii text
binary - file contains binary data
cpumask - file contains a cpumask type
```
