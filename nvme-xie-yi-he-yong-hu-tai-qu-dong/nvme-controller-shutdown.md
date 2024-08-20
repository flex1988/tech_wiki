# NVMe Controller shutdown

NVMe控制器有两种 shutdown 方法，一种是normal shutdown，还有一种是abrupt shutdown。

#### normal shutdown

1. host停止提交 io command，以及允许 outstanding io 完成
2. host用删除 io submission queue 命令删除所有 submission queue，删除 submission queue 的结果就是所有的 outstanding io被终止
3. host 删除所有的 io completion queue
4. host 设置 cc.shn为01b，表明这是一个 normal shutdown。控制器通过把csts.shst 设置为01b 来表明 shutdown 完成

#### abrupt shutdown

1. 停止提交新请求
2. host 设置 cc.shn 字段到10b 表明一个 abrupt shutdown，控制器通过把 csts.shst 设置为01b 来表明 shutdown 完成
