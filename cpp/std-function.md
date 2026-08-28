# C++ std::function

std::function 是c++11 加入的一个模版类，它是一个通用的多态的函数包装器。

它可以持有，复制和调用任何符合其签名的可调用对象（callable object），定义在\<functional>头文件中。

它的核心作用是类型擦除（Type Erasure）。它将各种不同类型的可调用对象（它们都有自己独特的，互不兼容的类型）擦除掉，包装成一个统一的 std::function 类型

### 1 为什么需要 std::function

在 C/C++ 的底层，可以被调用的只有函数，在 x86\_64 汇编语言中由 callq 和 retq 来实现。但是在现代的编程语言中出现了 closure 的概念，closure 就是由函数和函数的参数绑定形成的，比如lambda 表达式，类的成员函数，std::bind等都是 closure 的实现形式。所以可调用的对象除了函数外，还有其他的类型。

1. 普通函数
2. lambda 表达式
3. 函数对象，重载了 operator() 类的实例
4. 类的成员函数
5. std::bind 返回的结果

这带来的问题是，它们的类型各不相同。比如函数指针 void(\*)(int) 和 函数指针 int(\*)(int) 类型不同。每个 lambda 表达式也有自己独特的，匿名类型。

这就导致了一个问题：如果你想写一个函数，它接受一个回调函数作为参数，你应该用什么类型来接受呢？

```
// 如果只接受普通函数指针，就无法传入 Lambda 或 Functor
void process(int value, void(*callback)(int)) {
    // ... do something ...
    callback(value);
}

auto my_lambda = [](int x) { /* ... */ };
// process(10, my_lambda); // 编译错误！Lambda 类型和函数指针类型不匹配
```

而 std::function 完美的解决了这个问题，它只关心签名（即返回值和参数类型），不关系具体的可调用对象类型。

### 2 如何使用 std::function

#### 2.1 语法

std::function 是一个类模板，模板参数是函数的签名

```cpp
#include <functional>

// 声明一个 std::function ，它可以持有任何“返回 void 接受一个 int” 的可调用对象
std::function<void(int)> my_func;

// 声明一个 std::function，它可以持有任何 “返回值为 bool， 接受 2 个 const std::string&”的可调用对象
std::function<bool(const std::string&, const std::string&)> string_comparer;

// 声明一个 std::function，它可以持有任何 “无返回值，无参数的”可调用对象
std::function<void()> task;
```

#### 2.2 例子

```cpp
#include <functional>
#include <iostream>

// 1 普通函数
void print_number(int n) {
    std::cout << "func: " << n << std::endl;
}

struct NumberPrinter {
    void operator()(int n) const {
        std::cout << "func object: " << n << std::endl;
    }
};

struct MyClass {
    void print_number(int n) {
        std::cout << "member func: " << n << std::endl;
    }
    int member_data_ = 42;
};

int main() {
    std::function<void(int)> func_wrapper;
    func_wrapper = print_number;
    func_wrapper(10);

    func_wrapper = [](int n) { std::cout << "lambda: " << n << std::endl; };
    func_wrapper(20);

    NumberPrinter printer_obj;
    func_wrapper = printer_obj;
    func_wrapper(10);

    MyClass my;
    func_wrapper = std::bind(&MyClass::print_number, &my, std::placeholders::_1);
    func_wrapper(10);

    func_wrapper = [&](int n) { my.print_number(n); };

    func_wrapper(10);

    return 0;
}

```

### 3 std::function 实现机制

std::function 机制来自于类型擦除和小对象优化（SBO）

* 类型擦除，std::function 内部会存储两个东西
  * 一个指向实际可调用对象副本的指针
  * 一个指向管理器的函数指针，这个管理器知道如何对存储的对象进行调用，复制，销毁等操作，当你把一个lambda赋值给 std::function 时，他会根据lambda的类型实例化一个对应的管理器，这样无论你存入什么，std::function 外部的接口都是统一的。
* 小对象优化（SBO），为了避免每次都进行堆内存分配，std::function 内部通常有一个小的缓冲区
  * 如果存入的可调用对象足够小，可以直接放在这个缓冲区内，避免了堆分配，性能很好
  * 如果存入的对象太大，std::function 就会在堆上分配内存来存储它，这时就会有一次堆分配的开销

#### 3.1 性能开销

与直接调用函数或者函数指针相比，std::function 有额外的性能开销

* 一次内部指针间接调用
* 可能存在的堆内存分配
* 更大的体积（2～3 个指针大小）

#### 3.2 所有权

std::function 会复制并拥有它所持有的可调用对象，如果你的lambda通过值捕获了一个大对象，那么在赋值到std::function时就会复制这个大对象
