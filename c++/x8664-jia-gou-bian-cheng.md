# X86-64架构编程

### 1 寄存器

{% embed url="http://6.s081.scripts.mit.edu/sp18/x86-64-architecture-guide.html" %}

在汇编语法里，寄存器的名字开头前缀是%，所有的寄存器都是64位宽度的。

<figure><img src="../.gitbook/assets/image (2) (1) (1).png" alt=""><figcaption></figcaption></figure>

### 2 指令集

每个 mnemonic opcode 都代表一组指令。在每组指令内部，有一些变种，可能会用不同的参数类型（寄存器，立即数，或者内存地址）和参数大小（byte，word，dword，qword）。前者可以从参数的前缀来分辨，后者可以从指令的后缀来看。

举个例子，一个 mov 指令，把64-bit 寄存器%rax 置为立即数3可以这么写：

movq $3, %rax

立即数的前缀一般都是$，没有前缀的操作数会被当成内存地址。

<figure><img src="../.gitbook/assets/image (2) (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

### 3 Stack Organization

全局和本地的变量存到栈上，栈是由 寄存器%rbp 和%rsp 指定的一个内存区域。每个过程调用都会产生一个 stack frame，栈的本地变量和产生的临时数值就放在 stack frame 里。

<figure><img src="../.gitbook/assets/image (3) (1).png" alt=""><figcaption></figcaption></figure>

### 4 Calling Convention

使用了标准的 Linux 函数调用约束，细节在[https://refspecs.linuxbase.org/elf/x86\_64-abi-0.99.pdf](https://refspecs.linuxbase.org/elf/x86_64-abi-0.99.pdf)。

caller 使用寄存器传递前六个参数到 callee。从左到右，分别是%rdi %rsi %rdx %rcx %r8 %r9。剩下的参数在栈上。

callee 负责保护寄存器%rbp %rbx %r12-%r15，因为这些寄存器是 caller 所有，剩余的寄存器属于 callee。

callee 把返回值放到%rax，负责清理它的本地变量，以及移除栈上的 return 地址

call enter leave ret 指令可以让调用约束变得简单。

