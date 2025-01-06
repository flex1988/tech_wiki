# xbyak

仓库地址 git@github.com:herumi/xbyak.git

_&#x58;_&#x62;yak是一个x86(IA-32),x64(x86\_64,AMD64)下的C++ JIT _assembler。_

_&#x58;_&#x62;yak is a C++ header library that enables dynamically to assemble x86(IA32), x64(AMD64, x86-64) mnemonic.

```cpp
#include "xbyak/xbyak.h"
struct Code : Xbyak::CodeGenerator 
{ 
    Code(int x) 
    { 
        mov(eax, x); ret(); 
    } 
};

int main() 
{ 
    Code c(5);
    int (*f)() = c.getCode<int(*)()>();
    printf("ret = %d\n", f());
}
```

```
[root@centos ~]# g++ -I xbyak/ xbyak.cpp
[root@centos ~]# ./a.out
ret = 5
```

