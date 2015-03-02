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

 通过这段简单的代码理解汇编代码工作过程中堆栈的变化

## 实验过程

这里并不使用实验楼提供的环境，一来实验楼使用起来并不流畅，二者他连gdb都没有，而且没有root权限，所以实验中使用了本地的Debian7.9，gcc 4.7.2,gdb 7.4.1的环境

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
- b *0x80483fa
- r
- si

#### 逐行分析

- push %ebp 将main函数栈的栈底对应的内存地址压栈
- mov %esp,%ebp 将main函数栈栈底指向栈顶
- sub $0x4,%esp 将main函数栈栈顶地址值减4
- movl $0xc,(%esp) 将参数12值压栈，可以通过在gdb命令行打入x /d 0xbffffc84,可以发现其值确实为12
- call 0x80483e7 <f>  跳转到函数f
- push %ebp 将f函数栈的栈底对应的内存地址进行压栈
- mov %esp,%ebp 将f函数栈栈底指向栈顶
- sub $0x4,%esp 将f函数栈栈顶地址值减4
- mov 0x8(%ebp),%eax 将参数值12写入寄存器eax
- mov %eax,(%esp) 将eax值写入堆栈中，可以打入x /d 0xbffffc78查看其值为12，正确
- call 0x80483fc <g> 跳转到函数g
- push %ebp 将g函数栈栈底对应的内存地址压栈
- mov %esp,%ebp 将g函数栈栈底指向栈顶
- mov 0x8(%ebp),%eax 将参数值12传入寄存器eax,可通过打入 x /d 0xbffffc84查看其值确实为12
- add $0x5,%eax 寄存器eax加5
- pop %ebp 退栈
- ret 返回
- leave 删除为存储变量值x建立的堆栈空间
- ret 函数返回
- add $0x5,%eax 函数返回值存在寄存器eax中，并对其值加5
- leave 删除为存储变量值x建立的堆栈空间
- ret 函数返回

## 体会

计算机通过堆栈的弹出压入实现变量的传递和函数的跳转，堆栈是构成程序基本元素。另外学习汇编有助于加深对程序的理解！
