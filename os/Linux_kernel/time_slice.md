# Linux时间片轮转调度算法的实验分析

## 课程说明

周元斌+原创作品转载请注明出处+ 《Linux内核分析》MOOC课程http://mooc.study.163.com/course/USTC-1000029000 

## 实验环境

Debian wheezy (kernel:3.2.0), qemu(1.1.2), gdb(7.4.1)

内核源码3.9.4，打上mengning为mykernel做的补丁（https://github.com/mengning/mykernel）

调试环境：

![image](https://github.com/hduffddybz/MyDocument/raw/master/img/time_2.png)

## 实验1（搭建qemu+mykernel调试环境）

- 为3.9.4内核源码打上mykernel补丁
- make allnoconfig（该配置会省略掉绝大多数的内核选项）
- make menuconfig(选中compile for debugging info)
- qemu -kernel arch/x86/boot/bzImage -s -S(允许使用gdb调试), gdbserver tcp::12345
- 另一个窗口敲入gdb vmlinux，target remote localhost:12345

### mykernel环境的启动过程

当qemu将kernel载入到ROM之后，会跳转到start_kernel函数执行，start_kernel会做一些定时器，内存，虚拟文件系统，调度器，console等初始化工作，完成这些工作后就会跳转到my_start_kernel函数里面执行（这是补丁做的工作），另外启动过程中late_time_init函数还会做时钟中断处理函数的初始化工作，mykernel补丁中在timer_interrupt函数中增添了my_timer_handler的处理，就把时钟中断的处理过程转入到了my_timer_handler中。以上两个工作是mykernel补丁做的主要工作。

## 实验2（利用mykernel完成时间片轮转调度算法实验分析）

### 进程控制块

时间片轮转意味着多进程的执行环境，需要为一个进程保存相关上下文信息才能做到进程的切换。一个进程控制块需要保存如下信息：进程号:pid，进程状态，该进程所使用到的堆栈，eip,esp指针，进程入口函数，下一个进程控制块，因而在mykernel中设计了这样的结构体：

``` c
typedef struct PCB{
    int pid;
    volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
    char stack[KERNEL_STACK_SIZE];
    /* CPU-specific state of this task */
    struct Thread thread;
    unsigned long	task_entry;
    struct PCB *next;
}tPCB;
```

### 多进程环境的初始化

在mykernel中在启动kernel时候会初始化0，1，2，3四个进程，所做的工作包含了初始化进程号，进程状态，进程入口函数，进程堆栈，并将四个进程链接起来，并将进程0作为当前执行的进程，其代码在my_start_kernel所示：

``` c
void __init my_start_kernel(void)
{
    int pid = 0;
    int i;
    /* Initialize process 0*/
    task[pid].pid = pid;
    task[pid].state = 0;/* -1 unrunnable, 0 runnable, >0 stopped */
    task[pid].task_entry = task[pid].thread.ip = (unsigned long)my_process;
    task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
    task[pid].next = &task[pid];
    /*fork more process */
    for(i=1;i<MAX_TASK_NUM;i++)
    {
        memcpy(&task[i],&task[0],sizeof(tPCB));
        task[i].pid = i;
        task[i].state = -1;
        task[i].thread.sp = (unsigned long)&task[i].stack[KERNEL_STACK_SIZE-1];
        task[i].next = task[i-1].next;
        task[i-1].next = &task[i];
    }
    /* start process 0 by task[0] */
    pid = 0;
    my_current_task = &task[pid];
	asm volatile(
    	"movl %1,%%esp\n\t" 	/* set task[pid].thread.sp to esp */
    	"pushl %1\n\t" 	        /* push ebp */
    	"pushl %0\n\t" 	        /* push task[pid].thread.ip */
    	"ret\n\t" 	            /* pop task[pid].thread.ip to eip */
    	"popl %%ebp\n\t"
    	: 
    	: "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)	/* input c or d mean %ecx/%edx*/
	);
}   
```


void __init my_start_kernel(void)
{
    int pid = 0;
    int i;
    /* Initialize process 0*/
    task[pid].pid = pid;
    task[pid].state = 0;/* -1 unrunnable, 0 runnable, >0 stopped */
    task[pid].task_entry = task[pid].thread.ip = (unsigned long)my_process;
    task[pid].thread.sp = (unsigned long)&task[pid].stack[KERNEL_STACK_SIZE-1];
    task[pid].next = &task[pid];
    /*fork more process */
    for(i=1;i<MAX_TASK_NUM;i++)
    {
        memcpy(&task[i],&task[0],sizeof(tPCB));
        task[i].pid = i;
        task[i].state = -1;
        task[i].thread.sp = (unsigned long)&task[i].stack[KERNEL_STACK_SIZE-1];
        task[i].next = task[i-1].next;
        task[i-1].next = &task[i];
    }
    /* start process 0 by task[0] */
    pid = 0;
    my_current_task = &task[pid];
	asm volatile(
    	"movl %1,%%esp\n\t" 	/* set task[pid].thread.sp to esp */
    	"pushl %1\n\t" 	        /* push ebp */
    	"pushl %0\n\t" 	        /* push task[pid].thread.ip */
    	"ret\n\t" 	            /* pop task[pid].thread.ip to eip */
    	"popl %%ebp\n\t"
    	: 
    	: "c" (task[pid].thread.ip),"d" (task[pid].thread.sp)	/* input c or d mean %ecx/%edx*/
	);
}  

其中执行完进程初始化，内存分布如下所示：
![image](https://github.com/hduffddybz/MyDocument/raw/master/img/time_1.png)

而这段嵌入式汇编的作用是设置当前进程的esp（movl %1,%%esp），设置ebp(pushl %1,popl %%ebp),跳转到进程入口函数执行（pushl %0, ret）

注意到my_process函数执行的很简单，过了一段时间片后，时间中断处理函数会将my_need_sched设置为1，这时my_process函数会主动执行进程的上下文切换！

### 进程的上下文切换

进程的上下文切换在my_schedule函数完成，next结构体保存的是下一个进程的控制块，而prev结构体保存的是当前进程的控制块，进程切换可分为两种情况，一种待切换的进程已经运行过但是被挂起，另一种情况为待切换的进程为新的进程，因而就有了两种切换方式：

#### 情形1（切换到runnable进程）

代码如下：
``` asm
asm volatile(	
        	"pushl %%ebp\n\t" 	    /* save ebp */
        	"movl %%esp,%0\n\t" 	/* save esp */
        	"movl %2,%%esp\n\t"     /* restore  esp */
        	"movl $1f,%1\n\t"       /* save eip */	
        	"pushl %3\n\t" 
        	"ret\n\t" 	            /* restore  eip */
        	"1:\t"                  /* next process start here */
        	"popl %%ebp\n\t"
        	: "=m" (prev->thread.sp),"=m" (prev->thread.ip)
        	: "m" (next->thread.sp),"m" (next->thread.ip)
    	); 
```

其完成的主要功能是：保存当前进程ebp（pushl %%ebp,）保存当前进程esp（movl %%esp，%0），获得下一个进程的ebp，esp（movl %2,%%esp），保存当前进程下一条指令的eip（movl $1f,%1）, 跳转到下一个进程执行

#### 情形2（切换到新进程执行）

唯一的不同在于增添了mov1 %2， %%ebp的过程这是新的进程并不像曾经运行过的进程一样出现过ebp压栈的情形。
