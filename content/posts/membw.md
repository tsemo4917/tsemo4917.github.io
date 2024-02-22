---
title: '虚拟机监控器内存带宽控制'
date: 2024-02-22T20:35:54+08:00
---

## 内存带宽控制

论文题目：**Supporting Temporal and Spatial Isolation in a Hypervisor for ARM Multicore Platforms**

项目代码：https://github.com/pa007/xvisor-next/tree/cache-coloring_and_memory-reservation

```
CONFIG_MEMORY_RESERVATION
```

解决这个问题的一种简单实用的技术是实现一种**内存带宽预留**机制，该机制**限制给定时间窗口内的内存访问次数**，这可能是 Yun 在非虚拟化背景下首次提出的基于软件的解决方案多处理器系统。

Hypervisor上怎么做？给每个vcpu一个**内存访问预算budget**，并**周期性补充**；这些信息都是静态配置的；

这种方法可以减少VM由于内存争用而引起的干扰，内存争用会隐式地受到内存预算的限制，并且独立于在其他VM内运行的软件的实际行为。

### Intel RDT & ARM MPAM

https://cloud-atlas.readthedocs.io/zh_CN/latest/kernel/cpu/intel/intel_rdt/intel_rdt_arch.html

MBA (Memory Bandwidth Allocation)：https://www.intel.com/content/www/us/en/developer/articles/technical/introduction-to-memory-bandwidth-allocation.html

Xen上的一个文档：https://xenbits.xen.org/docs/unstable/features/intel_psr_mba.html

Xen对intel QoS的处理：https://wiki.xenproject.org/wiki/Intel_Platform_QoS_Technologies

ARM MPAM的寄存器：https://developer.arm.com/documentation/ddi0595/2021-12/External-Registers/MPAMCFG-MBW-MAX--MPAM-Memory-Bandwidth-Maximum-Partition-Configuration-Register

### ARM上应该怎么做获取内存访问的budget？

通过PMU，可以记录到data memory access的数量，使用硬件功能，减少软件记录的开销。

PMU还可以产生中断，当某个寄存器溢出的时候，这可以用来消耗预算，避免了软件对预算的反复检查。

**研究一下ARM PMU**

tx2的pmu版本：pmuv3

文档：https://developer.arm.com/documentation/ddi0488/h/performance-monitor-unit?lang=en

一个博客，看/proc/interrupt看看pmu的中断https://www.whexy.com/posts/PMU

cp15寄存器：https://developer.arm.com/documentation/ddi0406/b/Debug-Architecture/Performance-Monitors/CP15-c9-register-map?lang=en

### 怎么控制内存带宽？

**通过vcpu调度**

## PMU驱动

### 所有寄存器

[armv8 aarch64 PMU寄存器介绍](https://www.jianshu.com/p/98b482707845)

| PMCR_EL0         | Performance Monitors Control Register                       | 控制寄存器                        |
| ---------------- | ----------------------------------------------------------- | --------------------------------- |
| PMCCFILTR_EL0    | Performance Monitors Cycle Count Filter Register            | 过滤器                            |
| PMCCNTR_EL0      | Performance Monitors Cycle Count Register                   | 时钟周期数                        |
| PMCEID0_EL0      | Performance Monitors Common Event Identification register 0 | 检查某个event有没有实现，只读     |
| PMCEID1_EL0      | Performance Monitors Common Event Identification register 1 |                                   |
| PMCNTENCLR_EL0   | Performance Monitors Count Enable Clear register            | counter使能                       |
| PMCNTENSET_EL0   | Performance Monitors Count Enable Set register              |                                   |
| PMEVCNTR<n>_EL0  | Performance Monitors Event Count Registers                  | 各类事件的计数                    |
| PMEVTYPER<n>_EL0 | Performance Monitors Event Type Registers                   | 寄存器对应事件类型                |
| PMINTENCLR_EL1   | Performance Monitors Interrupt Enable Clear register        | 中断                              |
| PMINTENSET_EL1   | Performance Monitors Interrupt Enable Set register          |                                   |
| PMOVSCLR_EL0     | Performance Monitors Overflow Flag Status Clear Register    | 溢出状态                          |
| PMOVSSET_EL0     | Performance Monitors Overflow Flag Status Set register      |                                   |
| PMSWINC_EL0      | Performance Monitors Software Increment register            | 写这个寄存器可以使某个counter加一 |
| PMUSERENR_EL0    | Performance Monitors User Enable Register                   | 设置用户态权限                    |
| PMSELR_EL0       | Performance Monitors Event Counter Selection Register       | 选择寄存器                        |
| PMXEVCNTR_EL0    | Performance Monitors Selected Event Count Register          | 写入PMSELR_EL0寄存器后对应的值    |
| PMXEVTYPER_EL0   | Performance Monitors Selected Event Type Register           |                                   |

设置PMCR_EL0寄存器，控制是否记录EL0, EL1, EL2, EL3的事件；

设置PMEVTYPER<n>_EL0寄存器，配置统计的什么事件，具体去查PMU events；

也和这个寄存器相关：MDCR_EL2, Monitor Debug Configuration Register (EL2)

## PMU中断

相关寄存器：

- PMINTENCLR_EL1: Performance Monitors Interrupt Enable Clear register
- PMINTENSET_EL1: Performance Monitors Interrupt Enable Set register
- PMOVSCLR_EL0: Performance Monitors Overflow Flag Status Clear Register
- PMOVSSET_EL0: Performance Monitors Overflow Flag Status Set register

## 此前的内存带宽控制机制的问题

1. 由于DRAM内存的物理结构，其**带宽（单位时间内的读取频数）本就是动态变化的**，与访问的位置、顺序高度相关。
2. 虚拟机的访存带宽是静态分配的，对资源利用率、内存性能都会产生很大影响

这篇文章限制的对象是**物理核心**，我们Hypervisor限制的对象是**虚拟机**。

## 内存带宽控制的前提

按照DRAM子系统最小的吞吐量总和来限制。

> The memory bandwidth available for reservation is restricted to the minimum DRAM service rate as it would make memory access delay caused by concurrent accesses from other cores negligible.

## 整体结构

两部分：每个物理核心内存带宽节流器per-core regulator，还有一个全局的内存带宽回收管理器reclaim manager

- 节流器：监控核心的内存带宽情况，执行节流操作；读硬件寄存器来监控，寄存器溢出时产生中断。每个调节器都有一个基于历史的内存使用预测器。根据预测的使用情况，节流器可以主动放弃其部分预算，以便其他core在用完预算后可以回收这些带宽。
- 回收装置：全局的（加锁），用来接收和重新分发系统中所有节流器放弃的内存带宽；

此前已经做了缓存划分，排除了LLC的空间竞争，所以此时内存带宽预留是有效的；

两个基本的伪代码：

- 周期性的时钟中断处理：
- PMU寄存器溢出的中断处理：

### 内存带宽预留

两个角度：整个系统层面和单独每个核心层面；

- 系统层面：保证所有核心可以访问的带宽不能超过DRAM最小服务速率r_min，在保留带宽上提供性能隔离。第五节A段讲了怎么评估r_min。
- 每个核心层面：每个核心的带宽B_i的和为r_min

B_i = Q_i / P，Q_i为budget，P为补充周期。补充周期的选择很重要，这涉及到中断和调度的频率。

### 内存带宽回收

每个核心的Q_i是一个静态的值。

预测方法：指数加权平均线EWMA

- 指数移动平均（EMA）的原理：https://zhuanlan.zhihu.com/p/68748778
- https://zh.wikipedia.org/zh-hans/%E7%A7%BB%E5%8B%95%E5%B9%B3%E5%9D%87

全局回收装置包含一个共享的带宽G，在每个周期开始的时候各个核心通过放弃带宽来补充，周期结束时清零。

具体回收的规则：

1. 每个周期开始时，当前核心的预算q_i = min(Q_predict_i, Q_static_i)
2. G=sum(Q_static_i - q_i)，即预测的带宽消耗小于设定值时，将回收所有的额外带宽；
3. 如果预算耗尽，核心可以从回收器中申请预算：如果这个周期内使用的预算u_i小于固定分配的Q_static_i（即当预测值小于分配值），那么该核心会尝试申请刚刚放弃的预算数量；如果耗尽了固定分配值Q_static_i，那么有一个Q_min最小预算，取Q_min与G的最小值来设置PMU寄存器；Q_min根据每个特定系统的经验确定，是一个常数，否则太小的话会频繁产生中断，增大开销；
4. 可能出现一个核心在周期开始时，因为预测器给出的预算太小，放弃了大量预算而其他核心申请走了这些预算（G==0）导致该核心无法运行的情况。此时需要修正预测器，预测器会在下一个周期给尝试给该核心补偿预算，此时预测器的输入将编程Q_static_i + (Q_static_i - u_i)

### 备用内存带宽共享

即r_min的带宽肯定是很小的。超过r_min的带宽称为备用内存带宽（Spare Memory Bandwidth），这可以提升整个系统的吞吐量。

当所有核心都耗尽了他们的访存运算，且总和达到了r_min，此时所有核心都会被阻塞。那么到在本周期内，剩余的时间被称作空闲时间，memguard将唤醒所有物理核心来尽可能最大化内存吞吐量。这被称作空闲内存带宽共享机制。

**个人认为不太合理。理由：**

- memguard的对象是物理核心，而非关键任务（进程线程），该研究的目的是为了保障各个物理核心既能得到最低限度的带宽（guaranteed），也能通过该备用内存带宽共享机制最大化吞吐量（best effort）。所以这不是为了保障关键任务来设计的；
- 具体到Hypervisor，我们限制访存的对象是虚拟机的vcpu，即保障的是关键任务。所以我们只对非关键任务加以限制，而不限制关键任务的访存。而备用内存带宽会加剧内存竞争，这对关键任务的执行不利。故内存带宽回收机制也应该是存在于非关键虚拟机之间。

## 怎么测量r_min内存最小服务率

测量指标：latency 和bandwidth

延迟测量方法：通过链表随机访问内存，来测试延迟；参考Isobench。

带宽：顺序访问，最大程度利用DRAM的并行memory level parallelism (MLP)。

TX2上这两个的测试结果是随机访问是0.35GB/s，顺序访问是3.3GB/s，差距很大。

0.35GB/s = 64 * (10 ^ 9 / latency) / 10 ^ 6 = 约350MB/s

> 64代表cacheline大小，latency单位是纳秒ns，

这里取的是两倍，那么tx2可以按照0.7GB/s来做r_min

## 回收机制设计

首先我觉得应该分两级回收机制：

1. vcpu归还budget首先向所属vm归还，申请带宽也首先从所属vm申请。
2. 当某个vcpu耗尽了budget且该虚拟机无buget时，才向全局（即其他虚拟机）申请。

## 预测器设计——Exponential Moving Average

指数加权平均线EWMA

- 指数移动平均（EMA）的原理：https://zhuanlan.zhihu.com/p/68748778
- https://zh.wikipedia.org/zh-hans/%E7%A7%BB%E5%8B%95%E5%B9%B3%E5%9D%87
- 用途：时间序列预测
- 领域：金融、交通；
- [koorye.github.io](https://koorye.github.io/2021/07/29/2021-7-29-数学建模的时间序列模型/) 一次指数平滑法

Rust的EMA实现：

- https://crates.io/crates/ema-rs
- https://crates.io/crates/ta

## 预测/回收的细节

时机：每个带宽控制周期进行一次预测和回收，所以应该在supply_budget这个函数里面做预测和回收。

首先要获取该vcpu上个周期用了多少带宽：

- Stop pmu的时候保存用了多少带宽。
- 如果没有vcpu调度（vcpu独占核心的情况），此时没有stop pmu调用，该怎么办？

那么此时问题就变成了怎么更新used_budget：

- Vcpu被中断时，则会更新used_budget；
- Vcpu每个带宽控制周期会清零used_budget；
- vcpu_stop_pmu：调用update_remaining_budget更新当前剩余的remain_budget，再记录一下之前的预测结果，相减即可获得used_budget

那么remaining_budget仍然保留，used_budget即为last_predict - remaining

- Running->Runnable（vcpu调度）：调用vcpu_stop_pmu 更新remain
- Running->Running：调用vcpu_stop_pmu 更新remain，然后调用supply_budget在其中预测并更新last_predict 和remain_budget，最后调用vcpu_start_pmu来覆盖新的budget到pmu上；（相当于认为创造一个不存在的runnable状态来产生vcpu_stop_pmu）
- Running由于访存预算耗尽被中断，尝试申请额外带宽：
  - 调用try_rescue方法，该函数返回一个bool，如果申请到了则完成中断处理，返回vcpu执行；
  - 没有申请到则Blocked；
- Blocked->Runnable：wakeup该vcpu，调用supply_budget

预测结果：

- 如果vcpu运行的第一个周期，则预测结果应该为静态分配的带宽；
- 后续周期：根据上个周期的内存带宽消耗来预测；

需要改进的地方：内存最小服务率的测试，时间序列预测的方法，带宽耗尽时的内存带宽申请逻辑