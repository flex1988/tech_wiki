# SPDK介绍

SPDK是 INTEL 开发一套开源存储软件栈，全称是 Storage Performance Development Kit。

<figure><img src="../.gitbook/assets/image (24).png" alt=""><figcaption></figcaption></figure>

SPDK 主要由三层组成，最底层是驱动层，包括 NVMe 驱动和 NVMe-oF 等，以及 Intel QAT。

中间层是存储服务层，最核心的部分是提供了一个块设备的抽象层。

最上层是存储协议层，包括 iSCSI，nbd，NVMe-oF 等，通过该层，主机端可以通过存储协议以远程的方式访问 NVMe 提供的存储功能。

从代码结构来看，核心的三个目录是 app, lib, module。

app 包含一些应用程序的源码，比如 iscsi\_tgt, nvmf\_tgt spdk\_dd 等子目录，每个目录都可以编译为一个独立的应用程序。

lib 目录是 spdk 提供的核心代码，这些核心功能都是以 lib 的形式对外提供。

module 和加密 逻辑卷 RAID AIO Ceph 等有关。
