# c++ 多态和虚函数表

## 1 多态

c++  中，多态是面向对象编程的核心特性之一，c++ 允许使用基类的指针调用子类重写的方法，从而实现同一个接口有不同的行为。

比如下面这个例子

```cpp
class Animal {
    virtual void Speak() = 0;
}

class Human : public Animal {
    void Speak() override {
        std::cout << "hello" << std::endl;
    }
}

class Dog : public Animal {
    void Speak() override {
        std::cout << "wangwangwang" << std::endl;
    }
}

class Sheep : public Animal {
    void Speak() override {
        std::cout << "miemiemie" << std::endl;
    }
}

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

