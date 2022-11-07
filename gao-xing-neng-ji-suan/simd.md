# SIMD

SIMD全称Single Instruction Multiple Data（单指令流多数据流），是一种采用一个控制器来控制多个处理器，同时对一组数据中的每一个分别执行相同的操作从而实现空间上的并行性的技术。

在微处理器中，单指令刘多数据流技术则是一个控制器控制多个平行的处理微元，例如Intel的MMX或SSE，以及AMD的3D Now!指令集。

图形处理器（GPU）拥有强大的并发处理能力和可编程流水线，面对单指令流多数据流时，运算能力远超CPU吗。OpenCL和CUDA分别是目前最广泛使用的开源和专利通用图形处理器（GPGPU）运算语言。

目前，绝大多数的CPU都支持SIMD，不同的CPU架构和厂商提供了不同的SIMD指令集来支持，以常用的x86架构来说，我们可以通过SSE指令集来使用x86架构下的SIMD能力。



#### Intel SIMD

#### ![](<../.gitbook/assets/image (1).png>)

1997 年，Intel 推出了第一个 SIMD 指令集 —— MultiMedia eXtensions（MMX）。MMX 指令主要使用的寄存器为 MM0 \~ MM7，大小为 64 位。

1999 年，Intel 在 Pentium III 对 SIMD 做了扩展，名为 Streaming SIMD eXtensions（SSE）。SSE 采用了独立的寄存器组 XMM0 \~ XMM7，64位模式下为 XMM0 \~ XMM15 ，并且这些寄存器的长度也增加到了 128 位。

2000 年，Intel 从 Pentium 4 开始引入 SSE2。

2004年，Intel 在 Pentium 4 Prescott 将 SIMD 指令集扩展到了 SSE3。

2006 年，Intel 发布 SSE4 指令集，并在 2007 年推出的 CPU 上实现。

2008 年，Intel 和 AMD 提出了 Advanced Vector eXtentions（AVX）。并于 2011 年分别在 Sandy Bridge 以及 Bulldozer 架构上提供支持。AVX 对 XMM 寄存器做了扩展，从原来的128 位扩展到了256 位。

2013年，Intel 在发布的 Haswell 处理器上开始支持AVX2。同年，Intel 提出了 AVX-512。

2016 年，Xeon Phi x200 (Knights Landing) 是第一款支持了 AVX-512 的 CPU。如扩展名所示，AVX-512 主要改进是把 SIMD 寄存器扩展到了 512 位。

#### 用编译器自动SIMD

在编译程序时，可以直接通过-O3 -mavx2的方式来使用SIMD指令集，这种方式依赖编译器自动向量化，很多行为依赖编译器特性。

比如这个例子

```cpp
void vectorization(int arr[], int size, int val)
{
    for (int i = 0; i < size; i++)
    {
        arr[i] = arr[i] * i + val;
    }

}

int main()
{

int arr[128];
vectorization(arr, 128, 123456);
return 0;
}
```

1. 首先看一下cpu支持的指令集，通过cat /proc/cpuinfo可以看到，我这个cpu是支持avx,avx2等指令集的
2. gcc -O3 -mavx2 -S simd.cpp\
   \
   `flags : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 ht syscall nx mmxext fxsr_opt pdpe1gb rdtscp lm rep_good nopl extd_apicid eagerfpu pni pclmulqdq ssse3 fma cx16 sse4_1 sse4_2 x2apic movbe popcnt aes xsave avx f16c rdrand hypervisor lahf_lm cmp_legacy cr8_legacy abm sse4a misalignsse 3dnowprefetch osvw topoext vmmcall fsgsbase bmi1 avx2 smep bmi2 rdseed adx smap clflushopt xsaveopt xsavec xgetbv1 arat`
3.  可以看到已经使用了avx指令\


    ```
        vmovd   -36(%rsp), %xmm4
        vmovd   -28(%rsp), %xmm7
        vpinsrd $1, %ebx, %xmm6, %xmm1
        vmovd   -32(%rsp), %xmm5
        vpinsrd $1, %r13d, %xmm4, %xmm2
        vpinsrd $1, %ecx, %xmm7, %xmm3
        movl    %edx, -28(%rsp)
        vpinsrd $1, %r12d, %xmm5, %xmm0
        leaq    (%rdi,%r8,4), %rbx
        xorl    %ecx, %ecx
        xorl    %r8d, %r8d
        vpunpcklqdq     %xmm1, %xmm3, %xmm1
        vmovdqa .LC0(%rip), %ymm3
        vpunpcklqdq     %xmm2, %xmm0, %xmm0
        vbroadcastss    -28(%rsp), %ymm2
        vinserti128     $0x1, %xmm0, %ymm1, %ymm0
    ```

#### Intel Intrinsics

直接用SIMD指令写程序门槛太高，所以intel提供了一些封装好的内联函数，叫Intrinsics\
[\
官方使用手册：https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html](https://www.intel.com/content/www/us/en/docs/intrinsics-guide/index.html)

