# tcmalloc解析1



tcmalloc是google开源的一款高性能malloc库，相对于jemalloc或者libc的malloc，在多线程下性能特别优异，是现在c++系统的基石，应用非常广泛。

一直想整体研究下它，但一直没找到很充足的时间，而且前面的一些学习也容易忘掉，看来还是要找个地方记录一下进度，这样下次就能接起来了

今天现在研究下他的编译和构成。

**1. 编译**

首先把它的代码从git上checkout出来编译

```
1. git clone git@github.com:gperftools/gperftools.git
2. git checkout gperftools-2.9
3. cd gperftools-2.9
4. sh autogen.sh
5. cmake .
```

这样就能直接在根目录编译出来UT和lib等

**2. debug tcmalloc**

剖析tcmalloc，那么最直观的方法是gdb单步调试看他的malloc和free的运行逻辑，所以首先需要能够debug编译出来的程序

1. 首先写一个最简单的malloc程序

```
#include "stdio.h"
#include "stdlib.h"
int main()
{
    char* p = (char*)malloc(4096);
    printf("p %p\n", p);
    return 0;
}
```

1. 开启静态链接 ./configure --enable-static make 生成的target在.libs目录
2. 用.libs下面的本地lib链接 g++ test.cpp .libs/libtcmalloc\_debug.a -lpthread
3. gdb调试程序，b test.cpp:5行，然后s就可以跳入到tcmalloc的代码里了 ![image](https://user-images.githubusercontent.com/7221964/126760834-f876a433-faa8-44c5-9062-ee0f389a3e09.png)
