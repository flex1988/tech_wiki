# C++11 std::move

#### 1. 左值，右值

左值是可以取到地址的变量，一般存在于系统的内存里，右值是无法取到地址的值，一般可能存在于寄存器当中，比如：

```
int a = 1;
```

a是左值，1是右值。

#### 2. 左值引用，右值引用

左值引用指向左值，右值引用指向右值，右值引用一般用来在移动构造函数函数或者移动赋值运算符等来高效的转义资源，但是const左值引用是可以指向右值的。

```
int a = 1;
int& a_ref = a;         // 左值引用指向左值
const int& a_ref1 = 1;  // const左值引用可以指向右值

int&& a_ref_r = 1;      // 右值引用指向右值
int&& a_ref_r1 = std::move(a); // std::move可以把左值转为右值引用
```

#### 3. 移动构造函数函数

事实上std::move什么都没有做，只是类型转换，真正的转义资源是在移动构造函数里做的。

当std::move把左值转为右值引用后，就可以调用对象的移动构造函数了。

std::move一般用在对象中有在堆上分配的资源赋值的场景，可以减少内存拷贝，比如：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <string>

class A
{
public:
    A()
    : res_(NULL)
    {
      res_ = new char[128];
    }

    // 拷贝构造，需要复制资源
    A(const A& a)
    {
      res_ = new char[128];
      memcpy(res_, a.res_, 128);
    }

    // 移动构造，直接转移资源
    A(A&& a)
    {
      res_ = a.res_;
      a.res_ = NULL;
    }

    ~A()
    {
    }

    void* Resource()
    {
      return res_;
    }

private:
    char* res_;
};

int main()
{
    A a = A();
    A b = a;
    printf("a %p b %p\n", a.Resource(), b.Resource());
    A c = std::move(a);
    printf("a %p b %p c %p\n", a.Resource(), b.Resource(), c.Resource());
    return 0;
}


a 0xf4a010 b 0xf4a0a0
a (nil) b 0xf4a0a0 c 0xf4a010
```

b是拷贝构造，所以b的资源和a不是一个

c通过std::move把值转为右值引用，在移动构造函数里直接把资源转移避免拷贝，提高了性能，所以导致a的资源最后为空，c的资源就是a的资源。
