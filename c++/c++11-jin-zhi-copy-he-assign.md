# C++11 禁止copy和assign

C++编译器可以为类生成默认的构造函数，copy构造函数，copy assign操作符和析构函数。但很多时候类只会有一个对象，所以禁掉编译器默认生成的copy构造函数可能会避免一些潜在的问题。

一般我们可以用一个宏来吧copy构造函数和copy assign操作符私有，比如：

```
DISALLOW_COPY_AND_ASSIGN(Type) \
private:                       \
    Type(const Type&);         \
    Type& operator=(const Type&);
    
class SomeClass
{
public:
    
DISALLOW_COPY_AND_ASSIGN(SomeClass);
};
```

而在C++11中可以显式的删掉这些构造函数

```
DISALLOW_COPY_AND_ASSIGN(Type) \
    Type(const Type&) = delete;         \
    Type& operator=(const Type&) = delete;
```
