# NVMe PRP List

在分布式存储领域，一个存储 server 从网络上接收数据 buffer 并写入到 SSD 磁盘中。由于网络数据传输通常都是封包发送的，所以写盘的数据 buffer 往往并不是连续的内存。但如果每次 io 都只写一小部分数据又会很低效，所以需要一种高效的把多个分片数据一下传入下去的协议。

这种模式叫做 scatter gather，就是把多个地址不连续的数据段作为一个整体传输，在网络和存储介质领域都有很广泛的应用。

在 NVMe 协议上，有两种支持 scatter gather 机制的实现，一种是PRP list，就是把数据分为多个不连续的4k 对齐的 page，另一种是 sgl，把数据分为多个不等长的区段。但sgl 引入的比较晚，基本上所有的 NVMe SSD都支持 prp list，但只有最新的 SSD 才会支持 sgl。

#### PRP List

PRP 全程 Physical Region Page，一个 PRP entry 指向了一个4k对齐的 memory page。PRP 用来在 controller 和主存间用 scatter/gather 的模式传输数据。

物理内存页大小是 host配置在 CC.MPS 中的，一个 PRP entry尾部的n个 bits 是 pagesize，n+1到63是 page address，对于对齐的 page 来说尾部的 n 个 bits 都是0。

在 NVMe 协议中，PRP List 是 PRP entries 的集合，PRP list 描述了提供了额外的 PRP entry。

由于 NVMe command 的特殊定义，第一个 PRP entry 在 command 可能有一个非0的 offset。

dw6-dw9是 NVMe IO Command 里的 prp 部分

```
    union {
        struct {
            uint64_t prp1;   /* prp entry 1 */
            uint64_t prp2;   /* prp entry 2 */
        } prp;

        struct nvmed_sgl_descriptor sgl1;
    } dptr;
```

第一个 prp1记录第一个4k，如果数据只有8k，那么 prp2记录第二个4k，如果数据有12k，那么 prp2指向一个 prp list 的 buffer。
