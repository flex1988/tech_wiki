# c++ 多态和虚函数表

## 1 多态

c++  中，多态是面向对象编程的核心特性之一，c++ 允许使用基类的指针调用子类重写的方法，从而实现同一个接口有不同的行为。

在基类中定义函数 virtual void Speak();，virtual 表示为虚函数，可以被继承的子类函数重写。

在子类中定义函数 void Speak() override;，表示重写了基类的 Speak 函数。

比如下面这个例子

```cpp
#include <set>
#include <iostream>

class Animal {
public:
    virtual void Speak() = 0;
};

class Human : public Animal {
    void Speak() override {
        std::cout << "hello" << std::endl;
    }
};

class Dog : public Animal {
    void Speak() override {
        std::cout << "wangwangwang" << std::endl;
    }
};

class Sheep : public Animal {
    void Speak() override {
        std::cout << "miemiemie" << std::endl;
    }
};

int main() {
    std::set<Animal*> animals;
    animals.insert(new Human);
    animals.insert(new Dog);
    animals.insert(new Sheep);
    
    for (auto& animal : animals) {
        animal->Speak();
    }
    return 0;
}


```

## 2 虚函数表

虚函数表 vtable 是 c++ 实现多态的关键技术，在每个存在继承关系的类的对象中，最前面一个指针就是虚函数表，gdb p 对象可以看到，vptr.Animal = 一个指针，指向的就是类的虚函数表。

\
\
