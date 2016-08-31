---
layout: post
title: "How to Make ASLR Win the Clone Wars: Runtime Re-Randomization (2016)"
categories: notes
tags: ALSR 内存保护 内存错误 控制流完整性 NDSS 
---

> 原文：[How to Make ASLR Win the Clone Wars: Runtime Re-Randomization][src] 

[src]: http://www.cc.gatech.edu/~klu38/publications/runtimeaslr-ndss16.pdf

## 为什么要做
现有的方案大多在进程装载（load-time）前进行一次性地地址随机化，这背后的假设是对进程的攻击失败之后进程会重新加载，这样就会重新执行随机化。然而，对于守护进程来说，受攻击而崩溃之后是fork新的子进程而不是重新加载，也因此继承了父进程的地址布局，使得攻击者可以多次对同一内存布局进行多次猜测和攻击（称为clone-probing攻击）。

在fork()之后执行execve()也可以使得fork出的子进程重新执行装载时的地址随机化，但这样就破坏了fork()的本意：父子进程间共享系统资源，如打开的文件、网络端口等，从而节省装载新进程的开销。对于守护进程，这样做还意味要小心重构子进程继承下来的资源（文件、共享内存、全局变量……）。


## 做了什么
* 终极目标：设计了一个RuntimeASLR，不需要修改binary代码即可进行随机化，可阻止clone-probing攻击，并保留父进程的语义（不需要在fork之后重新构造父进程打开的资源）。
* 为达到终极目标用的先进技术：完整的指针跟踪算法，用于标记、追踪将被随机化的指针。
* 开源，并对Nginx做测试。

## 怎么做到的
![Overview of RuntimeASLR's architecture](/images/runtime-aslr.png)
1. Taint Policy Generation：确定binary中和指针有关的指令，以及这些指令如何影响污点指针（Taint Policy）。如果寄存器或者内存里的值指向当前内存映射地址区间，则认为是指针。这种方法有误报，但很低，尤其是多次独立运行后对每次的结果都进行对比的情况下（文中有理论计算和实验佐证）。
2. Pointer Tracking：以（程序启动时）OS初始化的指针等作为污点源，如：rsp、PC指针，以及syscall的返回值等。然后根据上一步策略做污点传播分析，获得所有需要随机化的指针的列表。
3. Address Space Re-randomization：以模块为粒度对上一步得到的指针做随机化。

前两步用Pin进行动态插桩获得所需信息，第三步在子进程fork之后Pin就退出。

### Taint Policy Generation
当前跟踪技术的主要问题是跟踪的不完整，而这里不能有误报漏报，不然程序就可能崩溃。所以利用Pin对引用寄存器和内存的指令插桩，观察他们对指针的影响。此外，对创建指针的syscall加钩子，观察哪些内存是合法的。（需要手工分析syscall的行为。）

为了验证跟踪时记录的是指针而不是数据，跑多次执行程序并跟踪，提取出重合的记录。原理是数值每次大小都落在合法内存区域的概率非常小。

### Pointer Tracking
源点标记寄存器和内存的污染状况用hash表保存，供第三步实用。

### Address Space Re-randomization
为了保证进程执行效率，把随机化的代码做成动态库，在fork子进程之后、把Pin从进程中剥离（fork会把Pin占用的上下文也复制过来）时调用，修改所有指针的指向地址，清楚Pin的残留代码和数据，完成对子进程地址随机化的过程（利用系统调用`mremap(rand\_num)`），并切换上下文。

## 效果如何
验证了指针识别方法的正确性、地址随机化的有效性和运行效率。结果表明可以识别目前最全的指针；随机化之后的有问题Nginx版本可以抵御公开的BROP攻击；在进程第一次启动时启动时间明显变长，但之后对进程的影响很小。

## 相关工作
### 指针跟踪
一般分两种方法：静态的基于类型的分析，和基于启发式的分析。前者容易误报，后者则很难推理出覆盖所有情况的污点传播规则，也就是漏报比较多。

### 随机化
和本文类似的工作有Morula，为Android进程做地址随机化。它的思路是一个维护一个Zygote进程池，池子里的每个Zygote进程都被随机化了。Zygote进程的作用是作为每个App的初始化进程，所以每次App启动时，都从池子里挑一个Zygote进程初始化App。

ASR等在装载时进行随机化，所以会丢失进程上下文的语义（点解？）。Isomeron和DSD的思路是动态地选择间接跳转的目标，以降低被ROP猜中地址的可能性，这对clone-probing攻击没用。TASR在编译时随机化代码段，所以不能保护数据指针。

-------- copied from  http://www.inforsec.org/wp/?p=1009 --------
随机化的粒度可以改进。粒度小了，熵值增加，就很难猜出ROP gadget之类的内存块在哪里。ASLP在函数级进行随机化，(binary stirring在basic block级进行随机化，ILR和IPR在指令级。前者将指令地址进行随机化；而后者把指令串进行重写，来替换成同样长度，并且相同语义的指令串。

随机化的方式可以改进。Oxymoron解决了库函数随机化的重复问题： 原先假如每个进程的library都进行fine-grained的ASLR，会导致memory开销很大。该文用了x86的segmentation巧妙地解决了这个问题；并且由于其分段特性，JIT-ROP之类的攻击也很难有效读取足够多的memory。Isomeron利用两份differently structured but semantically identical的程序copy，在ret的时候来随机化execution path，随机决定跳到哪个程序copy，有极大的概率可以让JIT-ROP攻击无效。

Remix提出了一种在运行时细粒度随机化的方法。该方法以basic block为单位，经过一个随机的时间对进程(或kernel module)本身进行一次随机化。由于函数指针很难完全确认(比如被转换成数据，或者是union类型)，该方法只打乱函数内部的basic blocks。该方法另一个好处是保留了代码块的局部性(locality)，因为被打乱的basic blocks位置都很靠近。打乱后，需要update指令，以及指向basic block的指针，来让程序继续正确运行。假如需要增加更多的熵值，可以在basic blocks之间插入更多的NOP指令(或者别的garbage data)。

另一种方法（CCS2015）是用编译器来帮助定位要migrate的内存位置(指针)，并且在每次有输出的时候进行动态随机化。该方法对于网络应用比如服务器，由于其是I/O-intensive的应用，可能会导致随机化间隔极短而性能开销巨大。

-------- end of copy --------




还有很多工作加强了程序的CFI，以应对ASLR被旁路后的攻击，这和本文工作是正交的。

## 然后呢

没了。