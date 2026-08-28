# PCIe Configuration Space

PCIe协议中的每个角色都有一个 Configuration space，包括 RootComplex 和 EndPoint Device，RootComplex 的 Configutation space位于host memory，Device的 configutation space 位于自己的 memory 内。

PCIe Configutation space可以通过多种方式访问，通过IO 寄存器或者 mem，以及在 linux 上可以通过 sysfs。

对于 EndPoint  device 来说 configuration space 都是 type0的，如下图：

<figure><img src="../.gitbook/assets/image (23).png" alt=""><figcaption><p>Type0 Configuration space</p></figcaption></figure>

