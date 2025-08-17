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

虚函数表 vtable 是 c++ 实现多态的关键技术，在每个存在继承关系的类的对象中，最前面一个指针就是虚函数表，gdb 打印对象可以看到，vptr.Animal 指针指向的就是类的虚函数表。x类的虚函数在虚函数表中依次排列。

<figure><img src="../.gitbook/assets/image (40).png" alt=""><figcaption></figcaption></figure>

### 2.1 没有虚函数的类没有虚函数表

比如，下面的例子，gdb打印对象可以看到没有虚函数表

```
class Car {
public:
    void Speak() {
        std::cout << "bbbb" << std::endl;
    }
};

int main() {
    Car c;
    c.Speak();
    return 0;
}
```

<figure><img src="../.gitbook/assets/image (41).png" alt=""><figcaption></figcaption></figure>

### 2.2 虚函数表是一个函数指针的数组，对象的虚函数位于这个数组中

```
#include <iostream>

class Animal {
public:
    virtual void Speak() = 0;
    virtual void Run() = 0;
};

class Human : public Animal {
public:
    void Speak() override {
        std::cout << "hello" << std::endl;
    }

    void Run() override {
        std::cout << "human run" << std::endl;
    }
};

class Dog : public Animal {
    void Speak() override {
        std::cout << "wangwangwang" << std::endl;
    }

    void Run() override {
        std::cout << "dog run" << std::endl;
    }
};

class Sheep : public Animal {
    void Speak() override {
        std::cout << "miemiemie" << std::endl;
    }

    void Run() override {
        std::cout << "sheep run" << std::endl;
    }
};

int main() {
    Animal* h = new Human;
    Animal* d = new Dog;
    Animal* s = new Sheep;

    printf("Human::Speak: %p Human::Run %p\n", (void*)&Human::Speak, (void*)&Human::Run);

    return 0;
}
```

gdb 可以看到 Human类实现的虚函数 Speak 0x40127e 和 Run 0x4012aa 位于 vtable 中

<figure><img src="../.gitbook/assets/image (42).png" alt=""><figcaption></figcaption></figure>

### 2.3 虚函数表中只有虚函数

```
class Animal {
public:
    virtual void Speak() = 0;
    virtual void Run() = 0;
};

class Human : public Animal {
public:
    void Speak() override {
        std::cout << "hello" << std::endl;
    }

    void Speak1() {
    }

    void Run() {
        std::cout << "human run" << std::endl;
    }
};

int main() {
    Animal* h = new Human;

    printf("Human::Speak: %p Hunam::Speak1 %p Human::Run %p\n", (void*)&Human::Speak, (void*)&Human::Speak1, (void*)&Human::Run);

    return 0;
}
```

gdb可以看到 Human 虚函数表中只有重写的基类虚函数 Speak 和 Run，而函数 Speak1 由于不是虚函数所以不在虚函数表中

![](<../.gitbook/assets/image (43).png>)\


### 2.4 如果两个对象是同一个子类，那么虚函数表的指针相同

不同的子类虚函数表不同，但同一个子类，每个对象的虚函数表都是一个

```
Human* h = new Human;
Human* h2 = new Human;
```

![](<../.gitbook/assets/image (45).png>)\
