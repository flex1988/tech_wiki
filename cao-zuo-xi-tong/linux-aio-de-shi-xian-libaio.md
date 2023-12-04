# Linux AIO的实现Libaio

#### 介绍

Asynchronous I/O（AIO）是一种异步执行I/O操作的方法，用户进程发起I/O请求后不用等待I/O完成，可以去做其他事情，同时不停轮训I/O结果，直到I/O完成后异步执行接下来的操作。

Linux上实现异步I/O的几种方式：

1. 内核系统调用
2. 封装了内核系统调用的库Libaio
3. 线程池模拟的异步I/O
4. spdk
5. io\_uring，依赖高版本内核

在分布式存储领域应用最多的是2和4，这里我们只研究Libaio的使用方式。

#### ABI 接口

Linux内核为了支持异步I/O提供了5个系统调用

```
// io_setup创建异步io context，nr_events是指能够并发处理的io数
int io_setup(unsigned nr_events, aio_context_t* ctx);
// io_destroy尝试取消io context关联的io，并且销毁io context
int io_destroy(aio_context_t ctx);
// 提交一组请求
// 参数1: io context
// 参数2: io数
// 参数3: 每个io关联一个iocb，此处传入nr个连续的iocb地址
// struct iocb
// {
//     __u64 aio_data; // 用户传入的字段，在event中返回，可以用来作为callback
//     __u32 aio_key;  // flags
//     __u16 aio_lio_opcode;  // io类型，读、写、fsync
//     __s16 aio_reqprio;     // io优先级，没有作用？
//     __u32 aio_fildes;      // 打开文件的fd
//     __u64 aio_buf;         // io buf
//     __u64 aio_nbytes;      // io length
//     __s64 aio_offset;      // io offset
//     __u64 aio_reserved2;
//     __u32 aio_flags;
//     __u32 aio_resfd;       // 用来配合eventfd实现event mode
// }
// 返回值
// ret == nr，所有的请求提交成功
// ret > 0 && ret < nr，提交了成功前ret个请求
// ret < 0，一个没有提交成功，有可能是提交前参数不对，或者提交第一个的时候出错
int io_submit(aio_context_t ctx, long nr, struct iocb** iocbs);
// 试图取消一个io_submit提交的io
int io_cancel(aio_context_t ctx, struct iocb*, struct io_event* result);
// 轮训事件
// 参数1: io context
// 参数2: 最小的事件数，少于min_nr个事件可能会hang住线程
// 参数3: 最大的事件数
// 参数4: events事件数组，events数组最少要有nr个元素
// 参数5: 超时时间，当一直不满足min_nr时要不要返回，为空时一直hang住
// 返回值
// ret == nr，所有的events数据都得到了事件，内核可能有更多的事件未处理
// min_nr <= ret <= nr，get到ret个事件，内核中没有剩余完成事件了
// 0 < ret < min_nr，超时了，只有ret个事件
// ret == 0，出错了
int io_getevents(aio_context_t ctx, long min_nr, long nr, struct io_event* events,
    struct timespec* timeout);
```

#### 关键数据结构

```
struct io_event
{
    __u64    data;  // iocb传入的data字段
    __u64    obj;   // iocb的指针
    __u64    res;   // 错误码
    __u64    res2;  // 次级错误码
}

struct iocb
{
    __u64 aio_data; // 用户传入的字段，在event中返回，可以用来作为callback
    __u32 aio_key;  // flags
    __u16 aio_lio_opcode;  // io类型，读、写、fsync
    __s16 aio_reqprio;     // io优先级，没有作用？
    __u32 aio_fildes;      // 打开文件的fd
    __u64 aio_buf;         // io buf
    __u64 aio_nbytes;      // io length
    __s64 aio_offset;      // io offset
    __u64 aio_reserved2;
    __u32 aio_flags;
    __u32 aio_resfd;       // 用来配合eventfd实现event mode
}
```

#### Example

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <err.h>
#include <errno.h>

#include <unistd.h>
#include <fcntl.h>
#include <libaio.h>

int main() {
	io_context_t ctx;
	struct iocb iocb;
	struct iocb * iocbs[1];
	struct io_event events[1];
	struct timespec timeout;
	int fd;

	fd = open("/tmp/test", O_WRONLY | O_CREAT);
	if (fd < 0) err(1, "open");

	memset(&ctx, 0, sizeof(ctx));
	if (io_setup(10, &ctx) != 0) err(1, "io_setup");

	const char *msg = "hello";
	io_prep_pwrite(&iocb, fd, (void *)msg, strlen(msg), 0);
	iocb.data = (void *)msg;

	iocbs[0] = &iocb;

	if (io_submit(ctx, 1, iocbs) != 1) {
		io_destroy(ctx);
		err(1, "io_submit");
	}

	while (1) {
		timeout.tv_sec = 0;
		timeout.tv_nsec = 500000000;
		if (io_getevents(ctx, 0, 1, events, &timeout) == 1) {
			close(fd);
			break;
		}
		printf("not done yet\n");
		sleep(1);
	}
	io_destroy(ctx);

	return 0;
}
```
