# std::tie 和 std::tuple std::ignore

tuple 的概念是元组，可以将多种不同类型的元素组成一个结构，常用于函数的多返回值。

std::pair只能用于 2 个元素，std::tuple是对 std::pair的扩展。

```
#include <iostream>
#include <tuple>

struct A {
    int x;
    int y;
};

std::tuple<A, int, double> doSomeThing() {
    A a;
    a.x = 1;
    a.y = 2;
    int b = 1;
    double c = 0.8;
    return {a, b, c};
}

int main() {
    std::tuple<A, int, double> ret = doSomeThing();
    std::cout << std::get<0>(ret).x << std::endl;
    std::cout << std::get<0>(ret).y  << std::endl;
    std::cout << std::get<1>(ret) << std::endl;
    std::cout << std::get<2>(ret) << std::endl;

    return 0;
}
```

std::tie则可以创建 std::tuple的左值引用，从而避免内存拷贝的性能损失，也可以用于 tuple的解包。

```
#include <iostream>
#include <tuple>

struct A {
    int x;
    int y;
};

std::tuple<A, int, double> doSomeThing() {
    A a;
    a.x = 1;
    a.y = 2;
    int b = 1;
    double c = 0.8;
    return {a, b, c};
}

int main() {
    A a;
    int b;
    double c;
    std::tie(a, b, c) = doSomeThing();
    std::cout << a.x << std::endl;
    std::cout << a.y  << std::endl;
    std::cout << b << std::endl;
    std::cout << c << std::endl;
    return 0;
}
```

当有不关注的值时，可以用 std::ignore作为占位符

```
std::tie(std::ignore, b, c) = doSomeThing();
```

