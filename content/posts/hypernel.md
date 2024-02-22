---
title: 'Hypernel: 无需两阶段地址翻译的硬件辅助的内核保护框架'
date: 2024-02-22T20:18:38+08:00
---
标题原文：Hypernel: A Hardware-Assisted Framework for Kernel Protection without Nested Paging

发表于2018年的DAC（CCF A），作者来自韩国首尔国立大学。

## 介绍

操作系统内核的TCB可信计算基太大，任何小的漏洞都可能导致全盘崩溃。一种解决方案是使用Hypervisor来监控和保护关键系统。常用的有以下三种办法：

- 特权指令陷入：主要指的是系统寄存器。之前这篇“基于硬件隔离的移动系统安全研究“有类似的方式。
- Hypercall
- 嵌套分页：就是两阶段地址翻译。

但是Hypervisor的引入给整个系统带来了很大的开销。文中的引文指出，即使有硬件虚拟化，两阶段地址翻译仍然带来了30%的访存开销。粗粒度的内存保护也会造成额外开销。Hypervisor对内存的保护粒度是页。如果一个页中包含了敏感数据，那么整个页都会被标记。

为了解决这两个问题，作者提出了Hypernel。**Hypernel由两个部分组成：Hypersec和memory bus monitor (MBM)。**

- Hypersec主要负责提供安全的执行环境，对内核中受信任的部分和不受信任的部分进行隔离。位于EL2，是虚拟化的一部分。
- MBM则是处理器和内存之间的系统总线上的一个硬件拓展。通过MBM，Hypernel能够进行更细粒度，也就是**字粒度（word granularity）的内存监视**。当MBM发现对受保护的敏感数据尝试进行写入时，会引发中断。

**其中MBM由FPGA综合以后实现。**（感觉难度挺大）

攻击模型：利用任何现有的内核漏洞来改变内核内存。

## 设计

整个框架被分为两部分。（与TEE无关，看名字可能会认为是基于TrustZone做的）

- 一部分被称为normal space，主要包含操作系统内核和用户应用。
- 另外一部分被称为secure space，主要包含安全应用。

为了将normal space和secure space进行隔离，作者赋予了secure space同等于hypervisor的权限。Hypersec位于secure space中，负责提供隔离的执行环境，而MBM则为其提供了更细粒度的内存监视能力。

作者用hypercall替换了内核中负责更新页表的代码（**修改了内核代码**），使得Hypersec能够直接监视内核页表的更改，并禁止创建任何映射EL2安全空间的页表条目。通过这种方式，Hypersec能够在不使用两阶段翻译的情况下创建隔离环境。

此外，作者利用instruction trapping的功能对特权指令进行了捕获，避免攻击者修改一些重要的系统寄存器例如TTBR导致Hypervisor无法正常工作。

MBM连接在系统总线上来监视CPU和内存。当对监控区域的内存进行修改，MBM会**触发中断**以通知Hypersec进行处理。MBM可以通过EL2的Hypersec来获知处理器内部的信息，所以不容易被绕过。

### 工作流程

整个过程就是围绕Hypercall和MBM验证内存写操作，整体比较清晰。

## 实现和测试

### 平台

主板：ARM Versatile Express Juno r1 平台，ARMv8-A，双核A57+四核A53 （TX2是四核A57）

子板：ARM. LogicTile Express 20MG Daughter Board，包含一个Xilinx Virtex-7 **FPGA**, XC7V2000T-1

### 规模

Hypersec：EL2的Hypervisor，1.5kLoC

Linux Kernel：200LoC

MBM：FPGA上大约55k个门电路

### 实现

设置HCR_EL2的TVM位，代理敏感寄存器的操作。

修改内核代码，通过Hypercall来修改页表，由Hypervisor代理这一操作。

### 性能

肯定都说自己是好的，优于KVM的。

总体上的性能开销是3.1%，而KVM是13.5%.

具体性能还是看原文吧。

## 总结

**值得借鉴的地方**

- 脱离SoC上的硬件限制，引入外部硬件辅助。（Zynq）

**存在的问题**

- 没有开源，猜测Hypersec是Type-1的，很多设备都是直通。
- 把KVM的性能开销简单的归结于两阶段地址翻译有失偏颇，应该还和设备模型、内核调度等有关。

**感觉不太适合我们的Hypervisor优化演进**

- 要修改内核代码。
- 咱们的Hypervisor算是一种Partitioning Hypervisor（Bao JailHouse也说自己是Partitioning Hypervisor），而本文更强调Security。
- 还是要思考怎么针对Partitioning Hypervisor继续演化。