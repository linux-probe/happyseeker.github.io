---
layout: post
title:  "Mips TLB miss异常"
date:   2016-12-27 16:15:21
author: HappySeeker
categories: Kernel
---

最近分析龙芯KVM的实现，顺便又粗看了遍MIPS的手册，跟KVM相关的主要模块包括：

1. CPU虚拟化
2. 内存虚拟化
3. IO虚拟化

目前龙芯上CPU虚拟化跟标准内核差异不大，需要软硬件配合支持，目前龙芯整体能支持。
内核虚拟化是龙芯KVM方案的关键，直接决定了性能，这也是本文的源头。
IO虚拟化，目前龙芯由于没有自己的桥片，很难做什么，主要基于KVM中现有的virtio(半虚拟化)，这里也不关注。

# 结构化TLB

龙芯的内存管理机制，最核心的莫过于MIPS中**结构化TLB**的相关处理。

## MIPS的MMU架构

通常，处理器的MMU架构分两类：

1. 结构化页表。这也是X86和PowerPC使用的方式。就是我们熟悉的：基于页表来做虚拟地址到物理地址的映射，当然，这种方式下，也存在TLB，但仅作为页表的缓存。地址翻译过程描述为：当CPU需要做访存操作时，MMU先在TLB中查询相应条目，如果没找到，就从页表中查找。如果页表中找到，则填充TLB，如果没找到，则触发缺页异常。这一整套操作都由MMU硬件完成。

2. 结构化TLB。这是MIPS使用的方式。核心思想是：所有的地址翻译都经过TLB完成。结构化TLB中也存在页表，但这基本由软件(操作系统内核)使用和维护；地址翻译过程描述为：当CPU需要做访存操作时，MMU从TLB中查找，当TLB中没找到时，直接触发TLB Miss(也叫TLB Refill)异常，TLB Miss异常处理中(内核实现)，完成TLB的填充，通常的实现为：也维护X86中类似的页表，从页表中取相应的条目，填充到TLB中，其中还涉及更复杂的二次异常的逻辑，后面单独介绍。TLB的填充操作完全由软件负责，当然，硬件层面有一些辅助，用于提升性能，后面单独介绍。如此设计，利弊明显。

  - 好处：硬件设计简单，软件实现灵活性大。
  - 坏处：性能可能不足(还是要分场景)？

## TLB Miss中Page Fault

如前所述，结构化TLB中，当TLB中不存在映射时，从页表中找。那么，当页表中也不存在，此时该怎么办？换句话说，页表中的条目是从哪里来的？显然硬件不会自动生成。

这里需要明确：

- MIPS硬件实现上，没有X86中对应的**缺页异常**。
- MIPS MMU硬件不会去查页表，只会查TLB
- 页表内容显然需要软件来维护。

### MIPS Exception

MIPS中的异常(Exception，翻译可能有不同)分为两类：

1. 高优先级。即需要尽快处理的异常，每种异常都有一个专用的固定的异常入口，因此可以得到快速处理。包含3种：
  - NMI、启动、重启
  - TLB Refill，即TLB Miss异常。
  - Cache Error

2. 普通优先级。即General Exception，也称**其它异常**，即除上述高优先级异常之外的所有异常(由CP0的STATUS寄存器[6:2]区分，也称ExcCode)，都归属于此类，如中断、TLB Load/Store/Modify异常、系统调用等，此类异常对应同一个异常入口地址(处理接口)，然后在该处理接口中根据不同的异常类型，调用不同的处理接口，相当于将事件进行了二次转发处理，所以，相对于前面的高优先级异常，处理较慢。

多数情形下（非嵌套），异常处理其实就相当于一个伴随模式切换的过程调用。异常的处理流程描述为：

1. 异常发生后，MIPS置CP0寄存器Status[EXL] = 1，并将当前PC值存入EPC(Exception PC，即指向发生异常的指令)。

2. 从当前指令流跳转到异常处理接口。

3. MIPS硬件不负责保存上下文，因此需要软件先保存上下文。

4. 异常处理完成后，同样由软件负责恢复上下文，再执行指令**eret**，从异常返回。

5. eret所作的操作为：将EPC/ErrorEPC的值置入PC，同时清除CP0_Status寄存器的EXL位。MIPS下，当Status寄存器的EXL(exception)位为1，即表示处理器进入异常处理，处于特权模式下。


需要注意：当在异常处理过程中(Status[EXL] = 1)，又出现新异常时，CPU将不会重新设置EPC和CAUSE[BD]，当新异常处理完成后，由于EPC之前没有重置，就直接返回到发生异常原始指令处，在TLB Refill的情况下，通常返回用户态。

### 页表如何维护？

这是一个关键而且难以理解问题，每个进程都有自己独立的虚拟地址空间，即每个进程都需要页表，在X86中，MMU可以直接使用页表(页目录地址写到CR3寄存器中就可以了)，还可以通过page fault来动态按需创建更新页表项，但MIPS中没有这样的机制，怎么办？

- 预先映射？即事先将页表创建好，即静态方式，那么映射2G的用户态地址空间就需要4M大小的内存。现在内存不值钱，多的是？但实际上进程实际使用的内存远比虚拟内存小，这样浪费就有点过分了。

- 使用虚拟地址，动态分配？主流MIPS中确实是这样实现的。但这其中涉及一些关键问题：
  - 页表使用的虚拟地址也需要做地址映射，而MIPS中，是基于TLB映射，而如果TLB中此时不存在此映射，显然会再次触发(由硬件触发)TLB Miss异常，说**再次**，是因为我们当前已经在TLB Miss异常中了，嵌套了？此时如何处理，如何实现页表的维护？

MIPS硬件在这里做了特殊设计：当访存发现TLB中不存在对应地址映射时(此时)，同时判断CP0 Status寄存器的EXL位，如果为1,表示当前已经在异常处理中，此时，不再重复进入TLB Refill异常(高优先级异常，对应异常编号为1)，而是直接进入普通优先级异常(通用异常，对应异常编号为3)处理，同时会在CP0的STATUS寄存器[6:2]中填入相应的ExcCode(页表中不存在地址翻译条目的情况，对应的ExcCode为2，即TLB Load异常)，通用异常的处理接口中，会根据ExcCode调用相应处理接口，其中对于**缺页**的情况，会最终调用do_page_fault接口，实现内存分配和页表维护。

在TLB Refill过程中再次发生的TLB miss，通常也称为TLB Load/Store异常，此时处理流程跟TLB refill是不同的。过程类似于X86中的缺页异常。

## 缺页后如何更新TLB

如前面所述，当在异常处理过程中(Status[EXL] = 1)，又出现新异常时，CPU将不会重新设置EPC和CAUSE[BD]，当新异常处理完成后，由于EPC之前没有重置，就直接返回到发生异常原始指令处。

也就是说，当TLB Refill过程中再次发生TLB miss(实现了类似X86中的缺页异常的功能，完成页表维护)，进行处理后，会直接返回最初发送TLB Refill异常的地方，而不会回到第二次TLB Miss发生的地方，也就是说，虽然页表更新成功了，但TLB中还没有填入相应的映射呢，那页表中的内容是如何写到TLB中的呢？

答案是：又一次异常。当返回最初发生TLB Refill异常的指令后，会重新执行原来的指令，此时由于TLB中仍没有相应的映射条目，所以硬件会再次触发TLB Refill异常，而此时，页表中的条目已经准备好了，异常处理中会直接从页表refill到TLB。

## MIPS地址翻译全过程

再次完整梳理一遍MIPS地址翻译的全过程：

1. 当CPU做访存操作时，MMU硬件会自动到TLB中寻找相应的映射条目。
2. 如果映射条目存在，则直接取出做映射，得到物理地址，这是最快的路径。
3. 如果条目不存在，则MMU硬件触发TLB Refill(也称TLB Miss)异常，跳转到固定的高优先级异常入口地址处理，其中是内核填入的处理接口，会通过Context寄存器读取位于内存中的页表中对应的条目，

  - 如果页表中相应映射条目存在，则直接读取，并填入TLB(这些操作由软件，也就是内核完成)。
  - 由于页表本身也是通过虚拟地址映射的(通常kseg2段)，所以其自身也需要通过TLB映射，而此时异常处理接口中会通过页表的虚拟地址来做访存操作，如果页表不存在相应映射条目，则MMU硬件会再次触发TLB Miss异常。
  - 由于MIPS硬件上做了特殊处理，对于在异常中(Status[EXL] = 1)再次发生异常的情况，不会重新进入，而会进入General exception的处理，并设置ExcCode为2。
  - General exception处理中，会根据ExcCode，条用相应的处理接口，此时对应为handle_tlbl。
  - 在handle_tlbl中，会根据情况条用do_page_fault接口更新和维护页表。
  - General exception处理完成后，直接返回到第一次发生TLB Refill异常的地址，重新执行原来触发异常的指令，此时会再次触发TLB Refill异常，然后从页表refill相应的条目到TLB。至此，处理结束。

可以看出，在最糟糕的情况下(页表和TLB中都不存在相应映射条目)，会触发**3次异常**，才能完成TLB Refill(而X86的page fault只需要一次异常)，这样的性能能好么？

实践证明，其实还不错。最糟糕的情况相对比较少～
