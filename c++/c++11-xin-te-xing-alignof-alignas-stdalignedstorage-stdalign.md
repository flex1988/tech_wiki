# C++11 新特性alignof alignas std::aligned\_storage std::align

在现代计算机系统中，CPU通过总线读取设备的数据一般是8B对齐读的，如果一个对象的指针不按照8B对齐，那么肯定效率不高，另外CPU Cache的CacheLine也是64B或者128B，所以内存对齐在高性能C++编程中是一个很重要的问题。

C++11引入了几个新概念，alignof alignas std::aligned\_storage std::align，其中alignof 和alignas是关键词，std::aligned\_storage是类，std::align是函数。

### alignment

在C++程序里的对象，结构体在编译的时候都会按照alignment来内存对齐，一般基础类型会按照sizeof的大小来对齐，对齐的大小也就是alignment。

比如一个变量int a，取a的地址ptr，ptr的地址一定满足ptr % 4 == 0

可以在编译期检查内存对齐的宏：

```
#define CHECK_ALIGN(ptr, alignment)                       \
  do{                                                     \
    constexpr size_t status                               \
       = reinterpret_cast<uintptr_t>(ptr) % alignment;    \
    static_assert(status == 0, "ptr must be aligned");    \
  }while(0)                                               \
```

### alignof

alignof是C++11引入的关键词，可以得到类型的内存对齐要求，用法类似于sizeof

```
#include <iostream>

int main()
{
    char a;
    int b;
    long c;

    std::cout << "alignof(char) " << alignof(a) << " alignof(int) "
            << alignof(b) << " alignof(long) " << alignof(c) << std::endl;

    return 0;
}

g++ x.cpp -std=c++11

alignof(char) 1 alignof(int) 4 alignof(long) 8
```

对于复杂类型来说，情况又有些不同，可以看到alignof返回的是最大的属性的alignment，而sizeof保证了所有字段在内存对齐时的长度，下面的例子把两个char的字段放到了4个字节里，为了和下面的int对齐。

```
#include <iostream>

int main()
{
    struct s
    {
        char x1;
        char x2;
        int x3;
    };

    std::cout << "sizeof(s) " << sizeof(s) << " alignof(s) " << alignof(s) << std::endl;

    return 0;
}

sizeof(s) 8 alignof(s) 4
```

有时候我们在存储领域工作时，需要把内存的结构和磁盘上的数据完全一样，而内存对齐会浪费磁盘空间，这时候我们可以使用#pragma pack(1)来避免自动对齐

```
#include <iostream>

int main()
{
#pragma pack(1)
    struct s
    {
        char x1;
        char x2;
        int x3;
    };

    std::cout << "sizeof(s) " << sizeof(s) << " alignof(s) " << alignof(s) << std::endl;

#pragma pack()

    return 0;
}

sizeof(s) 6 alignof(s) 1
```

### alignas

alignas是C++引入的新关键词，作用是改变一个类的alignment，比如从CacheLine或者SIMD操作等角度考虑，更改类的alignment会更有效率，在C++11之前可以用gcc提供的\_\_attribute\_\_(\_\_aligned\_\_(alignment))来实现。

```
sizeof(s) 8 alignof(s) 4 sizeof(s1) 16 alignof(s1) 16
```

另外用alignas来修改alignment时有一些限制，首先不能小于原生的对齐需求，其次alignment必须是2的整数次幂。

### std::aligned\_storage

std::aligned\_storage是C++11中新引入的一个用于满足内存对齐要求的静态内存分配类，位于type\_traits

```
template<std::size_t len, std::size_t align>
struct aligned_storage
{
};
```

当类std::aligned\_storage的对象构造完成时，就分配了长度为len，而且内存地址满足algin的对齐要求。

### std::align

std::align是C++11中引入的一个方法，可以用来从一段内存中出去满足alignment对齐需求的内存。

```
/// alignment 是想要分配的内存符合的内存对齐大小
/// size 想要分配内存的大小
/// ptr 是个输入输出参数，输入时指向待使用的内存，输出时调整为符合alignment对齐要求的内存地址
/// space 是ptr指向的内存剩余的空间
/// 如果 ptr 经过调整后能满足大小为 alignment 的对齐要求，则返回ptr的值，否则返回 nullptr
void* align(std::size_t alignment,
             std::size_t size,
             void*& ptr,
             std::size_t& space);
```
