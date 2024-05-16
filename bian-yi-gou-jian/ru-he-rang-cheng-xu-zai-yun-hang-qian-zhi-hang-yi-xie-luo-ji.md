# 如何让程序在运行前执行一些逻辑

最近在使用 fio 的时候，发现 ioengine 的加载是在\_\_libc\_csu\_init函数中的，而函数\_\_libc\_csu\_init是 libc在运行程序的 main 函数前，用来调用初始化的一组函数。

和\_\_libc\_csu\_init配套的还有一个\_\_libc\_csu\_fini，在 main 函数退出后运行。

那么 fio 是如何把 ioengine\_register 函数放到\_\_libc\_csu\_init里的呢？

```
static void fio_init fio_nvmed_register(void)
{
    register_ioengine(&ioengine);
}

static void fio_exit fio_nvmed_unregister(void)
{
    register_ioengine(&ioengine);
}
```

观察模块加载函数有一个 fio\_init和 fio\_exit比较特别，在代码中找到定义，是一个编译器的属性。

```
#define fio_init        __attribute__((constructor))
#define fio_exit        __attribute__((destructor))
```

他们的作用是把函数标记为需要被\_\_libc\_csu\_init 调用的函数类型，objdump -d 可以看到函数在 section text.startup

![](../.gitbook/assets/image.png)

