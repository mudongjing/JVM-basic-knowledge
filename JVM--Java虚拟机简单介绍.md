> 首先，如果需要真正深入了解jvm的内部情况，可以登录java的官网下载对应版本的[规范](https://docs.oracle.com/javase/specs/index.html "规范")，可下载对应的pdf文件做详细了解。也可以阅读《深入理解Java虚拟机：JVM高级特性与最佳实践》。

> 本文主要为读者描绘jvm的大致情况，方便读者在之后的学习中有方向性地了解更细节的内容。主要基于当前常用的JAVA8。

> java虚拟机规范并不是关于虚拟机的实现，而是介绍了一些规则，因此读者会发现为什么虚拟机还有什么hotspot之类。并且由于虚拟机对应的时class文件，因此其它语言可以依赖自己的编译器获得一个可以运行在java虚拟机上的文件。

<table><tr><td bgcolor=yellow>本文内容较杂，但由浅入深，将各部分知识敲碎再组合。适合一次性读完。</td></tr></table>

[TOC]

### 1. 虚拟机主要内容

- 首先，从代码运行开始，虚拟机需要加载class文件，此时需要依赖类加载器。
  
  1. 由于类既有Java自带的类，还有用户定义的类，以及一些特殊场景下需要特别照顾的类。
  
  2. 于是，虚拟机内部包含多种类加载器，同时用户也可以为特殊场景定义自己的类加载器。

- 在类加载之后，就到了内存之中。内存管理则是一个大内容。
  
  - 首先，宏观上，内存包含了以下两类，公有区和私有区：
    - <font color=Blue>公有区：</font>
      
        1. 堆(heap)
        2. 元空间【方法区（这是逻辑概念】(MetaSpace)
    - <font color=Blue>私有区：</font>
      1. 虚拟机栈
      2. 本地方法栈
      3. 程序计数器
  - 其次，垃圾回收器（GC)用于垃圾对象占用内存的回收
    
    - <font color=Orange>垃圾回收算法</font>
    
    - <font color=Orange>现有主流垃圾回收器</font>
  
- 对象创建流程
  
  - 首先将对应的class文件内容加载到方法区中，【在hotspot中，先生成一个对应的c++对象，之后在堆中生成实际的java对象】
    - 实际上，对象名在调用对象时，首先指向c++对象，该对象中包含着对Java对象的指针。
  - java对象的保存是在堆中，但内部仍然包含一系列选择，包含栈存储，TLAB，Eden,S1/S0,Old

### 2. 类加载

#### 2.1 简要概述

所谓类加载，是指将class文件加载到内存中，由JVM对数据做校验、解析、初始化，最终得到java类型。而class文件本身是按照规范的一组指令的有序集合。

> 类加载是在运行期间完成，而不是c++那种提前编译，导致运行效率相对低，但是由此我们可以在运行期间对class文件做操作，更具灵活性和扩展性。例如cglib的动态代理，就是在运行期间完成一个代理对象的class实现并加载的内存中替代原有对象，实现功能增强。
> 更厉害的是，Alibaba的arthas软件，可以在项目运行期间，替换其中的某个class文件【避免小问题导致的项目整体停止】，这些是提前编译完全做不到的。

------------------------------------------------------------已有的类加载器-------------------------------------------------------------------------

```text
BootstrapClassLoader(最底层)【实际是由c++实现的(针对hotspot)】
ExtClassLoader（扩展类）
AppClassLoader
```

例如，我们希望创建一个HashMap的对象，由于这种属于jdk较为底层的类，不是你开发者可以随便搞的，也算不上是扩展的，就需要交由BootstrapClassLoader负责创建，类似地，不同层级的类由对应的加载器负责。
如此划分，加载器能够各司其职，防止底层的类被后来的类覆盖。同时，如果项目过大，包含了众多的依赖或模块，出现多个类的全限定名相同，这一机制能保证只有一个类能实现，而不是出现多个造成歧义。

> 各加载器的加载路径获得方式：
> BootstrapClassLoader：System.getProperty("sun.boot.class.path")

> ExtClassLoader：System.getProperty("java.ext.dirs")
> AppClassLoader:System.getProperty("java.class.path")

> 通过路径可获悉不同加载器负责的类的范围。其中ext可以在本地安装路径中的jre\lib\ext中看到不少jar文件。

#### 2.2 加载过程

> 类的加载需要先查找现有的类，再创建没有的类

每个加载器有自己的缓存保存已创建的类。
当一个类需要加载时，首先到达AppClassLoader中，由它查找自身管理的类中是否有现成，则直接返回，否则询问ExtClassLoader，由它再查找它管理的类中是否有现成的，类似地，到达bootBootstrapClassLoader，当发现没有的现成的，它将看看这个类是否是由它负责的，则创建并返回，否则把任务交给ExtClassLoader，类似地，再传递给AppClassLoader，总会在这里创建出来的。
---------这一过程被称为双亲委派机制【就是可以向上或向下踢皮球】

>  *双亲委派机制只是JVM的一种推荐做法，实际的一些实现可能有自己的加载方式，可以了解一下Tomcat的方式，所有的类都会自行加载，只有实现不了才会委派上层完成。*

> 如果学习过spring系列，或操作过动态代理的同学，估计会注意到有时参数需要传递一个类的class对象或赋予一个类加载器作为参数。
> 即使用方法getclassloader()获得，通过这个方法可以获悉对象是由哪个加载的，进一步的getParent()可以得知加载器的上层是哪个。但是在查询ExtClassLoader的父加载器时，返回的是null【因为前文告知了，hotspot最底层的加载器是c++实现的，没有对应的java代码】（常用的JVM多是HotSpot，它本身是c++编写的，而其它JVM，如MRP、Maxine是Java编写的，对应的bootstrap则不为null)
> 
> 【BootstrapClassLoader是与对应的JVM为一个整体，而其它类加载器则属于后来者】

另外加载器可以实现自定义，jdk中关于加载器的定义的是：

1. 除bootstrap】均继承自ClassLoader抽象类
2. 下面有SecureClassLoader实现类
3. 再下面有URLClassLoader类
4. 再下面就有了AppClassLoader、ExtClassLoader类，可以继承自上面的类自定义加载器。

<font color=Red>【需要注意的是，类的继承关系，和踢皮球的关系可不相同，新定义的类加载器无论继承自谁，它的parent都是AppClassLoader。】</font>

### 3. 内存管理

这里主要先介绍内存的各个组成部分的作用。

#### 3.1 内存公有区

- 元空间（方法区）：方法区本身是逻辑概念，相当于接口，是由永久代或元空间实现出来的，，由于这个区域没有垃圾回收，因此可以适当调大一些，防止报OutOfMemoryError：Java heap space。
  
  - 主要负责存储：类信息、常量、运行时常量池、静态常量、即时编译器编译后的代码

- 堆：包含多个部分，Eden、S0/S1(From Survior,To SUrvior)、Old(老年代)。主要存储实例对象，包括数组之类的。【-Xmx,-Xms负责调节堆的大小】
  
  - 实例对象生成后，将放在堆中，位置顺序是，Eden->S0<-->S1->老年代。

> ~~jdk1.7实现的方法区称为永久代，1.8则实现为元空间，本质上是与堆保存在一起，但概念上有所差别（jdk1.7合并给堆）【1.7之前，hotSpot称之为永久代】，如果需要调整该区域的内存大小，可用jvm命令【-XX:MaxMetaspaceSize=..】（这是1.8的命令，之前的版本估计也没人在乎）~~

堆的内存分配比例：

<table border="0" bordercolor="black" width="300" cellspacing="0" cellpadding="0"><tr><td >老年代</td><td colspan="3">新生代</td></tr><tr><td >2/3</td><td colspan="3">1/3</td></tr><tr><td></td><td>Eden</td><td>S0</td><td>S1</td></tr><tr><td></td><td>80%</td><td>10%</td><td>10%</td></tr></table>

#### 3.2 内存私有区

由于现在的系统以及程序要求多线程，导致需要为不同的线程分配特有的空间进行独立运行。于是，需要分出这样一块内存用于划分成多个独立的小块。

每个小块内部包含虚拟机栈、本地方法栈以及程序计数器。

- 虚拟机栈：本身是一个栈，内部保存的是进程的局部变量和临时结果。内部每个方法又有独立的栈帧。
  
  - 栈帧内部包含：局部变量表，返回地址（方法结束位置），操作数栈，动态链接。
    1. 局部变量表中，引用类型保存为对应实例的地址。变量表由变量槽（slot）为单位计算大小。
    2. 操作数栈，用于执行方法对变量的计算操作。不过是将变量表中的数据进行入栈/出栈操作。（栈的运行深度有限制，否则产生StackOverFlowError，主要取决于给定的内存大小，可通过命令-Xss指定内存大小，IDEA中，可在运行配置中找到VM option选项填写命令，更多常用的配置命令可看一下[阿里的文档](https://help.aliyun.com/document_detail/148851.html)）
    3. 动态链接，其它方法在运行时常量池中的地址，用于方法调用。
    4. 若栈在运行期间需要扩展时，发现无法提供足够的空间，则会产生OutOfMemoryError。
- 本地方法栈:这个栈实际与虚拟机栈是非常类似的，虽然给出了差别介绍，但实际的jvm将二者视为一体。只是，Java中有些内部的方法或可以自定义一些方法，它们的底层实现可以是其它语言，Java中有一些底层方法是由C/C++实现，并在方法前标记为native方法。包含这类native方法的将交由本地方法栈负责

【虚拟机栈和本地方法栈有时直接统称Java栈】

### 4. 垃圾回收器（GC）

GC的内容的内容较多，内部包含很多机制和算法，但真正需要做的就是对利用回收器做调优。

- 首先，判断垃圾

  1. 这是一个非常老掉牙的知识点：
     - 主要是，最初有一个引用计数法，当一个实例对象的身上带有的引用数量不为0，则代表它仍有用，则不视为垃圾，<font color=Red >但实际上可以有垃圾抱团的现象</font>；
     - 实际使用的<font color=Blue>根可达性方法</font>。
       - 由于程序的运行通常从一个main方法作为入口进入，那么从这个入口开始，内部开始产生多个对象（这里我们姑且称之为main对象），而这些对象之后可能会使用另外一些对象，如调用一些外部方法之类的，总之，之后产生的所有对象都与这些main对象有着引用关系。
     - 引用分类：
       1. 强引用：传统的引用
       2. 软引用：连接那些不必要但可以有用的对象。如果将要发生内存溢出时，将回收这部分对象。SoftReference类
       3. 弱引用：更弱，对象只能生存到下次垃圾回收。WeakReference
       4. 虚引用：近乎不存在，无法用过这个引用获得对象实例，只是为了在回收了该对象后有一个系统通知。PhantomRefernce。
  2. jvm内部保存着所有的对象，从main对象开始，按照引用关系，一步步向下搜索，只要被搜索到，即视为有用的对象，而其它对象自然是垃圾。

- 识别到垃圾对象后，需要做的就是回收垃圾
  
  - 实际的垃圾回收，也是一个老掉牙的知识点了。
    
    - 1、标记清除：顾名思义地，直接回收垃圾的内存，但造成大量内存碎片。
    - 2、复制清除：耗费内存，直接将原有的内存划为两半，一半将另一半有用的对象直接复制过来，对面的内存直接变成整个空闲内存。
    - 3、标记压缩：标记清除的同时，将碎片整理起来，速度慢。
    
    > ~~需要告知的是，由于回收垃圾可能会影响程序的运行效率，jvm并不是发现垃圾就回收，即使有一些方法表面意思是释放对象内存，也只是做个样子，只有在内存真的紧张时，才可能出发垃圾回收机制~~
  
- 垃圾回收器：
  
  1. 经典垃圾回收器：Serial、ParNew、Parallel Scavenge、Serial Old、CMS、Garbage First(G1)
  2. 低延迟垃圾回收器：Shenandoah、ZGC
  3. Epsilon

- 新生/老年代：在jdk1.8及之前，是做上述堆的划分的。经典回收器也是基于此作相应的设计。
  
  - 新生代就是前面提及的Eden、S0/S1，当对象创建初期大多放在这里
  - 老年代就是Old喽！只要对象活得够久或空间够大就可以放在这里。

--------------------------------------------------------------------

### 5. 对象存储

<font color=Orange>首先要注意，类产生的对象和作为数组的对象在创建机理上是不同的。类对象有类加载器一层层负责，而数组则直接由Bootstrap负责。</font> 【在JVM规范中，指出了底层的实现中，类对象使用了常用的new，而数组则有自己的newarray等方法】

#### 5.1 实例存储

- 虽然过程有所区别，但经历了加载机制和创建过程后，最终都产生了相应的实例对象，并将实例存放在堆【正常来说】中。

```mermaid
graph TD
id1([实例产生])-->方法区存放引用
id1 -->id2(堆中存放实际对象)
id2--> id3{是否发生逃逸或评估对象占用空间过大}
id3-->|YES|id4(尝试在TLAB中存储)
id3-->|NO|id5(进入栈上分配)
id4-->id6{TLAB是否具有足够的空间}
id6-->|YES|分配在TLAB
id6-->|NO|id7{对象是否过大且超过阈值}
id7-->|YES|id8(分配到Eden)
id7-->|NO|分配到TLAB
id8-->id9[发生垃圾回收]
id9-->id10(转移到S0或S1)
id10-->idman(S0或S1内存已满)
idman-->id12
id10-->id90(发生垃圾回收)
id90-->id11{是否到达指定寿命}
id11-->|YES|id12(转移到Old区)
id11-->|NO|id9
id10-->|退出|等待内存回收
id12-->|退出|等待内存回收
```

对象创建后，具体的存储流程如上图。

> 首先是，试图放在方法栈中，其次考虑TLAB，最后分配到熟悉的Eden,S0/S1,Old区。

下面简单介绍一下上述的一些过程细节：

- 为了提高效率，首先试图将对象放在栈中，这里的栈为方法栈。但首先不能过大，否则存不下；其次，由于对象是相应的方法产生的，改对象需要在此对象的控制之下，而不能被其它外部的变量调用，否则则发生了逃逸。
  - 由于是放在方法栈中，对象会随着方法的结束而结束，不需要GC，但是发生逃逸则导致方法结束了，对象不应该结束。则破坏了栈的结构。
- TLAB(Thread Local Allocation Buffer)【本地分配缓存区】，是线程在Eden区预留的一块内存，空间非常小。
  - 如果对象够小，且有足够的空间，就直接放在这里。
    - 这里还设置了一个阈值，只要超过，就省去中间的环节，直接放在Old区。
    - 较大但还可以的，则放在Eden区。
- Eden区是一个新对象的集中区，基本上新产生的实例都放在这里，而新的实例大多也短命。在系统触发了垃圾回收机制后，仍存活的实例就转移到S0或S1中。
- S0/S1，是遵循复制清除的垃圾处理机制，S0/S1就是被一份为二的内存。每次触发了垃圾回收机制，双方就进行转移。
  - 并且，每经历依此垃圾回收，或者的实例的寿命就+1。直到到达指定的寿命，实例就会转移到Old区【这里寿命，一般为15】。

#### 5.2 对象定位

对象的定位，自然需要获取它在内存中的地址，而这个地址则放在Java栈的本地变量表中，但真正获取到对象，这里有两种方式：【为了得到完善的对象信息，我们一方面要获得对象在堆中的数据信息，另一方面要获得方法区中对象的类型信息】

- 句柄访问：这是一种间接方式，堆中可能划分出一块内存作为句柄池，而本地变量表中的地址则指向这个句柄池。句柄池中包含了我们需要的两个信息的指针。【这个方法较慢，但好处在于变量表与实际对象地址是解耦合的，尤其在复制清除算法中，避免了频繁修改变量表中的值】
- 指针访问：这个方法符合我们的预期，它首先指向堆中的对象数据信息，而其中包含了指向方法区中对象类型信息的指针，再到达方法区获取类信息。【这个方法很快，也是hotspot的主要方式】

> 由于hotspot底层为c++，需要创造一种数据结构来描述java类与对象的信息。 hotspot中对于类及其对象的描述使用OOP-Klass模型【OOP(Ordinary Object Pointer)描述实例信息，Klass描述类信息，是与对应java类相关的c++对等体】

- OOP一定程度上包含了java对象的地址信息
- Klass则是一个抽象基类，描述了对应Java类的信息
  - 【除了类本身的信息，为了保证多态性，另外维护了一个虚函数表】

OOP-Klass导致了在hotspot的底层上，调用实例的过程实际上是先到达c++的对等体上，获得了Java对象的地址，再到达堆中实例的位置。【更多关于OOP-Klass模型的信息，可参考附录14.2】

### 6. 垃圾回收操作

#### 6.1 前期准备

> 前面已经简单介绍了垃圾回收器的概况，但只是介绍了大致的思路，具体如何操作，则涉及到垃圾如何记录，以及安全地清除垃圾。

- 首先是垃圾回收的触发

  由于假设不同对象的寿命有所区分可划分出了不同的区域，就有了分代收集。

  - Minor Collection：对新生代进行收集，【S0/S1是复制清除】
  - Full Collection：所有。【标记压缩清除】

  当Eden区内存耗尽，则触发Minor Collection。当老年代已满，则触发Full Colleciton。

- 查找根节点【又称根节点枚举】，即我们在进行可达性分析首先需要的起始点。

  - 这一过程速度较快，但是为了保证一致性，即不能在我们查找根节点的时候，这些节点还在不停变化。【则需要进行快照，而且<font color=Red>这一过程要求暂停一切用户线程</font> 】

> 由于线程是具有一个私有区域，私有区域中又包含了虚拟机栈，虚拟机栈中又包含了局部变量表，局部变量表中又包含了各种引用地址。Hotspot定义一个数据结构（Oop Map)，表示对象内部各偏移量对应的数据类型，以及以及<font color=Orange>特定位置</font>栈和寄存器中引用的位置。

- 通过Oop Map可快速地分析各个引用，但是不能使用过多，否则存储空间成本过大。因此，我们的特定位置就是设定一个==安全点==。
  - 当到达安全点时，线程将不会继续向下执行，直到jvm完成当前的要做的步骤
    - 主动式中断：在安全点出设置一个标志，以表明当前是否完成了垃圾收集操作，否则线程将轮询访问该标志
    - 抢先式中断
  - 但是，如果一个线程处于休眠状态，而没有进行适当的中断，可能会错过这样的安全点。因此需要额外引入==安全区域==
    - 即扩大了指定的代码区域
- 记忆集，前面提到了分代收集，例如收集新生代垃圾时，如何判断这些对象没有被老年代或方法栈中的对象引用着，就需要这样一个记录跨代引用的数据结构。【避免了从头开始遍历所有引用】
  - 为了优化记忆集的存储和维护成本，需要考虑其记录精度，（字长精度，即处理器的寻址位数，直接记录跨代指针；对象精度，记录的对象中包含一个跨代指针的字段；卡精度，记录一块内存区域，内存中包含对象，对象又包含跨代指针）
    - 卡精度是常用方式，又称**==卡表==**。hotspot中用字节数组表示卡表，卡表内的元素为卡页，卡页的值是对应内存地址的起始位置【源码中CARD_TABLE [this address >>9]=0;表示一个卡页负责的范围是2^9字节】
    - 一个卡页可以包含多个对象，只要这些对象中有一个具有跨代指针，那么整个卡页被标记为变脏，之后的垃圾回收，只要识别这些变脏的区域并进行扫描即可。
    - **注意**：记忆集是一种抽象概念，可以有不同实现，而卡表只是其中一种具体实现而已。

#### 6.2 回收器操作

前面已经提及理论当前Java自带的10中垃圾回收器，但其中一些并不是直接使用，而是根据分代收集而组合完成工作。其中G1，Shenandoah，ZGC是可以独立使用的，Epsilon稍微特殊。

```mermaid
graph LR
serial(Serial)----|<font Color=Red>X</font> JDK9|CMS(CMS)
serial----serialold(Serial Old)
CMS----serialold
parnew(ParNew)----|<font Color=Red>X</font> JDK9|serialold
parnew----CMS
paralleS(Parallel Scavenge) ----serialold
parallO(Parallel Old)----paralleS

```

上图中，相连的双方代表可以组合使用。（部分是在JDK9及之后被废弃）

- 传统
  - Serial：单线程，且必须暂停其它工作线程【Stop The World (STW)】，主要应用与新生代。
    - 不先进，但资源要求小。在小规模环境下使用较好。
  
  ```mermaid
  sequenceDiagram
  	autonumber
      用户线程->>safePoint1: 运行
      safePoint1->>GC线程: (暂停其它线程STW) 对新生代采用复制清除算法
      GC线程->>用户线程: 结束STW
      用户线程-->>safePoint2:运行
    	safePoint2-->>GC线程: STW, 对老年代采用标记整理算法
  ```
  
  

<center>Serial/Serial Old示意图</center>

- ParNew【简称PN】：多线程并行版Serial

$$
吞吐量=\frac{用户代码运行时间}{用户代码运行时间+垃圾收集时间}
$$



- Parallel Scavenge【简称PS】：多线程的新生代收集器。可设定吞吐量相关的参数。
- Serial Old【简称SO】：单线程，使用标记整理算法，主要用于老年代。
  - 可作为CMS失败时的后备方案
  - Paralle Scavenge内部具有PS MarkSweep收集器进行老年代收集，这一收集器的实现本质上与SO相同。【JDK5及之前】，讲解是说PS与SO组合使用。

- Parallel Old【简称PO】：Parallel Scavenge的老年代版本，多线程，标记整理算法。【JDK6伊始】。常常优先考虑PS-PO组合

- CMS(Concurrent-Marking-Sweep)：着重于低延迟，多线程并发，老年代，标记清除算法。

  ```mermaid
  sequenceDiagram
  	autonumber
      用户线程->>safePoint1:运行
      safePoint1->>safePoint2:STW，初始标记
      safePoint2-->>用户线程:结束STW
      用户线程->>safePoint3:运行，并发标记
      safePoint3->>safePoint4:STW，重新标记
      safePoint4-->>用户线程:结束STW
      用户线程->>safePoint5:运行，并发清理
  ```

  - 初始标记：仅标记根节点
  - 并发标记：正常的从根节点向下遍历，由于此时用户线程不停止，导致结果与最后实际情况有误差
  - 重新标记：为修正误差，停止用户线程并修正结果
  - 并发清除：收集死亡对象

  > 默认的回收线程数为$\frac{CPU核心数+3}{4}$
  >
  > 核心数较少时，反而影响实际程序性能。
  >
  > - 增量式并发收集器：（JDK7就已经不提倡）效果一般，类似多任务抢占式运行，利用有限的线程交替执行不同任务，运行反而更慢。

  > CMS激活，JDK5默认老年代空间到达68%，JDK9默认92%。老年代必须预留一定空间共回收进程使用。否则将并发失败，需冻结用户进程，并启用Serial Old。

- G1：JDK9及之后的默认。一定程度上是CMS的继承者。可收集堆内存任意区域。

  - Region：G1将Java的堆划分为多个独立等大的区域【是垃圾回收的最小单位】，让这些区域根据需要充当Eden或老年代等角色【实际仍然是分代收集算法】
    - Humongous：存储大对象【超过Region容量的一半】，超过整个Region容量的对象为超级大对象，将存放在多个Humongous中。
    - Region大小可通过-XX:G1HeapRegionSize设置，范围为1MB~32MB，2的次幂。
  - G1中已没有传统的新生代和老年代。而是将所有的Region区域按照回收空间大小以及回收时间设定优先级并进行排列，有效地提高了垃圾回收的效率。
  -   Region也需要维护自己的卡表，G1的卡表同时记录了两种形式，我被谁引用了以及我引用了谁
    - 这里的卡表是一种哈希表，键值对应指向对象的Region地址，value包含其中对象所在的索引。
    - 由于设置的卡表较多，导致G1需要相对较多的空间存储这些信息
  - TAMS(Top at Mark Start)，Region中包含两个TAMS(pre,next)，pre对应的是在垃圾回收时，该区域内上次扫描开始的位置，next代表这次扫描开始的位置。【则区域内处于TAMS另一侧的】

  G1的大致操作如下：

  1. 初始标记：标记根节点
  2. 并发标记：递归扫描做可达性分析
  3. 最终标记：对结果做修正
  4. 筛选回收：【整体上标记整理算法，局部是标记复制算法】首先，根据回收价值和成本，以及用户期望的停顿时间，选择需要回收的区域，将这些Region中存活的对象复制到其它的空Region中，才将原有的Region整体清空。多条收集器线程并行执行。

  ```mermaid
  sequenceDiagram
  	autonumber
  	用户线程->> safePoint1:运行
  	safePoint1->> safePoint2:STW，初始标记
  	safePoint2-->> 用户线程:结束STW
  	用户线程->>safePoint3:运行，并发标记
  	safePoint3->>最终标记:STW
  	Note over 筛选回收: 多回收线程并行
  	最终标记->> 筛选回收:STW
  ```

  

- 低延迟
  - ZGC：基于Page或ZPage（相当于Region）的堆内存布局，但这里的Region根据动态性，可动态创建和销毁，并且有不同型号的容量。采用了==染色指针技术==进行标记。
    - 运作过程为：并发标记、并发预备重分配、并发重分配、并发重映射
  - shenandoah：目前，这一回收器似乎还是只有OpenJDK支持，而Oracle是故意反对的【不是sun自己做的】。
    - 从代码上看是对G1的继承者。均使用Region概念，以及相同的回收策略。不同点主要在于：
      - 支持了并发的整理算法，可与用户线程并发执行
      - 默认不使用分代收集
      - 废弃了记忆集，而是用连接矩阵记录引用关系
    - 步骤阶段为：初始标记、==并发标记==、最终标记、==并发清理==、==并发回收==、初始引用更新、最终引用更新、==并发清理==

- Epsilon



### 7. Class文件结构

class文件包含的主要内容有：

​	魔数，（副版本号，主版本号），（常量池计数器，常量池），

（访问标志，类索引，父类索引，接口计数器，接口表），

（字段计数器，字段表），（方法计数器，方法表），（属性计数器，属性表）。

> 关于class文件的内部信息，非常繁多，以至于java虚拟机规范专门列出一章讨论，更具体的细节，读者可查看规范，这里仅阐述整体的框架。

- 魔数(magic)：class文件的头4个字节。仅用于确定是否为一个class文件。【不同文件都具有这样的魔数，Java从class对应的魔数值为“0xCAFEBABE"(咖啡宝贝)】

- 版本号：魔数之后的2个字节为副版本后，再之后的2个字节为主版本号。主版本号从45开始计算。读者可将自己的class文件读取为16进制【推荐用sublime text打开】，不仅可以发现前面的魔数，也会发现后面对应的版本号，我这里使用的是Java11编译的结果：

  ```
  cafe babe 0000 0037
  ```

  主版本号对应的16进制的37，转换后为10进制的55。副版本号在JDK1.2~JDK12，都固定为0。

- 常量池计数器：版本号之后的两个字节用于存放常量池的项数，即对应的容量。假如对应的容量值是0x37，即55，但实际容量大小是54，且项数的索引值是1~54。索引值0是作为特殊存在。

- 常量池：常量池计数器之后便是记录常量池中的各个量。包括如字符串，final常量值的字面量，也包含如全限定名，方法名及其描述符的符号引用（因为java的类是在加载后才知道对应的内存地址，因此符号引用不是指针）

  - 常量池中的每一个项都是一组包含多个量的组合，规范中称其为表。
    - 表的第一个字节便是`tag`，标记类型，判断当前的表是何种的字面量，又或是何种的引用
    - 之后，不同类型的表内部的组成也不同，根据不同情况，JVM会对表读出字符串或获取方法名等信息，也会有表的内部包含常量池的索引值，可跳转到相应位置读取信息。

- 访问标志：常量池结束之后，就是2个字节的访问标志，用于识别接口或类的访问信息。

  - 用于确定对应的是类还是接口，又或是枚举，以及之后出现的模块，同时也确定这是public的还是其它的。

- 类索引，父类索引：均为2个字节。访问标志后便是类索引，用于确定对应的全限定名称，类中除了java.lang.Object都具有父类索引。

  - 全限定名的确定过程大致为，首先自己的2个字节是作为索引值使用，利用该索引值找到一个CONSTANT_Class_info的常量，在常量中再获取一个索引值，定位到一个CONSTANT_Utf8_info常量，内部的字符串便是需要的全限定名称。

- 接口计数器，接口表：在父类 索引之后就是接口计数器，记录该类实现的接口数量，2个字节。紧随其后，是一组接口索引的集合，索引各自占据2个字节。若未实现接口，则接口计数器为0，不存在接口表。

> 计数器均占据2个字节
>
> 下述表中，均会对各个元素做详细 的描述，如方法的访问权限等，总之将代码中的各种信息转化到表中对应符号标志

- 字段计数器，字段表：字段表表述当前类的字段，但不包括从接口或父类那继承来的
- 方法计数器，方法表：同样不会描述父类或父接口的方法
- 属性计数器，属性表：



### 8. 类加载过程

>  前文介绍的双亲委派机制，这一机制是jdk1.2之后引入的推荐方法，而不是强制的，具体的实现代码在java.lang.ClassLoader中的loadClass()方法中。

#### 8.1 类的初始化

- 另外在加载之后，需要进行连接，这里包含了三个步骤【验证、准备、解析】，之后进行初始化，才能开始使用，如果任务结束，则对类进行卸载。<font color=Red>其中，为了效率，连接中的一些操作会在加载阶段同步进行</font>，其中的步骤也不是完全具有先后顺序。

这里需要注意的是其中的初始化问题，何时需要真正初始化一个类，首先必须有一个类的情况下则必须初始化，如：

1. new一个对象，使用类的静态字段或方法。反射调用一个类。还有一个是动态语言生成的方法句柄需初始化对应的类。
2. 由于对象的继承关系，首先需要初始化其父类。或程序的main函数所在的类也必须初始化。
3. 特别的，由于jdk8后接口存在default方法，继承此接口的类初始化，必定带动这个接口初始化。

【特别的例子说明】

- 如果B继承了A，且二者内部均有各自的静态代码，例如A中有静态变量a,B中有静态语句输出。如果调用了B.a，此时并不会返回B中的静态输出，即没有实现B的初始化（但有可能完成了加载、验证）。
  - 因为此时，我们仅需要的是A的静态变量，B虽然出场了，但我们既然开始初始化了A，B是否存在就不重要了，导致B的类不是必须的。
- 如果我们定义了一个B类型的数组，实际也还是不会有所输出。我们当前只是完成了数组的分配，相当于盖房子，入住手续后面可以进行，没那么急。而且，这个定义的数组本质上也是一个类，是一个继承自Object由JVM自动生成的关于B的类，内部包装了数组操作。
- 类似地，如果A中的静态变量是一个常量，那么调用A.a时，A是不会初始化的。因为在编译期间，这些常量已提前放到了常量池中。此时这个变量和A有关系，但又不完全有关系。

#### 8.2 类的加载

前一节提前说了一下类的初始化，但这一切的开头都需要类的加载。

- 加载过程：由全限定名称获取字节流，转化为方法区内的运行时数据结构、生成一个java.lang.Class对象作为该类的访问接口。

其中的字节流可以通过一切手段获得。例如网络、jar包、数据库等。

> 其中的jar包读取，在对应的位置字符串中指明对应的协议和jar包位置，再接上“!/"表明读取包内的文件，之后就是接上对应jar包内文件的位置。如”jar:file:\\\【jar包位置】!/【文件位置】“

前文中提及了如果有需要的话，可以自定义类加载器。现在，我们就可以继承URLClassLoader，再覆盖对应的类加载方法，包括从哪里读取类文件(其中，就可以用上述jar包读取的方式从自定义的jar包中读取自己的class文件并加载到内存中)。

如果我们有多个类似的jar包，但是内部class文件的代码逻辑有所差别【类似于人类的双标行为】，此时就可以通过自定义的类加载器，视情况加载不同的jar包。

<font color=Red>【需要注意数组】</font>

需要说明一下数组的组件类型，即数组去掉一个维度后的类型，如String[] 组件类型就是String，而String[][] \[ ][ ]的组件类型就是String[]。

首先，数组不同于普通java对象，它是直接由JVM创建 ，而不是类加载器完成。就如前面的【特别的例子说明】中提及的，数组是被JVM直接创建了一个对应的类。

1. 如果组件类型是String这类的非引用型的，那么就直接交给BootstrapClassLoader进行创建。
2. 否则，就仍然是个数组的引用，
   - java8的虚拟机规范中介绍是：此时数组被标记为已由组件类型的定义类加载器定义。
     - 但之后，被去掉一个维度的数组再次进入这样一个加载流程中，不断地被去掉一个维度，并不断地被标记，直到最后不再是引用类型，到达BootstrapClassLoader进行创建。

### 9. 对象存储

#### 9.1 对象存储结构

对象在堆内存的存储布局为：对象头（header），实例数据（Instance Data），对齐填充（Padding）。

> 对齐填充是保证对象的大小为8字节的倍数。这与CPU的缓存行有关，使得每次加载后都能立刻被CPU刷新执行，提高效率。【现在一些CPU的缓存行已经较大，如64字节，为了提高效率，一些项目会在对象中故意加若干个long或其它类型的变量以进行主动填充】

其中对象头中包含了`mark word`、`元数据指针`，若为数组则另外有一个`数组长度`。这些元素的大小均与对应JVM的位数有关，例如64位的JVM，则均为64bit。

元数据指针(klass word)则是用于指向方法区中目标类的类型信息，也就是说通过元数据指针可以准确定位到当前对象的具体目标类型。

mark word中可以包含大量信息，如锁状态，GC年龄。且占用的空间位数与相应的CPU处理位数相同。

32位情况：

<table border="1" cellspacing="0" width="50%" height="150">
<tr><th rowspan="2">锁状态</th><th colspan="2">25bit</th> <th rowspan="2">4bit</th><th>1bit</th> <th>2bit</th></tr>
<tr><td>23bit</td> <td>2bit</td><td>是否偏向锁</td><td>锁标志位</td></tr>
<tr>
    <td>无锁状态</td>
    <td colspan="2">对象的hashcode</td><td>分代年龄</td><td>0<td>01</td>
</tr>
<tr>
	<td>轻量级锁</td><td colspan="4">指向栈中锁记录的指针</td><td>00</td>
</tr>
<tr>
<td>重量级锁</td><td colspan="4">指向互斥量(重量级锁)的指针</td><td>10</td>
</tr>
<tr>
<td>GC标记</td><td colspan="4">空</td><td>11</td>
</tr>
<tr>
<td>偏向锁</td><td>线程ID</td><td>Epoch</td><td>分代年龄</td><td>1</td><td>01</td>
</tr>
</table>

64位情况：

![](http://p2pworker.xyz/wp-content/uploads/2021/06/markword64.png)



#### 9.2 指针压缩

- 对象头
  - 32位系统：占用 8 字节(markWord4字节+kclass4字节)
  - 64位系统：开启压缩指针时，占用 12 字节，否则是16字节(markWord8字节+kclass8字节，开启时markWord8字节+kclass4字节)
- 引用类型
  - 32位系统：占4字节 (因为此引用类型要去方法区中找类信息,所以地址为32位即4字节同理64位是8字节)
  - 64位系统：开启压缩指针时，占用4字节，否则是8字节

> 如果所使用的 64 位虚拟机的版本在 Update14-Update22 之间，可以通过选项
>
> “-XX:+UseCompressedOops”显式开启指针压缩功能，而在 Update23 版本之后，指针压缩功能将会被缺省开启。
>
> JDK7及之后是默认开启的。

以下举例说明对象在64位环境下空间的冗余：

​		32位操作系统 花费的内存空间为：对象头(8字节) + 实例数据(int类型(4字节) + 引用类型(4字节)) +对齐(0字节)																共16个字节

​		64位操作系统：对象头(16字节) + 实例数据 (int类型(4字节) + 引用类型(8字节))+对齐(4字节)

​																共 32个字节

​		同样的对象需要将近两倍的容量,所以需要开启压缩指针：

​		64位开启压缩指针后：对象头(12字节) + 实例数据(int类型(4字节) + 引用类型(4字节))+对齐(0字节)

​																 共24个字节

**JVM的实现方式是:**

- 通过前面填充对齐的了解，我们知道JVM中存储的对象大小均是8字节的倍数，因此我们在定位对象时，不需要考虑对象中间的7个字节，引用的地址只需要是8的倍数就可以完成工作。
- 在64位的系统中，对象的地址仍然使用32位表示，但是当引用实际被使用时，即进入了寄存器时，则左移3位，变成了35位，可定位32GB的内存空间。
  - 为什么不多左移几位呢？那样就要求对象的大小是更多字节的倍数，对齐填充可能占据更多的内存。

> 因此当堆内存大小超过32GB，就不能使用压缩指针来工作了。
>
> 而且当堆的大小仅仅略大于32GB，由于指针扩大为两倍，需要消耗更多的空间，可能导致实际能用的空间不如32GB的情况。

​	

- 哪些信息会被压缩？
  1.对象的全局静态变量(即类属性)
  2.对象头信息:64位平台下，原生对象头大小为16字节，压缩后为12字节
  3.对象的引用类型:64位平台下，引用类型本身大小为8字节，压缩后为4字节
  4.对象数组类型:64位平台下，数组类型本身大小为24字节，压缩后16字节

- 哪些信息不会被压缩？
  1.指向非Heap的对象指针
  2.局部变量、传参、返回值、NULL指针

### 10. class文件加载

在实际使用中，我们当前线程的类加载默认为AppClassloader。如果我们自定义了一个类加载器从外部的jar包或其它途径读取了一组字节流加载为对应的类，由于当前默认的类加载器对这个类是不做负责的，导致我们无法直接的在代码中用普通的类形式调用相关的方法，只能通过类的newInstance()方法生成一个实例，而实例也只能通过getMethod()方法调用指定的方法，invoke读取结果中的对应数据。

可使用当前线程即Thread.currentThread().setContextClassloader()来设置当前需要的类加载器，此时就可以像正常操作那样调用类并使用，但最后作为临时操作【可放在某个方法中，临时改变方便自己的操作，最后该还原】。

#### 10.1 自定义的类加载器

下面是摘自官方文档的一些介绍：

> 支持并发加载类的类加载器称为*并行能力*类加载器，需要通过调用ClassLoader.registerAsParallelCapable方法在其类初始化时间注册自身【默认情况下， `ClassLoader`类注册为并行】（它的子类仍然需要注册自己，如果它们是并行的能力）。
> 在委托模式不是严格层次化的环境中，类加载器需要并行，否则加载类可能导致死锁，因为加载程序锁定在类加载过程中保持（参见[`loadClass`](java/java/lang/../../java/lang/ClassLoader.html#loadClass-java.lang.String-)方法）。
> 
>  方法[`defineClass`](java/java/lang/../../java/lang/ClassLoader.html#defineClass-java.lang.String-byte:A-int-int-)将字节数组转换为类别`类`的实例。 这个新定义的类的实例可以使用[`Class.newInstance`](java/java/lang/../../java/lang/Class.html#newInstance--)创建。
> 
> 类加载器创建的对象的方法和构造函数可以引用其他类。 要确定所引用的类，Java虚拟机调用最初创建该类的类加载器的[`loadClass`](java/java/lang/../../java/lang/ClassLoader.html#loadClass-java.lang.String-)方法。
> 
> 网络类加载器子类必须定义从网络加载类的方法[`findClass`](java/java/lang/../../java/lang/ClassLoader.html#findClass-java.lang.String-)和`loadClassData` 。 一旦下载构成类的字节，它应该使用方法[`defineClass`](java/java/lang/../../java/lang/ClassLoader.html#defineClass-byte:A-int-int-)创建一个类实例。 

上述的官方描述，已经说明了一个类加载中主要的方法，因此我们在自定义时只需要覆盖这些方法即可，主要的步骤包括：

1. 在findClass()中指明类的来源，使之能够读入字节流。这里的操作空间就非常大了，可以从任意地方的任意位置的任意文件中获取流，只要是能够执行的。例如，可以放在某个服务器上的某个目录下的txt文件，只要这个文件磁盘上的内容是class文件的形式即可。这样我们可以随便定义目标文件的后缀名让人无法察觉。
2. 在获取了字节流之后，findClass()方法内部需要调用一个defineClass()方法从字节流生成一个类。这也可以自主发挥一下如何搞些奇怪的手段改变字节流搞事。

大致的效果是：

自定义一个类加载器，

```java
public class MyClassLoader extends ClassLoader
{
    //省略

    protected Class<?> findClass(String name) throws ClassNotFoundException
    {
         //这里如果继承自URLClassLoader，可以使用url读取网络或本地位置的文件
        File file = new File("目录"+文件name);
        try{
            byte[] bytes = getMyClassBytes(file);//读取字节流，可自定义读取方法

            Class<?> c = this.defineClass(name, bytes, 0, bytes.length)//生成类
            return c;
        } 
        catch (Exception e)
        {
            e.printStackTrace();
        }
       //省略
    }
}
```

调用类加载器，

```java
MyClassLoader myClassLoader = new MyClassLoader(); //我们的类加载器
Class<?> clazz = Class.forName("类名", true, myClassLoader); 
Object obj = clazz.newInstance();//生成一个原始的类对象
//因为默认发类加载器中并不负责这个类，因此一旦脱离的自定义的类加载器，无法给他一个合适的名分，只能作为纯粹的对象，不能直接调用它原有的各种技能
```

另外，如果定义的类的全限定名与上层的类加载器有冲突，不希望被双亲委派机制交由上级处理，可以覆盖loadClass方法指明当对象为null时，由自定义的加载器负责创建。

```java
protected Class<?> loadClass(String name, boolean resolve)//name为类的全限定名
    throws ClassNotFoundException
{
    synchronized (getClassLoadingLock(name)) {

        Class<?> c = findLoadedClass(name);//查看缓存中是否已存在对应的类
        if (c == null) {
            /*try {
                if (parent != null) {
                    c = parent.loadClass(name, false);
                } else {
                    c = findBootstrapClassOrNull(name);
                }
            } catch (ClassNotFoundException e) {
                   e.printStackTrace();
            }*/
            //上述为常用的双亲委派机制
            //我们这里不希望委派
            if (c == null) {
                //直接调用自己的方法生成类
                c = findClass(name);
            }
        }
       //略
        return c;
    }
}
```

----------------------------

----------------------------------------==<font color=Blue>【疲倦的话，稍微缓一缓吧！】</font>==----------------------------------

---------------------------------

#### 10.2 类的验证

在获得了类的字节流后，首先需要判断这是不是一个合格的类文件。

验证操作主要有：格式验证、语义分析、操作验证，以及符号引用验证等。

- 格式验证主要检查字节码文件的字节流是否是符合class文件的格式要求。
  - 这一过程本质上是在在加载阶段就开始运行的。
- 语义验证是在上述验证结束后，字节流加载到方法区后执行。主要检验是否符合语法
- 操作验证则是对类的方法审核其不会导致崩溃等问题
- 符号引用验证，主要是对常量池中的各种符号引用执行验证（这一段实际是发生在解析阶段）【将常量池中所有的符号引用全部转换为直接引用】

#### 10.3 class文件加密与解密

由于Java生成的class文件非常方便就可以被反编译，一些重要的逻辑如果不希望被人看见，则必须要对class文件进行加密。当然，如果只是想稍微隐藏一下，也可利用前面提及的自定义类加载，让自己的class字节流隐藏在某个地方的某个文件中。【常见的反编译工具为**jd-gui**，当然Idea肯定也可以，VsCode安装个插件也没问题】

下面的内容主要来源于书本《Java虚拟机精讲》的7.3节。

首先，加密算法主要有对称加密和非对称加密。在需要双方互通信息时，主要需要非对称加密，用户将自己的公钥传输过去，对方利用公钥加密自己的内容，用户可以使用私钥解读。而对称加密则使用单一的密钥加密和解密，在传输加密信息时，必须双方均持有密钥，导致密钥分发过程造成不安全。但对于本地使用的文件而言，对称加密就足够了。【更多密码学知识，可参考《图解密码技术》】

该书中则使用了对称加密的3DES算法。

- [ ] 加密过程为：C=Ek3( Dk2( Ek1( P ) ) )。
- [ ] 解密过程为：P=Dk1( EK2( Dk3( C ) ) )。

Ek()和 Dk()代表 DES 算法的加密和解密过程，K 代表 DES 算法使用的密钥，P 代表明文，C 代表密文。

 具体的代码实现如下：

```java
import  javax.crypto.SecretKey;
import javax.crypto.Cipher;
import javax.crypto.spec.SecretKeySpec;
public class Use3DES {
    private static final String ALGORITHM = "DESede"; // 定义加密算法,
    //-------------加密-----------------------------
    public static byte[] encrypt(byte[] key, byte[] src) {
        byte[] value = null;
        try {
            SecretKey deskey = new SecretKeySpec(key, ALGORITHM);// 生成秘钥 key
            Cipher cipher = Cipher.getInstance(ALGORITHM);//对目标数据执行加密操作
            cipher.init(Cipher.ENCRYPT_MODE, deskey);
            value = cipher.doFinal(src);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return value;
    }
    //--------------解密-----------------------------
    public static byte[] decrypt(byte[] key, byte[] src) {
        byte[] value = null;
        try {
            SecretKey deskey = new SecretKeySpec(key, ALGORITHM);/* 生成秘钥 key */
            /* 对目标数据执行解密操作 */
            Cipher cipher = Cipher.getInstance(ALGORITHM);
            cipher.init(Cipher.DECRYPT_MODE, deskey);
            value = cipher.doFinal(src);
        } catch (Exception e) {
            e.printStackTrace();
        }
        return value;
    }

    public  static void main(String[] args) {
        try {
            byte[] key = "01234567899876543210abcd".getBytes();
            byte[] encoded = encrypt(key,
            "测试数据...".getBytes("utf-8"));
            System.out.println("加密后的数据->" + new String(encoded));
            System.out.println("解密后的数据->"
            + new String(decrypt(key, encoded), "utf-8"));
        } catch (Exception e) {
         e.printStackTrace();
        }
    }
}
```

上述代码的运行结果是：

​                                       ==加密后的数据->P湻9浪k0肟==
​                                        ==解密后的数据->测试数据...==

类似地，我们可以对class文件的字节流做同样的加密和解密操作，具体的代码放在附录14.1。

### 11. jvm优化

cmd 查看jvm调优参数:主要以—X或-XX开头的命令。

【可通过命令-XX:MaxTenuringThreshold配置指定次数】（Parallel Scanvenge 15,CMS 6,G1 15)

Parallel Scavenge 吞吐量参数

### 12. 多线程或多并发环境的JVM

此类情况下，由于面临当前操作可能会被其它进程或线程修改，因此重点在于使用锁技术，以及如何维护当前的工作。

#### 12.1 对象创建

根据前文的介绍，我们知道对象的创建主要的工作有：可能需要先进行类加载，再判断对象可以存放的位置，划分出指定的内存空间后，再加载自己的各种变量和方法，最后将各种引用关联起来，得到一个可以使用的对象。当然，最后需要将对象是引用与对应的对象名关联起来。

- 其中，在划分内存空间的时候，如果其它线程或进程也在试图使用内存，则会导致内存分配失败。例如Eden区使用的指针碰撞技术。

  - 指针碰撞通过简单的移动便划分出新的使用内存，为保证当前的移动不被干扰，操作内部使用了CAS原语。而CAS的底层实现则是利用汇编的lock命令，锁定总线，禁止其它命令进入【这里设计者偷懒，没有针对不同CPU的特定情况优化】。

- 对象名与实例连接：

  在单例模式下，我们只需要创建一个对象并重复使用即可，但是本节的情况下，如何保证仅创建了一个对象。

  大致的代码如下：

  ```java
  public class T {
      private static T  t=null;//准备好对象名
      //外部有大量的线程准备使用里面的对象
      //但此时还没创建出来
      if(t==null){//此时还是没有实例
          synchronized (T.class){//锁定，并进行创建
              if (t==null){
                  t=new T();
              }
          }
      }
  }
  ```

  上述流程为DCL(双重校验锁机制)，首先评估是否需要创建，避免直接进行锁竞争，获得锁之后，再次确认，避免重复创建。

  - 但这里存在一个问题，JVM会进行指令重排，在对象创建期间，应该是完成对象的彻底构建，才会将对象的引用交给对象名。但是指令重排，可能会提前将引用赋予对象名，导致其它线程发现已有对象并使用了不完整的实例。

  - 为此，需要额外加上`volatile`指令，即

    ```java
     private static volatile T  t=null;
    ```

    以禁止上述的指令重排序。

    valatile功能的实现则来自内存屏障：

    - Store：将处理器缓存的数据刷新到内存中。
    - Load：将内存存储的数据拷贝到处理器的缓存中。

    | 屏障类型   | 指令示例                 | 说明                                                         |
    | ---------- | ------------------------ | ------------------------------------------------------------ |
    | LoadLoad   | Load1;LoadLoad;Load2     | 确保Load1数据的装载先于Load2及其后所有装载指令的的操作       |
    | StoreStore | Store1;StoreStore;Store2 | 确保Store1立刻刷新数据到内存(使其对其他处理器可见)的操作先于Store2及其后所有存储指令的操作 |
    | LoadStore  | Load1;LoadStore;Store2   | 确保Load1的数据装载先于Store2及其后所有的存储指令刷新数据到内存的操作 |
    | StoreLoad  | Store1;StoreLoad;Load2   | 确保Store1立刻刷新数据到内存的操作先于Load2及其后所有装载装载指令的操作。它会使该屏障之前的所有内存访问指令(存储指令和访问指令)完成之后,才执行该屏障之后的内存访问指令 |

    为了实现volatile的内存语义，编译器在生成字节码时，会在指令序列中插入内存屏障来禁止特定类型的处理器重排序。在每个volatile写操作的前面插入一个StoreStore屏障，确保前面的写操作完成。在每个volatile写操作的后面插入一个StoreLoad屏障，确保自己的写操作完成。

- 此外为了实现单例模式的安全性，可以使用holder模式，即插入静态内部类，以避免竞争。

  ```java
  public class Singleton {
      /**
       * 类级内部类，也就是静态的成员内部类，该内部类的实例与外部类的实例没有绑定关系
       * 只有被调用的时候才会装载，从而实现了延迟加载，即懒汉式
       */
      private Singleton() {
  
      }
  
      private static class SingletonHolder {
          /**
           * 静态初始化器，由JVM来保证线程安全
           */
          public static final Singleton INSTANCE = new Singleton();
      }
  
      public static Singleton getInstance() {
          return SingletonHolder.INSTANCE;
      }
  }
  ```

  

#### 12.2 垃圾回收

垃圾回收器在CMS之后，都开始了并发标记，即在用户线程运行期间对系统中的对象实例进行垃圾标记。这期间可能发生某些垃圾被重新引用，因此又需要进行结果修正。

而如何与用户线程共存且完成标记的任务则包含了许多思考。

- 卡表：前文中就提及了卡表是维护区域之间引用关系的。但如何在对象被引用后，就将对应的卡表标记为变脏，就需要写屏障来工作。
  - 写屏障其实就是类似spring中的AOP编程（面向切面编程）。引用对象是切点，写屏障就是在这个切点前后增加语句执行额外的操作，以此完成卡表的标记工作。
  - -XX:+UseCondCardMark命令可开启有条件的写屏障，即对应的卡页已经是脏的，就不去再标记了。
  - 伪共享，还是关于CPU缓存行的问题，如果多个卡表元素落入同一个缓存行中，不同的线程修改各自的元素，线程之间需要频繁第争夺缓存行的使用权。通过有条件的写屏障一定程度上避免了这种争夺。
    - 而类似之前一节有说过的，我们可以主动填充缓存行使其立即刷新，这里也可以人为地增加多余的成分占据一个缓存行，使得不同线程对应的卡表元素处于不同的缓存行。

在利用卡表标记了对象之间存在引用关系之后，借助卡表就可以快速地对引用链进行遍历。但由于此时用户进程也在继续，则需要三色标记来判断对象的存活状态。

- 三色标记

  扫描是自上而下不断递归深入地进行。利用不同颜色标记对应的对象状况，如果是黑色，则认为上下均完成扫描，不会再从这个对象开始递归扫描。灰色代表自己被扫描，但是自己引用的对象未全部完成扫描。白色即代表那些未被扫描过的，最后指示为不可达，即被当作是垃圾对象。

```mermaid
graph TD
	id0(<font color=white>上下均已扫描</font>)-->id1
	id1(<font color=white>上下均已扫描</font>)-->id2(<font color=white>下面未完成扫描</font>)
	id2-->duli(<font color=white>上下均已扫描</font>)
	id2-->hui(<font color=white>下面未完成扫描</font>)
    style id1 fill:#000000
    style id0 fill:#000000
    style id2 fill:#9b9b9b,stroke:#9b9b9b
    style hui fill:#9b9b9b,stroke:#9b9b9b
    style duli fill:#000000
    id2-->id3(自己未被扫描)
    style id3 fill:#ffffff
    style id4 fill:#ffffff
    id0--状况1:新插入-->id4(自己未被扫描)
    id2-.状况2:删除.->id4
```

若同时发生了上图中的状况，最后都会导致某些存活的对象被视为垃圾。

- 状况1：黑色对象之后突然引用了一个独立的白色对象，由于这个白色对象不被其它非黑色对象引用，导致垃圾回收器之后都不会扫描到这个对象，致使该对象被视为垃圾。
- 状况2：如果白色对象与所属的灰色对象直接的可达路径完全被切断，则导致之后从灰色对象向下递归扫将无法扫描到这些对象，本来这样没什么问题，但是如果这之前这些对象被黑色对象引用了，则回到了状况1的情况，同样会导致存活对象被当作垃圾。

为避免对象误判为垃圾：

**CMS**：采用增量更新

- 即当黑色对象添加了指向白色对象的引用，则将这个新引用记录下来（相当于将黑色对象重新标记为灰色），在完成了并发标记后，将从这些黑色对象再做扫描。

**G1**：采用原始快照

- 相比于CMS中记录黑色对象添加的引用，这里将记录那些灰色对象删除的引用，当完成并发标记后，再对这些记录进行扫描。

上述两种方式，读者会发现，都发生了一旦引用变化就会立即记录，这就是再次利用了写屏障的技术。

### 13. 其它的类加载机制

#### 13.1 Tomcat

作为一个Web应用，Tomcat需要管理多个不同的网站，即对应的不同的jar/war包，导致Tomcat内部包含了多个应用类库。常规的委派机制无法解决类与类的全限定名的冲突，故，Tomcat内部支持委派机制，但有自己的一套加载系统。

Tomcat的类加载器关系如下：【版本为Tomcat7之后】

```mermaid
graph BT
System --> Bootstrap
Common --> System
Catalina --> Common
Shared --> Common
WebApp0 --> Shared
WebApp1 --> Shared
```

其中，WebApp0和WebApp1是随着用户自定的jar包而创建，用于加载对应目录下所有class，资源，jar文件，即负责加载指定Web的资源，且只对该Web可见。

下面是关于上图中各类加载器的解释，具体细节可查看[官方文档]([Apache Tomcat 10 (10.0.6) - Class Loader How-To](http://tomcat.apache.org/tomcat-10.0-doc/class-loader-howto.html))

- BootStrap：加载JVM提供的基本运行类，加之目录`JAVA_HOME/jre/lib/ext`下的所有jar包中的类。
- System：通常从CLASSPATH环境变量的内容初始化该类加载器。但实际上是加载 Tomcat 启动脚本（$CATALINA_HOME/bin/catalina.sh |bat）指定位置的类。【所有这些类对 Tomcat 内部类和 Web 应用程序都是可见的】
- Common：加载Tomcat以及应用通用类，位于CATALINA/lib目录下。是一个公用类加载器。
- Catalina：用于加载Tomcat应用服务器的类加载器（路径为server.loader）。【默认路径为空，此时由Common加载应用服务器】
- Shared：是Web应用的父加载器（路径为shared.loader）。【默认路径为空，此时由Common作为Web应用的父加载器】

<font color=Red>**Tomcat中类的实际加载顺序**</font>，需要考虑类加载器的`delegate`属性值，【默认`false`】（通过在对应的conf/Context.xml中\<Loader delegate="true" /\> 进行修改。

将依此从下述位置中查找类或资源，

- false
  
  1. 缓存
  2. JVM的Bootstrap
  3. */WEB-INF/classes* 目录
  4. */WEB-INF/lib/\*.jar* 
  5. System class loader
  6. Common class loader

- true，此时将使用java默认的委派机制
  
  1. 缓存
  2. JVM的Bootstrap
  3. System class loader
  4. Common  class loader
  5. */WEB-INF/classes* 目录
  6. */WEB-INF/lib/\*.jar*
  
  由于这里的类初次加载时无论怎么样，都需要先经过BootstrapClassLoader（且，这里包含了也包含了扩展类），因此恶意创建的JDK基类，是无法加载的。

#### 13.2 字节码生成

这部分就非常简单了，就是动态代理，既包含了JDK自带的Proxy类，也包含CGLib等，都属于根据已有的条件额外创建了一个class文件，也属于一种类加载机制。

#### 13.3  模块化

JDK9才引入了JPMS模块系统，还是静态的。而之前已经有了OSGi(Open Service Gateway Initiative)【动态模块化规范】，其中的模块一般就是已jar包的形式封装，用以实现热插拔功能。

OSGi的Bundle类加载器
之间只有规则，没有固定的委派关系。例如，某个Bundle声明了一个它依赖的Package，如果有其他
Bundle声明了发布这个Package后，那么所有对这个Package的类加载动作都会委派给发布它的Bundle类
加载器去完成。不涉及某个具体的Package时，各个Bundle加载器都是平级的关系，只有具体使用到某
个Package和Class的时候，才会根据Package导入导出定义来构造Bundle间的委派和依赖。
另外，一个Bundle类加载器为其他Bundle提供服务时，会根据Export-Package列表严格控制访问范
围。如果一个类存在于Bundle的类库中但是没有被Export，那么这个Bundle的类加载器能找到这个类，
但不会提供给其他Bundle使用，而且OSGi框架也不会把其他Bundle的类加载请求分配给这个Bundle来
处理。

### 14. <font color=Red>附录</font>

#### 14.1 <font color=red>class文件3DES加密和解密（自定义的类加载器）</font>

```java
import java.io.FilterInputStream;
import java.lang.ClassLoader;
import java.lang.ClassNotFoundException;
import java.io.BufferedInputStream;
import java.io.IOException;
import java.io.BufferedOutputStream;
import java.io.FileOutputStream;
public class MyClassLoader extends ClassLoader {
    private String byteCode_Path;
    private byte[] key;
    public MyClassLoader(String byteCode_Path, byte[] key) {
        this.byteCode_Path = byteCode_Path;this.key = key;}
    @Override
    protected Class<?> findClass(String className) throws ClassNotFoundException {
        byte value[] = null;
        BufferedInputStream in = null;
        try {
            in = new BufferedInputStream(new FileInputStream(byteCode_Path + className +                                                                             ".class"));
            value = new byte[in.available()];
            in.read(value);
        } catch (IOException e) {e.printStackTrace();} finally {if (null != in) {
                try {in.close();} catch (IOException e) {e.printStackTrace();}}
        }
        value = Use3DES.decrypt(key, value);//解密,自定义的关于3DES类
        return defineClass(value, 0, value.length);//将 byte 数组转换为一个类的 Class 对象实例 
    }
    public static void main(String[] args) {
        BufferedInputStream in = null;
        try {
            in = new BufferedInputStream(new FileInputStream("class文件地址"));
            byte[] src = new byte[in.available()]; in.read(src); in.close();
            byte[] key = "01234567899876543210abcd".getBytes();//设置密钥
            BufferedOutputStream out = new BufferedOutputStream(new                                                                 FileOutputStream("加密后的文件地址"));
            out.write(Use3DES.encrypt(key, src));//解密并输出
            out.close();
            MyClassLoader classLoader = new MyClassLoader("文件上级目录",key);
              System.out.println(classLoader.loadClass("对应的类名").getClassLoader().
                                                                   getClass().getName());
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```



#### 14.2 <font color=red>OOP-Klass模型细节</font>

OOP



    CMS 三色标记<1.SATB 2.Incremental Update
    ZGC 颜色指针
    
    最厉害的垃圾回收器 zulu 的c4


​    jdk默认回收器为ps/po



c++为了实现多态特性，使用了动态绑定技术。核心是虚函数表。

一个类如果包含了虚函数，则需要为自己准备一个虚函数表，表的元素是虚函数的的函数指针【在编译期间完成表的建立】（虚表是一个类拥有的，相当于是静态的】

类的对象在创建之后，，也会被分配一个指向虚表的指针。

三色标记：错标，增量更新，重标

G1：三色标记，SATB











