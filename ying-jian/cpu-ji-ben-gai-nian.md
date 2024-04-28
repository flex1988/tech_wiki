# CPU 基本概念

#### 1 什么是 CPU

中央处理器（CPU）是一种硬件组件，它是服务器的核心计算单元。服务器和其他智能设备将数据转换为数字信号，并对其进行数学运算。CPU 是处理信号并进行计算的主要组件，就像计算设备的大脑。从内存中获取指令，执行所需任务，并将输出发送回内存。负责处理操作系统和应用程序运行所需的所有计算任务。

#### 2 CPU 的组成

CPU 是一种复杂的电子电路，由多个处理数据和运行指令的关键组件组成。

**控制单元**

控制单元管理指令处理并协调 CPU 内部以及其他计算机组件之间的数据流。该单元有一个指令解码器组件，用于解释从内存中获取的指令，并将其转换为 CPU 可以运行的微操作。控制单元指示其他其他 CPU 组件执行所需的操作。

**寄存器**

寄存器是CPU 内的小型高速内存存储位置，这些位置存放 CPU当前正在处理的数据，便于快速访问数据。CPU 有多种类型的寄存器，例如：通用寄存器，指令寄存器，程序计数器等等。

**ALU**

算术逻辑单元（ALU）对数据执行基本的算术运算（加，减，乘，除）和逻辑运算（AND/OR/NOT）。该单元从CPU 内的寄存器接收数据，根据控制单元的指令对其进行处理，然后生成结果。

**内存管理单元**

可能有单独的总线接口单元或内存管理单元，取决于 CPU 架构。

**时钟**

CPU依靠时钟信号来同步其内部操作。时钟在特定频率下产生稳定的脉冲，这些时钟周期协调 CPU的操作。时钟速度以赫兹（HZ）为单位测量，决定了 CPU 每秒可执行多少条指令。现代 CPU 具有可变的时钟速度，可根据工作负载进行调整，以平衡性能和功耗。

**CPU Socket**

CPU芯片是要通过主板上的插槽链接到服务器上的，这种插槽叫做CPU Socket，一般有几个 Socket也就有几个 CPU 芯片。

**CORE**

随着 CPU 多核技术的发展，一般一个 CPU 芯片上会有多个核，每个核就是一个 CORE，多个CORE 之间可以并行执行逻辑，每个 CORE 都有自己独立的寄存器和 L1 Cache L2 Cache

**Hyper Threading**

Hyper threading 技术使操作系统以为实际的 CORE 数是物理 CORE 数的两倍，当程序频繁加载数据等操作时，可以把核心让出来给另一个线程，因此可以提升性能。

以下图为例，Architecture代表了芯片的架构，CPU 逻辑核96个，2个 socket，每个 socket 24个物理核，Thread(s) per core:    2说明开启了hyper threading。

<pre><code>Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                96
On-line CPU(s) list:   0-95
<strong>Thread(s) per core:    2
</strong>Core(s) per socket:    24
Socket(s):             2
NUMA node(s):          2
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 85
Model name:            Intel(R) Xeon(R) Platinum 8255C CPU @ 2.50GHz
Stepping:              7
CPU MHz:               3100.079
CPU max MHz:           2501.0000
CPU min MHz:           1000.0000
BogoMIPS:              5000.00
Virtualization:        VT-x
L1d cache:             32K
L1i cache:             32K
L2 cache:              1024K
L3 cache:              36608K
NUMA node0 CPU(s):     0-23,48-71
NUMA node1 CPU(s):     24-47,72-95
</code></pre>
