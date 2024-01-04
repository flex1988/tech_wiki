# C++新特性 noexcept

### 1 关键词noexcept

noexcept是C++11里的新关键词，比如在leveldb的源码中Status类中有出现。

```
class LEVELDB_EXPORT Status {
 public:
  // Create a success status.
  Status() noexcept : state_(nullptr) {}
  ~Status() { delete[] state_; }
  ...
}
```

它的作用是告诉编译器，该函数不会抛出异常，可以引导编译器做更多的优化。

如果在运行时，noexcept函数抛出了异常，程序会直接终止，调用std::terminate函数

### 2 C++的异常处理

C++中的异常处理是在运行时而不是编译时检测的。为了实现运行时检测，编译器会创建额外的代码，这会带来性能损失。

在实践中，一般会用两种异常抛出方式：

1. 一个函数可能会抛出异常
2. 一个函数不会抛出异常

后面的方式可以用一些关键词标示，这样编译器可以做相应的优化，在以往的C++版本中用throw表示，在C++11中被noexcept代替，不会抛出异常用throw()，可能会抛出异常用throw(...)

```
void swap(Type& x, Type& y) throw()  // C++11之前
{
    x.swap(y);
}

void swap(Type& x, Type& y) noexcept // C++11
{
    x.swap(y);
}
```

