---
layout: post
title: "Enhancing symbolic execution with veritesting
(2014)"
categories: notes
tags: 漏洞检测 程序分析 符号执行 binary分析 ICSE 
---

> 原文：[Enhancing symbolic execution with veritesting][src] 

[src]: http://dl.acm.org/citation.cfm?id=2568293

动态符号执行（DSE）和静态符号执行（SSE）一个为路径生成公式，一个为语句生成公式。前者生成公式时会产生很高的负载，但生成的公式很容易解；后者生成公式很容易，公式也能覆盖更多的路径，但是公式更长更难解。方法上的区别在于DSE会摘要路径汇合点上两条分支的情况，而SSE为两条分支fork两条独立的执行路径。

SSE目前还不能对大规模的程序分析（如Cloud9+state merging），问题主要在于循环的表示、方程复杂度、缺少具体状态、和对syscall等的模拟。Veritesting可以在SSE和DSE之间切换，减少负载和公式求解难度，并解决静态方法需要摘要或其他方法才能处理的系统调用和间接跳转。之前的工作Mayhem是DSE。

提出了MergePoint工具，可以对32bit的Linux二进制文件实施Veritesting。实验对象包括几个测试集和Debian发行版。采用了三个标准判断实验的好坏：发现的bug数，节点覆盖率和路径覆盖率。(SSE的引入应该能提高覆盖率?)

## 方法
从DSE开始，接受具体输入，探索输入对应的路径以及附近的路径。在合适的时候切换到SSE，然后再切回DSE。只对程序片段做SSE，而且只做测试，而不是验证。DSE在遇到分支时，会将两个分支都放到工作队列中供以后执行，这个过程叫fork（这样生成的约束通常求解时间不超过1秒）。而Veritesting在遇到符号分支时，观察分支点后面的代码是否含有syscall和间接调用或其他难以静态推理的语句，如果没有，则切入SSE。SSE工作在DSE生成的CFG上，它会找到合适它分析的代码，并标识出无法继续分析的边界，目的是摘要这段代码的所有路径。之后切换回DSE。具体有四步：

1. 从分支点开始恢复CFG，到函数边界、syscall或者不认识的指令时停止。这是个过程内的cfg，没有被解析的跳转目标都转向cfg的exit点。
2. CFG化简消环；并找出transition point，即SSE结束的地方。transition point指cfg的边界点，它可能是未解析的跳转、函数边界或syscall（之前的那个节点）。由于此时还是DSE，所以可以确定cfg中环（循环）的执行次数，也可以用户指定。消环的办法是将回路取消，每循环一次就指向一个新创造的节点。最后的cfg每条边上都附注了到达当前路径的条件。
3. 对化简过的cfg做SSE，得到其中所有的可达路径。实际是单次的gated single assigment的数据流分析（1995年的技术）。为了防止cfg过大（overapproximation），计算DSE和SSE约束的合集，如果可达才留下对应的路径。
4. 收尾，根据transition point和SSE的状态，计算需要DSE继续fork的路径。检查transition point 是否在可达路径上，如果在，则创建一个DSE执行器（就是之后肯定会被DSE执行）。DSE执行的路径对应测试用例，目前只保证为每个被SSE分析的cfg中的节点生成一个测试用例，不然测试集太大。也可以改成为cfg的每条边生成一个用例。为了防止cfg过小漏掉路径（underapproximation），对cfg的exit节点的约束（即所有forked状态的路径约束的并集）取反求解，如果可满足，则为对应路径fork一个状态。

Veritesting是online的，和程序执行并行进行的。如果算法失败了，则切回DSE，直到遇到下一个符号分支点。

## 实现

先对binary插桩，用污点分析确保只在和输入相关的指令上执行DSE，然后将x86指令即时编译（JIT）为IR，最后符号执行。MergePoint接受程序、用户配置（时间限制、程序输入等），输出测试用例、bug和统计信息。

使用Hash consing技术避免替换符号值时的重复计算，将形如s+s+s+s+s=42的表达式优化为5*s=42。类似技术在其他SE工具中也有应用。但这里把hash consing做到了IR里，这样就不会出现重复的表达式了。

采用分布式结构建在有100个节点的虚拟云上，并行执行（多个）程序。基于之前研发的[Mayhem工具](mayhem.html)，17000行OCaml，9000行C，3500行Erlang（用于任务分发）。插桩工具用PIN，SMT求解器用Z3

## 实验

三组benchmark：GNU coreutils (de facto standard for evaluating bug-finding systems since KLEE)，BIN suite（1023 programs），Debian（33248 programs）。大多数类似的工作都用了第一组。每个程序设定运行时间，每组时间15分钟到1小时不等。每组先用DSE做一遍，再加上Veritesting做一遍。

1. 找到了其他SE工具都没找到的bug，其中一个隐藏了9年之久。
2. 点覆盖率大部分比别的工具高，小部分低于S2E，因为这部分程序使用的socket系统调用还没法处理。
3. 路径覆盖率完成度和DSE一样，但是更快

对Debian的检查采用了统一的输入规范（符号参数个数、大小，以及符号文件输入的大小）。分别用不同的文件输入大小检查了两次，一次23byte，一次2100byte检查缓冲区溢出。出错以程序crash为准。发现了200万个crash，分成了1万多个bug，其中152个被证明可利用
