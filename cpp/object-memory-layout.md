# C++ 类对象的内存布局

c++ 类里面有成员函数，成员变量，还分为动态类型和静态类型等，那么 c++ 类 new 出来的对象内存布局是什么样的呢。

首先 c++ 标准没有硬性规定对象的内存布局，这通常由编译器和 ABI (Application Binary Interface) 来决定。在主流编译器上（gcc Clang MSVC），都遵循着一些共同的事实上的标准。

## 1 只有非静态数据成员的类

对象的内存布局就是其非静态数据成员按照声明顺序依次排列。

关键点：内存对齐

为了让 CPU 更高效地访问数据，编译器会对成员进行内存对齐。每个数据类型都有一个对齐的要求。编译器会在成员之间插入一些空白字符，以确保每个成员的其实地址都是其对齐要求的整数倍。

```
class MyClass {
public:
    char a;
    int b;
    char c;
};

int main() {
    MyClass* c = new MyClass;
    c->a = 1;
    c->b = 2;
    c->c = 3;
    return 0;
}
```

实际布局

```
// 地址偏移
// +0   +1   +2   +3   +4   +5   +6   +7   +8   +9   +10  +11
// | a | pad| pad| pad|      b      | c | pad| pad| pad |
// +-------------------+-------------------+--------------------+
//   ^                   ^                   ^
//   a (1 字节)        b (4 字节)          c (1 字节)
```

1. `a` (char) 放在偏移量 0 处。
2. `b` (int) 需要 4 字节对齐，所以它的起始地址必须是 4 的倍数。编译器在 `a` 后面填充 3 个字节（padding），使得 `b` 从偏移量 4 开始。
3. `c` (char) 放在 `b` 之后，即偏移量 8 处。
4. 为了让整个 `MyClass` 对象在数组中也能正确对齐，对象自身的总大小也需要是其最严格成员（这里是 `int`，对齐要求为 4）的整数倍。所以，在 `c` 后面再填充 3 个字节，使得总大小为 12 字节。

因此 MyClass 一般是 12 个字节

<figure><img src="../.gitbook/assets/image (46).png" alt=""><figcaption></figcaption></figure>

## 2. 包含成员函数的类

* 非静态成员函数，存储在代码段，由类所有的对象共享。当你调用一个成员函数时，编译器会隐式地传递一个指向该对象的指针，这个指针就是 this
* 静态成员函数，同样存储在代码段。他不与任何对象关联，所以没有 this 指针。

## 3. 包含静态数据成员的类

```
class StaticDataClass {
public:
    int a;
    static int b = 1;
};
```

静态数据成员不占用对象本身的内存空间。

## 4. virtual 关键词

当类包含虚函数时，布局开始变得复杂起来。为了实现多态，编译器引入了虚函数表（vtable）和虚函数表指针（vptr）

* 虚函数表，每个包含虚函数的类都有一个静态的，唯一的虚函数表，它是一个函数指针数组，存储了该类所有虚函数的地址
* 虚函数表指针，每个对象实例都会增加一个隐藏的成员，即 vptr，这个指针指向其所属类的 vtable。vptr 的大小通常是一个指针的大小

```
class Base {
public:
    virtual void func1() {}
    virtual void func2() {}
    int data_base;
};
```

```
// | vptr (8 bytes) | data_base (4 bytes) | padding (4 bytes) |
// +----------------+---------------------+-------------------+
//        |
//        +--> Base's vtable
//             +---------------------+
//             | &Base::func1        |  (offset 0)
//             +---------------------+
//             | &Base::func2        |  (offset 8)
//             +---------------------+
```

* 对象开头是一个 8 字节的 vptr
* vptr 指向 Base 类的 vtable&#x20;
* sizeof(Base) 的大小就是 8 + 4 + 4 = 16字节

## 5 继承下的布局

### 单继承

```
class Derived : public Base {
public:
    void func1() override {} // 重写虚函数
    virtual void func3() {}    // 新增虚函数
    int data_derived;
};
```

```
// | vptr (8 bytes) | data_base (4 bytes) | padding_base (4 bytes)| data_derived (4 bytes) | padding_derived(4 bytes) |
// +----------------+---------------------+-----------------------+------------------------+--------------------------+
//        |
//        +--> Derived's vtable
//             +-----------------------+
//             | &Derived::func1       | // 地址被重写
//             +-----------------------+
//             | &Base::func2          | // 地址继承自 Base
//             +-----------------------+
//             | &Derived::func3       | // 新增虚函数的地址
//             +-----------------------+
```

* derived 对象重用并扩展了 Base 的 vtable
* vptr现在指向了 derived 类的 vtable
* sizeof(derived) = 8(vtpr) + 4(data\_base) + 4(padding) + 4(data\_derived) + 4(padding) = 24字节

### 多重继承

多重继承时，派生类对象会包含多个基类子对象，他们的布局顺序通常与继承声明的顺序一致，如果多个基类都有虚函数，可能会有多个 vptr

```
class B1 { public: virtual void f1(); int b1_data; };
class B2 { public: virtual void f2(); int b2_data; };
class D : public B1, public B2 { public: int d_data; };
```

D 对象的布局

```
// | vptr_B1 | b1_data | pad | vptr_B2 | b2_data | pad | d_data | pad |
// +---------+---------+-----+---------+---------+-----+--------+-----+
//   |                         |
//   +--> D's vtable for B1    +--> D's vtable for B2
```

* 对象内包含了 B1子对象和 B2子对象
* 可能有两个 vptr，分别指向针对B1 和 B2的 vtable部分
* 当一个 D\*指针被转换为B2\*时，指针的值需要被调整（增加一个偏移量），以指向 B2 子对象的起始位置。这被称为 pointer adjustment 或者 thunking

## 6 虚继承

虚继承用来解决多重继承中的菱形问题，确保最终派生类中只有一个共同基类的实例。

实现方式通常是引入虚基类表指针（vbptr）。vbptr 指向一个表，该表描述了虚基类子对象相对于 vbptr 自身的偏移量

## 7 空基类优化 （Empty Base Optimization EBO）

如果一个基类是空的，编译器通常会进行优化，使得基类子对象不会占用任何空间。

```
class Empty {
};

class NonEmpty : public Empty {
    int data;
};
```

sizeof(Empty) 是 1

由于 EBO，sizeof(NonEmpty) 通常等于sizeof(int)，而不是 1 + sizeof(int)

空基类 Empty 的大小被吸收了
