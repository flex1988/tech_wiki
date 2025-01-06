# tcmalloc google heap profiler调查内存泄露

{% embed url="https://gperftools.github.io/gperftools/heapprofile.html" %}

用tcmalloc的heap profiler可以调查内存泄露问题，比如下面这段代码，每次会随机泄露一小块内存

<pre class="language-cpp"><code class="lang-cpp">#include &#x3C;map>
#include &#x3C;stdlib.h>

void leak_func()
{
    std::map&#x3C;int, char*> memorys;
<strong>    
</strong>    while (1)
    {
        char* m = new char[rand() % (1 &#x3C;&#x3C; 20)];
        memorys[rand()] = m;
    }
}

int main()
{
    leak_func();
    return 0;
}
</code></pre>

编译的时候不是必须链接tcmalloc，运行时用LD\_PRELOAD指定tcmalloc path也可以，编译好后指定LD\_PRELOAD和HEAPPROFILE运行程序，HEAPPROFILE指定了memory heap文件dump的地址

```
 LD_PRELOAD=/usr/lib64/libtcmalloc.so HEAPPROFILE=/data/ ./a.out
```

![](<../.gitbook/assets/image (1) (1) (1) (1) (1).png>)

很快就dump出来很多heap文件，可以直接分析某个文件的内存分配\
![](<../.gitbook/assets/image (2) (1) (1) (1).png>)

也可以比较两个heap之间的diff，比如某一段时间内存突然涨上去了就可以用两个heap文件来分析

![](<../.gitbook/assets/image (3) (1).png>)

第一列显示了使用的物理内存MB，第四列显示了函数本身和它的调用函数总共申请的内存，第二列和第五列是第一列和第四列的百分比，第三列是第二列的累积和

就我们这个简单的例子来看，很清晰的就能看到在leak\_func里泄露了5819.9MB内存
