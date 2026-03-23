---
layout: post
title: "系统的启动和引导"
modified:
categories: academic
excerpt:
tags: ["翻译", "内存系列"]
---

## CPU引导系统的过程

当你按下计算机的电源键后，机器就开始运转了。一旦主板上电，它就会初始化自身的固件(firmware)——芯片组和其他零零碎碎的东西——并尝试启动CPU。如果此时出了什么问题，比如CPU坏了或根本没装，大多数情况下机器都会处于僵死状态。

如果一切正常，CPU就开始运行了。在一个多处理器或多核处理器的系统中，会有一个CPU被动态的指派为引导处理器（bootstrap processor，简写为BSP），用于执行全部的BIOS和内核初始化代码。其余的处理器，此时被称为应用处理器（application processor，简写为AP），一直保持停机状态直到内核明确激活他们为止。这个过程中，CPU工作于实模式，分页功能是无效的，只有1MB内存可以寻址。任何代码都可以读写任何地址的内存，这里没有保护或特权级的概念。

CPU上电后，大部分寄存器都被初始化，包括指令指针寄存器（EIP）。它记录了下一条即将被CPU执行的指令所在的内存地址。通过将EIP加上一个隐藏的基地址（其实就是个偏移量），使CPU首先执行0xFFFFFFF0（长16字节，在4GB内存空间的尾部，远高于1MB）的指令。这个特殊的地址叫做复位向量(reset vector)，而且是现代Intel CPU的标准。

主板保证在复位向量处的指令是一个跳转，它跳转到BIOS执行入口点所在的内存映射地址。芯片组提供的内存映射功能将BIOS闪存映射到内存896KB~1MB的位置，而此时的RAM模块还只有随机的垃圾数据。

随后，CPU执行BIOS代码，初始化硬件，然后BIOS执行上电自检过程（POST），检测/初始化计算机中的各种组件：显卡、内存、键盘等。如果缺少，POST就会失败，导致BIOS进入停机状态并发出鸣音提示。此外，枚举出所有PCI设备的资源——中断，内存范围，I/O端口。现代的BIOS会遵循高级配置与电源接口（ACPI）协议，创建一些用于描述设备的数据表，这些表格将来会被操作系统内核用到。

POST完毕后，BIOS就准备引导操作系统了，它必须存在于某个地方：硬盘，光驱，软盘等。如果找不到合适的引导设备，BIOS会显示出错信息并停机，比如“Non-System Disk or Disk Error”没有系统盘或驱动器故障。

BIOS会读取硬盘的第一个扇区（0扇区），内含512个字节，这些数据叫做主引导记录，Master Boot Record，简称MBR。一般说来，它包含两个极其重要的部分：一个是位于MBR开头的操作系统相关的引导程序，另一个是紧跟其后的磁盘分区表。但BIOS简单的加载MBR的内容到内存地址`0x7C00`处，并跳转到此处开始执行，不管MBR里的代码是什么。

* 如果是Windows的MBR，则查看分区表，找到一个（唯一的）标记为活动（active）的分区，加载那个分区的引导扇区（boot sector，该分区的第一个扇区），并执行其中的代码。
* 如果是Linux之类的MBR，则先执行MBR本身的引导程序，然后从磁盘加载另一个含有额外的引导代码的扇区（可能是引导扇区，也可以特别指定），再读取引导文件，如Windows Server中的C:/NTLDR。最后，读取OS的内核文件，如Linux中的vmlinuz-2.6.22-14-server之类的内核镜像文件。

值得一提的是，由于内核镜像会比较大（>1M），超出了实模式下寻址空间，所以需要MBR反复在实模式和保护模式之间转化。待装载完之后，CPU仍在实模式下。即，此时的RAM中，除了少部分内核镜像位于1M之上的一段空间，其余有意义的空间均在1M之下，如Boot command line，Real-mode kernel header等。

## 内核的引导

以Linux为例，其内核引导过程始于Real-mode kernel header。它首先执行arch/x86/boot/header.S中的程序，该程序包含了引导扇区代码，以及Linux内核的第一条指令，也就是实模式内核的入口点。它是一个2字节的跳转指令，跳转到header.S中的start_of_setup函数处，执行栈的初始化工作，然后跳转到arch/x86/boot/main.c。

main()会处理一些登记工作，然后调用go_to_protected_mode()。后者有两件重要的任务：

1. 调用setup_idt()。将实模式的中断向量表（从内存地址0开始）地址保存在CPU的IDTR寄存器中
2. 调用setup_gdt()。保护模式下的逻辑地址需要全局描述符表进行转换，这个表的地址要放到GDTR寄存器里。

之后，go_to_protected_mode()调用protected_mode_jump汇编子程序设定CPU的CR0寄存器的PE位，打开保护模式。该子程序进而调用压缩状态内核的32位内核入口点startup_32。startup32会做一些简单的寄存器初始化工作，并调用一个C语言编写的函数decompress_kernel()，用于实际的解压缩工作。

解压缩工作会打印一条大家熟悉的信息“Decompressing Linux…”。解压后的内核从1MB位置开始的。解压完成之后，会显示“done”和“Booting the kernel”（正在引导内核）。然后跳转到保护模式内核的入口点，位于RAM的第二个1MB开始处，执行另一个名叫startup_32的代码。这段代码初始化保护模式下内存的各项寻址、转换、功能部件，以及中断描述符表，并最后调用start_kernel()，启动第一个进程，PID0。它启动进程kernel_init，并将自己睡眠，直至被唤醒或关机。

kernel_init()负责引导剩下的应用处理器，并使之进入保护模式。最终，它尝试启动一个用户模式（user-mode）的进程，尝试的顺序为：/sbin/init，/etc/init，/bin/init，/bin/sh，于是1号进程运行了。这些init程序包括了各种系统启动时需要加载的程序，如窗口管理器、登陆程序等。

Windows的启动过程和Linux类似，但一个最大的不同是，Windows把全部的实模式内核代码以及一部分初始的保护模式代码都打包到了引导加载程序（C：/NTLDR）当中。于是Windows用户模式的启动就非常不同了。没有/sbin/init程序，而是运行Csrss.exe和Winlogon.exe。Winlogon会启动Services.exe（它会启动所有的Windows服务程序）、Lsass.exe和本地安全认证子系统。经典的Windows登陆对话框就是运行在Winlogon的上下文中的。

