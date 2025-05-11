# 固态盘原理和故障分析

NVMe SSD是一种使用NVMe协议的高速固态盘，通过PCIe插槽与服务器连接，相比于传统的SATA SSD能够提供高达百万级的iops和低至微妙级的时延。

在现代的数据中心领域，特别是存储领域，NVMe SSD得到了广泛应用，已经称为了事实上的标准。

为了理解SSD故障的根因，我们最好先了解SSD的内部构造和原理，然后通过原理去分析SSD的故障产生，以及如何去更好的使用SSD以及诊断故障。

SSD的组成如左图所示，由控制器和一组NAND芯片组成。右图可以看到控制器更详细的组成，一个控制器内部由接口（PCIe，SATA，SAS），DRAM，FTL，Channel等组成，Channel连接了控制器和NAND芯片。\


<figure><img src="../.gitbook/assets/image (1) (1).png" alt=""><figcaption></figcaption></figure>

## Cell <a href="#cell" id="cell"></a>

在SSD内部，每个存储单元被称为一个Cell，Cell如何保存0或1，以及如何保证掉电后数据不会丢失，是整个存储器最基本的设计。

### 浮删晶体管 <a href="#fu-shan-jing-ti-guan" id="fu-shan-jing-ti-guan"></a>

浮栅晶体管（Floating-gate MOSFET）简称FGMOS，是由施敏（Simon M.Sze）首次发表在1967年5月16日贝尔实验室内部的期刊上。

浮栅晶体管的基本结构包括：控制栅极，氧化层，浮动栅极，隧道氧化层，衬底，源极和漏极。

写入时，在控制栅极施加正电压，电子就会通过隧道氧化层进入浮栅层，完成写入操作

擦除时，在衬底施加正电压，可以将电子从浮栅层吸取出来，完成擦除操作

读取时，在给控制栅极施加特定的电压时，浮栅极有电子时由于同性相斥，使通道处于不导通状态表示『0』，当浮栅极没有电子时，通道处于导通状态表示『1』

但随着擦写次数的增多，隧道氧化层老化，存储在浮栅中的电子就容易流式而导致数据出错，**所以每个存储单元的擦写次数是固定的**

在2014年闪存峰会上，施敏的努力终于得到了世人的认可，他因为发明浮栅晶体管而获得了终身成就奖，那一年全世界已经有了1021个浮栅晶体管，平均每个地球人都能分到上千亿个。

### **Charge Trap** <a href="#charge-trap" id="charge-trap"></a>

有了Floating-gate以后，业界又有人提出了Charge Trap，思路是把FG中的导体替换为绝缘体（一般是氮化硅），用捕捉的形式存储电子。

相比于FG，CT的电子存储更稳定，使隧道氧化层可以做的更薄，也不容易老化。

但也不是CT各方面都比FG好，在读取干扰和数据保持时间上，FG理论上会比CT表现更好。

### NAND Flash Layout <a href="#nand-flash-layout" id="nand-flash-layout"></a>

<figure><img src="../.gitbook/assets/image (1) (1) (1).png" alt=""><figcaption></figcaption></figure>

当前的主流SSD内部架构如图所示，层次是Chip->Die->Plane->Block->Page

Channel是控制器和NAND Flash存储芯片传输数据的通道，多个Channel之间可以并发传输

Chip是指一个NAND Flash存储芯片，多个Chip会共享一个Channel

Chip内部分为多个Die，每个Die是独立的硅体晶片

Die内部会分成多个Plane，每个Plane有自己的Cache Register和Page Register

Plane内部有若干个Block，擦除的单位是Block

Block内部有若干个Page，每次读写的最小粒度就是一个Page

Plane内部Block按照次序有顺序编号的ID，多个Chip上的Plane同一ID的Block组成了一个SuperBlock，同样的同ID的Page组成了一个SuperPage。

盘内的固件，每次会打开一个SuperBlock以SuperPage的粒度进行顺序写入，这样可以最大程度的提高写入的并发性。

在Block内部，cell之间为串联，一个cell的源极连接下一个cell的漏极，Wordline是字线，每条WL把一组cell的控制栅极连接，Bitline是位线，每条BL都会把一组cell中的一个cell连接起来。

每一行上的所有的Cell就组成了一个Page，每次读写必须已Page为粒度进行读取和写入。

当读取一个Page时，Block中未被选取的Page会被施加一个正电压，确保未被选中的Cell是导通的，这个正电压不会导致FG真的吸入很多电子影响自己保存的结果。

但长此以往，每次吸入一些，最后就可能会引起bit反转，但这个不是永久性损伤，重新擦写后可以继续使用。

写操作也有类似的问题，这种现象是读写干扰，为了解决这些问题，ssd一般使用ECC等技术对数据进行编码，以在读取的过程中对数据纠错。

在不断的发展过程中，NAND Flash又出现了可以在一个Cell里存储2个bit的MLC，3个bit的TLC，4个bit的QLC等。

以及为了进一步的提高容量，降低成本，NAND的制程工艺也从50nm一路到现在16nm，但是随着工艺的提升，NAND氧化层更薄，也导致了可靠性的下降。

为了解决这个问题，厂商们又将多个2d NAND堆叠起来形成一个3D NAND，既保证可靠性有提升了容量。

在3D NAND时代，在存储单元的技术方案上，各厂商基本都采用了Charge Trap的方案，但Intel和镁光仍然使用Floating-Gate的技术。

## FTL <a href="#ftl" id="ftl"></a>

通过上面NAND的介绍，可以看到NAND Flash的一些特性

\
1\. Page不能原地写，为了保证随机写的接口，需要多做一层映射，而且会产生垃圾需要GC

2\. 闪存单元的写入擦除次数是有限的，要尽量保证所有的闪存单元的PE次数是均衡的

3\. 闪存产生坏块需要做坏块管理

ds]asdasdasdasdkjk

FTL是SSD内部的关键组件，它负责L2P的映射，以及垃圾回收（GC），磨损平衡（Wear Leveling），坏块管理，读干扰处理等功能。

#### 垃圾回收 <a href="#la-ji-hui-shou" id="la-ji-hui-shou"></a>

OP空间一般占固态盘总体可用空间的7%-28%，若OP空间过小，固态盘将更频繁的进行垃圾回收，写放大系数更大，若OP空间过大，又会引起固态盘空间的浪费增加存储成本。
