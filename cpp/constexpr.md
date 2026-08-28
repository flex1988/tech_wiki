# C++11新特性 constexpr

constexpr是C++11引入的关键词，目的是为了引入更多的编译时计算能力，constexpr修饰的函数如果在编译器就能确定传入的参数，那么函数的调用就会替换为返回的值，如果编译器不能确定就和普通函数一样。

constexpr也可以用来修饰变量，和修饰函数不同的是，如果编译器不能确定值，就会报错。



比如这个例子

```
#include "stdio.h"

constexpr int sum(int a, int b)
{
    return a + b;
}

int main(int argc, char** argv)
{
    constexpr int sum1 = sum(1, 2);

    int sum2 = sum(*(int*)argv[1], *(int*)argv[2]);

    printf("sum1 %d sum2 %d\n", sum1, sum2);

    return 0;
}
```

<figure><img src="../.gitbook/assets/image (13).png" alt=""><figcaption></figcaption></figure>

sum1用constexpr修饰后，第一次调用编译器就能确定值，所以会被替换为值，生成的汇编代码中只调用了一次sum，而假如sum2也用constexpr修饰，就会发生一个编译错误，因为sum2的返回值在运行时才能确定。

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>
