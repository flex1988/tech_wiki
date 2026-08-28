# 如何实现一个FIO插件

FIO是一个开源的压力测试工具，主要用来测试磁盘的io性能。它支持13种不同类型的io引擎（libaio sync mmap libiscsi nbd nfs等等），它支持用户自定义各种不同的io pattern。

当我们测试自己开发的IO路径时，也可以添加一个自定义的ioengine。

#### ioengine接口

fio ioengine的接口在ioengines.h，要实现一个自定义的async engine，就需要实现这一套接口，然后fio的框架会调用engine的queue接口把io放进来，ioengine负责执行IO，然后ioengine会通过getevents接口来获取events，得到io执行结果。

```
struct ioengine_ops {
        struct flist_head list;
        const char *name;
        int version;
        int flags;
        void *dlhandle;
        int (*setup)(struct thread_data *);
        int (*init)(struct thread_data *);
        int (*post_init)(struct thread_data *);
        int (*prep)(struct thread_data *, struct io_u *);
        enum fio_q_status (*queue)(struct thread_data *, struct io_u *);
        int (*commit)(struct thread_data *);
        int (*getevents)(struct thread_data *, unsigned int, unsigned int, const struct timespec *);
        struct io_u *(*event)(struct thread_data *, int);
        char *(*errdetails)(struct io_u *);
        int (*cancel)(struct thread_data *, struct io_u *);
        void (*cleanup)(struct thread_data *);
        int (*open_file)(struct thread_data *, struct fio_file *);
        int (*close_file)(struct thread_data *, struct fio_file *);
        int (*invalidate)(struct thread_data *, struct fio_file *);
        int (*unlink_file)(struct thread_data *, struct fio_file *);
        int (*get_file_size)(struct thread_data *, struct fio_file *);
        int (*prepopulate_file)(struct thread_data *, struct fio_file *);
        void (*terminate)(struct thread_data *);
        int (*iomem_alloc)(struct thread_data *, size_t);
        void (*iomem_free)(struct thread_data *);
        int (*io_u_init)(struct thread_data *, struct io_u *);
        void (*io_u_free)(struct thread_data *, struct io_u *);
        int (*get_zoned_model)(struct thread_data *td,
                               struct fio_file *f, enum zbd_zoned_model *);
        int (*report_zones)(struct thread_data *, struct fio_file *,
                            uint64_t, struct zbd_zone *, unsigned int);
        int (*reset_wp)(struct thread_data *, struct fio_file *,
                        uint64_t, uint64_t);
        int (*get_max_open_zones)(struct thread_data *, struct fio_file *,
                                  unsigned int *);
        int (*get_max_active_zones)(struct thread_data *, struct fio_file *,
                                    unsigned int *);
        int (*finish_zone)(struct thread_data *, struct fio_file *,
                           uint64_t, uint64_t);
        int (*fdp_fetch_ruhs)(struct thread_data *, struct fio_file *,
                              struct fio_ruhs_info *);
        int option_struct_size;
        struct fio_option *options;
};
```

#### 实现ioengine接口

#### 1 每个线程都有一个thread data

<pre><code><strong>struct libfs_data {
</strong>        struct io_u **io_us;
        struct io_u **io_u_index;
        struct libfs_io_result* results;
        unsigned int entries;
        unsigned int queued;
        unsigned int head;
        unsigned int tail;
};
</code></pre>

#### 2 定义一组ioengine\_ops

```
static struct ioengine_ops ioengine = {
        .name           = "libengine",
        .version        = FIO_IOOPS_VERSION,
        .flags          = FIO_ASYNCIO_SYNC_TRIM | FIO_MEMALIGN | FIO_ASYNCIO_SETS_ISSUE_TIME,
        .init           = fio_libengine_init,
        .cleanup        = fio_libengine_cleanup,
        .open_file      = libengine_open_file,
        .close_file     = libengine_close_file,
        .get_file_size  = libengine_get_file_size,
        .setup          = libengine_setup,
        .queue          = fio_libengine_queue,
        .commit         = fio_libengine_commit,
        .getevents      = fio_libengine_getevents,
        .event          = fio_libengine_event,
        .invalidate     = libengine_invalidate,
        .option_struct_size = sizeof(struct libengine_options),
};

static void fio_init fio_libengine_register()
{
        register_ioengine(&ioengine);
}

static void fio_exit fio_libengine_unregister()
{
        unregister_ioengine(&ioengine);
}
```

#### 3 IO入队列函数

```
void fio_libengine_queued(struct thread_data *td, struct io_u **io_us,
                              unsigned int nr)
```

#### 4 提交队列中的请求

```
int fio_libengine_commit(struct thread_data *td)
```

5 POLL获取事件

```
int fio_libengine_getevents(struct thread_data *td, unsigned int min,
                                unsigned int max, const struct timespec *t)
```

#### 6 解析每一个事件，释放IO\_U

```
struct io_u *fio_libengine_event(struct thread_data *td, int event)
```

7 最后修改Makefile，把自定义engine添加进去

```
libengine_SRCS = engines/libengine.c
libengine_LIBS = -lengine
ENGINES += libengine
```
