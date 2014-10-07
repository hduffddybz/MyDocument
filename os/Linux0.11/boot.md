# 利用Bochs分析Linux0.11内核启动过程

本篇文章主要参考了赵炯的《Linux 0.11内核完全注释》和Linux 0.11内核的源码，利用了Bochs-2.1.1 for Windows和Source Insight 3.50等工具。

## 总体概述

在boot目录下主要有bootsect.s,head.s,setup.s三个汇编文件，三个汇编文件完成了启动引导的流程。

8086体系结构的计算机在上电后将由BIOS进行系统的自检，之后BIOS会将bootsect.s文件读入内存的0x7C00（31k）处，在bootsect.s执行时会将自己移动的内存的0x90000（576k）处，并把存储设备中的setup.s文件读入到内存的0x90200（576.5k）处，system模块读入到内存的0x10000（64k）处。三个文件的启动顺序如下：

	BIOS----->bootsect.s----->setup.s----->head.s----->main.c

在Linux0.11内核阶段，System模块大小不超过512k，因而将System模块放在以0x10000为起始地址的内存中并不会覆盖bootsect.s,setup.s的代码，setup.s的代码会将System模块从内存的0x10000位置移动到内存的起始位置。之所以不在一开始时就将System模块放置在内存的起始处是因为：setup.s的代码起始部分需要利用内存起始位置放置的中断向量表来获取一些硬件的参数，等待中断调用结束后才能覆盖这一部分的内存区域。

## bootsect.s代码分析

### 主要功能
bootsect.s是磁盘引导块程序，在bootsect.s中完成的功能主要有：

1. 将自己移动到内存的0x90000处并开始执行
2. 读取磁盘第2个分区开始的连续4个分区并将它们加载到bootsect后（0x90200）
3. 将磁盘上setup后的System模块加载到内存的setup模块后
4. 确定根文件系统的设备号，并转到setup模块执行

### 利用Bochs进行bootsect.s分析

Markdown中插入图片不是很方便，这里不截图了

1. 在0x0000:0x7c00处设置断点

	敲入命令vbreak 0x0000:0x7c00在0x7c00处设置断点，敲入命令C，直到遇到断点停止。

2. 将bootsect移动到0x90000处

	敲入命令u /10(表示从当前代码开始的10条指令)，其输出如下：
		
		mov ax, 0x7c0
		mov ds, ax
		mov ax, 0x9000
		mov es, ax
		mov cx, 0x100
		sub si, si
		sub di, di
		rep movs word ptr [di], word ptr [si]
		jmp 9000:0018
 	
	在这段代码中是将内存中的0x7c00的512字节的内容移动到0x9000为起始地址的内存中，并跳转至0x9000:0x0018内存中。

3. 设置段寄存器
	
	通过敲入s 264命令，程序将跳转到0x9000:0x0018处执行bootsect，接下来执行了：
	
		mov ax, cs
		mov ds, ax
		mov es, ax
		mov ss, ax
		mov sp, 0xff00

	这段代码中设置了ds，es，ss等寄存器，并将堆栈设置在0x9000:0xff00处

4. Setup模块读入内存
	
	Setup模块读入内存利用了BIOS的INT 13号中断，将磁盘第2个扇区后的连续4个扇区读入0x90200为起始地址的内存。
	
		mov dx, 0x0 ; dh:磁头号， dl:驱动器号
		mov cx, 0x2 ; ch:磁道号的低8位，cl:开始扇区
		mov bx, 0x200
		mov ax, 0x204 ; ah=0x02 读取扇区到内存，al:读取的扇区数量
		int 0x13

### 遗留问题

将System模块读入内存和确定根文件系统的设备号的代码没有调试到，欢迎告知。

## setup.s代码分析

### 主要功能

setup.s是一个操作系统加载程序，在setup.s中完成了如下功能：

1. 利用BIOS中断读取系统数据，将这些数据保存到0x90000开始的位置（这些数据有：光标位置、扩展内存数、显示页面、显示模式、字符列数、显示内存、显示状态、硬盘参数、根设备号等）
2. 将System模块由内存0x10000处移动到内存起始处
3. 为head.s能在保护模式下运行做准备
	
### 利用Bochs进行setup.s分析		

1. 读光标位置
	
	利用了BIOS中断获取当前屏幕光标位置，存在0x90000处，代码如下：

		mov ax, 0x9000
		mov ds, ax
		mov ah, 0x3 ;ah=0x3 读光标位置
		xor bh, bh
		int 0x10
		mov word ptr [ds:0x0], dx ；dh=行号，dl=列号

2. 	取系统扩展内存大小
	
	扩展内存大小存在0x90002处，代码如下：
		
		mov ah, 0x88 ;ah=0x88 取系统所含扩展内存大小
		int 0x15
		mov [ds:0x2], ax

	接下来还利用BIOS中断读取了硬盘参数、显示状态等，这里不再阐述这些代码了。

3. 将System模块移动到内存起始处

	这部分没看懂

4. 进入保护模式前的准备工作
	
	进入保护模式前，需要设置gdt、idt寄存器的值，打开保护模式开关，该部分代码没有仔细研究。
	
## head.s代码分析

### 主要功能

head.s位于system模块最开始的部分，主要完成了以下的功能：

1. 加载各个数据段寄存器的值，重新设置中断描述表、全局描述表
2. 设置分页处理机制
3. 调用main()程序
		