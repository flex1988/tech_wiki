# NVMe 协议 中断模式

中断模式可以让服务器用最少的资源消耗来处理设备的cmd 完成状态。

nvme 协议支持四种中断模式：

1. pin-based interrupt
2. single message MSI
3. multiple message MSI
4. MSI-x

MSI-x 是推荐使用的中断方式，它有最好的性能，最低的延时和最少的 CPU 消耗。

协议支持用中断聚合的方式来减少 host cpu 的消耗，中断聚合就是 host cpu 和延迟之间的权衡（admin cq 不会受到影响）。

**aggregation threshold filed** 定义了每个 vector basis 上最小的中断聚合区间

**aggregation time filed** 定义了当一个 cq entry 完成到singnal host 的最大延时

这两个值是由 host 提供给 controller的参考值，controller 发起中断的时间早于或者晚于这个时间都是允许的，而且不同厂商的实现也不同。
