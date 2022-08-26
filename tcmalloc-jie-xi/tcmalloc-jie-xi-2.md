# tcmalloc解析2

从malloc进来后，到了tc\_malloc然后就到了malloc\_fast\_path，先来分析下这个函数

```
template <void* OOMHandler(size_t)>
ATTRIBUTE_ALWAYS_INLINE inline
static void * malloc_fast_path(size_t size) {
  if (PREDICT_FALSE(!base::internal::new_hooks_.empty())) {
    return tcmalloc::dispatch_allocate_full<OOMHandler>(size);
  }

  ThreadCache *cache = ThreadCache::GetFastPathCache();

  if (PREDICT_FALSE(cache == NULL)) {
    return tcmalloc::dispatch_allocate_full<OOMHandler>(size);
  }

  uint32 cl;
  if (PREDICT_FALSE(!Static::sizemap()->GetSizeClass(size, &cl))) {
    return tcmalloc::dispatch_allocate_full<OOMHandler>(size);
  }

  size_t allocated_size = Static::sizemap()->ByteSizeForClass(cl);

  if (PREDICT_FALSE(!cache->TryRecordAllocationFast(allocated_size))) {
    return tcmalloc::dispatch_allocate_full<OOMHandler>(size);
  }

  return CheckedMallocResult(cache->Allocate(allocated_size, cl, OOMHandler));
}
```

首先是这个宏PREDICT\_FALSE，它基本上是对\_\_builtin\_expect的一个封装，由于现代的cpu有多级流水线，为了优化性能，cpu在执行本行的代码的时候已经在去取下面的代码了，这个时候假如接下来的代码有个if分支，那么cpu取哪个分支的代码对性能有这很大的影响，都去取的话肯定是不行的了，cache非常金贵，通常cpu会默认去取if后面跟着的分支，但是假如if分支是几乎不会执行的怎么办，这个宏的意义就来了，它告诉编译器接下来的分支很少会被执行，这样能减少程序的cache miss，提高性能。

第一个分支是判断new\_hooks\_是不是空，如果是不是空的话，就走到dispatch\_allocate\_full函数里，这里我们先研究一下这个new\_hooks\_是什么东西。

经过翻阅代码，我发现这个new\_hooks\_是tcmalloc添加的一些钩子函数，它可以让人在比如malloc或者free的时候做一些事，内部有提供一组接口去设置hook函数，我们来写个例子

```
#include <stdlib.h>
#include <stdio.h>
#include <gperftools/malloc_extension_c.h>
#include <gperftools/malloc_hook_c.h>

void hookMalloc(const void* p, size_t size)
{
   printf("malloc hook %p %ld\n", p, size);
}

void hookFree(const void* p)
{
   printf("free hook\n");
}

int main()
{
    char* p = (char*)malloc(4096);
    MallocHook_AddNewHook(&hookMalloc);
    MallocHook_AddDeleteHook(&hookFree);

    printf("ptr is %p\n", p);

    MallocHook_RemoveNewHook(&hookMalloc);
    MallocHook_RemoveDeleteHook(&hookFree);
    return 0;
}
```

编译运行可以看到我们设置的hook函数在malloc和free的时候被执行了

g++ hook.cpp ./.libs/libtcmalloc.a -g -O0 -lpthread -lunwind ![image](https://user-images.githubusercontent.com/7221964/126868569-ae884edf-9f89-4607-95c5-c815e72292dc.png)

这个分支是一个不太会走到的路径，可能是有特殊要求的时候才会开启Hook
