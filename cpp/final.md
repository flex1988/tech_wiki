# C++11 新特性 final

C++11中允许将类标记为final，表示类不能被继承，也允许将一个虚函数标记为final，表示函数不能在子类中重写

```
struct Base
{
    virtual void foo();
};
 
struct A : Base
{
    void foo() final; // Base::foo is overridden and A::foo is the final override
    void bar() final; // Error: bar cannot be final as it is non-virtual
};
 
struct B final : A // struct B is final
{
    void foo() override; // Error: foo cannot be overridden as it is final in A
};
 
struct C : B {}; // Error: B is final


main.cpp:9:10: error: 'void A::bar()' marked 'final', but is not virtual
    9 |     void bar() final; // Error: bar cannot be final as it is non-virtual
      |          ^~~
main.cpp:14:10: error: virtual function 'virtual void B::foo()' overriding final function
   14 |     void foo() override; // Error: foo cannot be overridden as it is final in A
      |          ^~~
main.cpp:8:10: note: overridden function is 'virtual void A::foo()'
    8 |     void foo() final; // Base::foo is overridden and A::foo is the final override
      |          ^~~
main.cpp:17:8: error: cannot derive from 'final' base 'B' in derived type 'C'
   17 | struct C : B // Error: B is final
```
