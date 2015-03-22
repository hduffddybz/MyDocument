# Linux内核启动流程

## 课程说明

周元斌+原创作品转载请注明出处+ 《Linux内核分析》MOOC课程http://mooc.study.163.com/course/USTC-1000029000

## 实验环境

Debian Wheezy(kernel:3.2.0),qemu(1.1.2),gdb(7.4.1)

内核源码：3.18.6

调试环境

![image](https://github.com/hduffddybz/MyDocument/raw/master/img/start_1.png)

## 实验目标

分析start_kernel的启动流程

计算机启动时会执行BIOS的代码，BIOS代码完成后会跳转到MBR中的引导程序，引导程序则会将Linux kernel的代码加载到内存当中并跳转到start_kernel函数处开始执行，分析start_kernel的代码至关重要了。

### 启动过程简要分析

- 初始化0号进程

该初始化通过set_task_stack_end_magic函数来实现，打印init_task.pid可以发现init_task的进程号为0

- 初始化页地址

- 虚拟文件系统初始化

- 系统保留中断向量的初始化及其他中断向量的初始化

该初始化是通过trap_init函数完成的

- 调度器初始化

- slab内存管理算法初始化

- init进程初始化

（电脑挂了，完全没法干活了！稍后再补上）
