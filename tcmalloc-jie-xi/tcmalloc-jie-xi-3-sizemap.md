# tcmalloc解析3 sizemap



接下来就是ThreadCache \*cache = ThreadCache::GetFastPathCache()，这是在从tls中，取一个ThreadCache的对象，看起来这是一个很重要的数据结构，如果没有初始化的话还是会到dispatch\_allocate\_full函数。

在然后是Static::sizemap()->GetSizeClass(size, \&cl)，看下要申请的内存根据大小是哪种object，如果没有查到就到dispatch\_allocate\_full函数，SizeClass也是个很重要的结构，下面在分析。

然后是cache->TryRecordAllocationFast(allocated\_size)，这个函数有点意思，先研究一下。

首先这个函数其实是在Sampler类里，Sampler实现了一个很高效的取样机制，它包含一个有点复杂的采样逻辑，在判断某次内存申请需要采样时，就会把当前线程的栈保存下来记录到一个地方，判断是否要采样的机制大概就是每次内存申请都会将bytes\_until\_sample\_变量减掉申请的内存大小，当bytes\_until\_sample\_小于0时，当前请求会被采样。每次采样后bytes\_until\_sample\_也会变化，它的数字也是geometrically distributed。4k的内存申请被采样的几率是 0.00778，而1MB的采样几率是0.865，具体逻辑如下：

```
// With 512K average sample step (the default):
//  the probability of sampling a 4K allocation is about 0.00778
//  the probability of sampling a 1MB allocation is about 0.865
//  the probability of sampling a 1GB allocation is about 1.00000
// In general, the probablity of sampling is an allocation of size X
// given a flag value of Y (default 1M) is:
//  1 - e^(-X/Y)
//
// With 128K average sample step:
//  the probability of sampling a 1MB allocation is about 0.99966
//  the probability of sampling a 1GB allocation is about 1.0
//  (about 1 - 2**(-26))
// With 1M average sample step:
//  the probability of sampling a 4K allocation is about 0.00390
//  the probability of sampling a 1MB allocation is about 0.632
//  the probability of sampling a 1GB allocation is about 1.0
```

采样逻辑是也是个辅助功能，应该也是很少才会打开的。

比较有意思的是这个函数本身，在将bytes\_until\_sample\_减掉当前申请的k后，判断是否bytes\_until\_sample\_小于0时，利用到了一个cpu的指令优化功能。sub , followed by conditional jump on 'carry'，不过这个机制暂时还没有理解，先放一放。

```
inline bool Sampler::TryRecordAllocationFast(size_t k) {
  // For efficiency reason, we're testing bytes_until_sample_ after
  // decrementing it by k. This allows compiler to do sub <reg>, <mem>
  // followed by conditional jump on sign. But it is correct only if k
  // is actually smaller than largest ssize_t value. Otherwise
  // converting k to signed value overflows.
  //
  // It would be great for generated code to be sub <reg>, <mem>
  // followed by conditional jump on 'carry', which would work for
  // arbitrary values of k, but there seem to be no way to express
  // that in C++.
  //
  // Our API contract explicitly states that only small values of k
  // are permitted. And thus it makes sense to assert on that.
  ASSERT(static_cast<ssize_t>(k) >= 0);

  bytes_until_sample_ -= static_cast<ssize_t>(k);
  if (PREDICT_FALSE(bytes_until_sample_ < 0)) {
    // Note, we undo sampling counter update, since we're not actually
    // handling slow path in the "needs sampling" case (calling
    // RecordAllocationSlow to reset counter). And we do that in order
    // to avoid non-tail calls in malloc fast-path. See also comments
    // on declaration inside Sampler class.
    //
    // volatile is used here to improve compiler's choice of
    // instuctions. We know that this path is very rare and that there
    // is no need to keep previous value of bytes_until_sample_ in
    // register. This helps compiler generate slightly more efficient
    // sub <reg>, <mem> instruction for subtraction above.
    volatile ssize_t *ptr =
        const_cast<volatile ssize_t *>(&bytes_until_sample_);
    *ptr += k;
    return false;
  }
  return true;
}
```
