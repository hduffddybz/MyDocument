# ARM 虚拟化技术综述

## 基础

在 ARM cortex-A7 和 cortex-A15 中引入了新的运行在 Non-secure mode 下的特权模式 hyp，为实现 hypervisor 之用。这种新的特权模式引入可减少 Hypersivor 的复杂性，主要体现在如下方面：1、无需软件实现 guest OS 的中断屏蔽，2、无需进行 guest OS 的页表管理。

ARM 的虚拟化包含了如下特性：1、对 stage-2 的页表转换进行管理，2、虚拟化 CPU 的所有功能包括了 CP15 协处理器，3、System MMU 选项来帮助实现 IO 虚拟化（？）。

可使用 HVC（Hypervisor call）指令进入 hyp 模式，进入 hyp mode 的中断向量入口地址的偏移量为 0x14，hyp mode 拥有单独的链接寄存器，SPSR，堆栈，HCR（Hypervisor Control Register） 寄存器用以屏蔽虚拟资源而 HSR（Hypervisor Syndrome Register） 寄存器用于存储进入 hyp mode 的上下文（？）。

引入了 hyp mode 后，guest OS 可工作在原先的 kernel/user 特权级结构，而 hyp 模式的优先级要比 guest OS 中的 kernel 级别来的高，因而 VMM 可以进行 guest OS 的访问控制。当然硬件上仍然保留有更高优先级的 secure state。

## 内存虚拟化

当启用了 hyp 模式后，虚实地址的转换将包含了两个步骤，第一个步骤 guest OS 会通过其维护的页表将虚拟地址(Virtual address) 转换成物理地址 IPA（Intermediate Physical address），第二个步骤则是通过 hypervisor 维护的 stage-2 页表将物理地址 IPA（Intermediate Physical address）转换为机器地址 PA（Physical address）。

## 中断虚拟化

中断应该能够被路由到不同的 guest OS 或者是 Hypervisor 更或者是执行在 TrustZone 环境下的 OS/RTOS。初始化的时候中断需要被 Hypervisor 接管，如果某个中断需要被某个 CPU 处理，那么 Hypervisor 需要为该 guest OS 进行虚拟中断的映射。

ARM 的中断虚拟化包含了如下特性：

- 包含有寄存器处理虚拟中断

- guest OS 中的 CPSR 寄存器的 I/A/F 位仅仅作用于该系统，物理中断不会被 CPSR 中的 I,A,F 位值所改变，在 guest OS 中改变 CPSR 寄存器的 I/A/F 位不会 trap 到 hypervisor 中

- 虚拟中断可以路由到 Non-secure 的 IRQ/FIQ/Abort 中断

- guest OS 可以管理虚拟中断控制器

GIC 拥有两套的内部寄存器，物理寄存器和虚拟寄存器，非虚拟化的系统和虚拟化中的 hypervisor 可控制物理寄存器，guest OS 管理虚拟寄存器。虚拟寄存器可以被 hypervisor 重定向可让 guest OS 认为自己正在操作物理寄存器，这种情况下在虚拟化和非虚拟化的场合 GIC 的注册代码和功能是完全一致的。

来看一个虚拟中断的例子：

### 虚拟中断例子


- 外部的 IRQ(被 hypervisor 配置为虚拟中断) 到达 GIC

- GIC 分发器会产生 IRQ 信号到指定的 CPU

- CPU 切换到 hyp 模式，CPU 会从物理接口读取中断状态

- Hypervisor 会从 GIC 的寄存器列表获得入口地址，这时候 GIC 分发器会向指定的 CPU 产生 vIRQ 中断

- CPU 进行中断处理，guest OS 会从虚拟 CPU 接口读取中断状态
