# 使用gcc,gdb学习X86汇编

## 前言

 这是中国MOOC Linux内核分析这门课的作业，因作业需求必须做个前言咯：

 周元斌+原创作品转载请注明出处+《Linux内核分析》MOOC课程http://mooc.study.163.com/course/USTC-1000029000

## 目的

 写这篇文章的目的在于提高对X86汇编，对X86体系结构的认识，熟悉gcc，gdb等工具的使用

## 源码与要求

``` c
#include <stdio.h>

int g(int x)
{
    return x + 5;
}

int f(int x)
{
    return g(x);
}

int main(void)
{
    return f(12) + 5;
}

```

 通过这段简单的代码理解汇编代码以及其工作过程中堆栈的变化

## 实验过程

这里并不使用实验楼提供的环境，一来实验楼使用起来并不流畅，二者他连gdb都没有，而且没有root权限无法随心所欲发挥，所以实验中使用了本地虚拟机的Debian7.9，gcc 4.7.2,gdb 7.4.1的环境

### 编译

使用gcc -o main -g main.c 来编译源码，生成的二进制代码可用gdb进行调试（由于-g参数的作用）

### 调试

- 使用gdb ./main 命令
- 在gdb命令行中使用layout asm,layout regs来查看汇编和寄存器

下面来看看调试的环境：

![image](https://github.com/hduffddybz/MyDocument/raw/master/img/gdb_1.png)

![image](https://github.com/hduffddybz/MyDocument/raw/master/img/gdb_2.png)

### 具体过程

#### 使用命令
- b *0x80483fa（设置断点）
- r（执行）
- si（单步汇编）

#### 逐行分析

（main函数）

- push %ebp 将main函数栈的栈底对应的内存地址压栈（用于函数返回之用）
- mov %esp,%ebp 将main函数栈栈底指向栈顶（新建一个栈存储变量值）
- sub $0x4,%esp 将main函数栈栈顶地址值减4
- movl $0xc,(%esp) 将参数12值压栈，可以通过在gdb命令行打入x /d 0xbffffc84,可以发现其值确实为12
- call 0x80483e7 <f>  跳转到函数f

（f函数）

- push %ebp 将f函数栈的栈底对应的内存地址进行压栈（用于函数返回之用）
- mov %esp,%ebp 将f函数栈栈底指向栈顶（新建一个栈存储变量值）
- sub $0x4,%esp 将f函数栈栈顶地址值减4
- mov 0x8(%ebp),%eax 将参数值12写入寄存器eax
- mov %eax,(%esp) 将eax值写入堆栈中，可以打入x /d 0xbffffc78查看其值为12，正确
- call 0x80483fc <g> 跳转到函数g
 
（g函数）

- push %ebp 将g函数栈栈底对应的内存地址压栈
- mov %esp,%ebp 将g函数栈栈底指向栈顶
- mov 0x8(%ebp),%eax 将参数值12传入寄存器eax,可通过打入 x /d 0xbffffc84查看其值确实为12
- add $0x5,%eax 寄存器eax加5
- pop %ebp 退栈
- ret 返回

（f函数）

- leave 删除为存储变量值x建立的堆栈空间
- ret 函数返回

（main函数）

- add $0x5,%eax 函数返回值存在寄存器eax中，并对其值加5
- leave 删除为存储变量值x建立的堆栈空间
- ret 函数返回

### 函数调用过程帧栈变化

这里增加对于函数调用过程的帧栈变化的理解

函数调用与函数返回主要用到了call，leave, ret等汇编指令完成的,call（函数调用）相当于如下指令：
- push %eip(下一条指令地址)
- jmp function

leave相当于如下指令：
- push %ebp,%esp(撤销栈空间)
- pop %ebp（跳转到上一个帧栈的%ebp）

而ret（函数返回）相当于如下指令：
- pop %eip


下面就以f函数的调用为例说明：

反汇编后看到这里发生了函数f的调用：0x8048407 <main+13>     call   0x80483e7 <f> ，可以看看调用前main函数的帧栈：

![image](https://github.com/hduffddybz/MyDocument/raw/master/img/gdb_3.png)

会发现ebp-esp=0x04,通过键入x 0xbffffca4可以发现在main堆栈的该段地址放的是变量值12（f（12）的12）.

键入si,则跳转到了函数f，此时的f函数的帧栈如下：

![image](https://github.com/hduffddybz/MyDocument/raw/master/img/gdb_4.png)

而同时也可以看看main函数调用处的下一指令地址：

![image](https://github.com/hduffddybz/MyDocument/raw/master/img/gdb_6.png)

从f函数的帧栈中可以看到call指令确实是将下一条指令的地址压入堆栈，并进行跳转。
进入函数f后要执行帧栈的建立，具体代码如下：

- push %ebp
- mov %esp,%ebp

经历如下步骤后，函数的帧栈变化成如下：

![image](https://github.com/hduffddybz/MyDocument/raw/master/img/gdb_8.png)

这里新建了函数f的帧栈(堆栈向下增长)。
再来看看函数返回的情况，执行return汇编后f函数的帧栈的变化：

![image](https://github.com/hduffddybz/MyDocument/raw/master/img/gdb_7.png)

这时的帧栈跟进入f函数时的帧栈一致，只要执行pop %eip（通过ret汇编完成）即可完成函数的返回

综上所述：函数调用与函数返回经历了如下过程：
1.（调用过程）调用处下一条指令地址入栈
2.跳转到新函数
3.旧的ebp入栈，新建堆栈使得ebp等于esp
4.（返回过程）撤销堆栈，使得esp等于ebp
5.更改ebp值为调用函数的ebp值，通过退栈实现
6.弹出调用处下一条指令的地址值至eip
## 体会

计算机通过堆栈的压入弹出实现变量的传递和函数的跳转，堆栈是构成程序基本元素。另外学习汇编有助于加深对程序的理解！
