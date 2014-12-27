# 实时操作系统调度算法研究--以RT-Thread为例

## 概述

 实时系统与一般的分时系统的区别在于实时系统在接收一个任务到完成任务的这段抖动时间要尽可能的短。实时系统分为硬实时系统和软实时系统，对于每个任务来说，硬实时系统必须在其截至时间内完成所有任务，而软实时系统只需在平均情况下在其截至时间内完成任务[1]。在汽车、工业、航空等领域，一些任务响应超过其执行截至时间将产生灾难性的影响（例如汽车中安全气囊弹出的响应），因而实时操作系统有着广泛的应用。著名的实时操作系统包括了FreeRTOS，VxWorks，RT-Linux等。

 RT-Thread是一款来自于中国的实时操作系统，包含了实时操作系统内核，TCP/IP协议栈，文件系统等等，她诞生于2006年初，以开源社区加服务公司形式进行运作，具体信息可见http://www.rt-thread.org/[2].

 RT-Thread是一个以线程为基本调度单元的多任务系统，线程提供了任务优先级，运行上下文等相关信息。RT-Thread提供了一种基于优先级的全抢占式调度方式，除了一些中断处理函数，禁止中断的代码不可抢占之外，其他任务都是可以抢占的。这是一种固定优先级的调度算法，其任务优先级不会根据任务执行的时间片进行调整，但在任务运行过程中也可通过一些函数接口来手动的更改优先级。

## RT-Thread调度算法设计实现

 对于一个系统来说，仅仅能提供任务的抢占功能并不能称的上实时系统。调度时间的确定性是实时操作系统的一个主要指标，而影响调度时间的重要因素在于查找最高优先级的时间复杂度，RT-Thread采用了基于位图的优先级算法，将时间复杂度降为O(1)，通过该算法可大大提高查找最高优先级算法的速度。

 RT-Thread存在多个不同优先级的线程，同时支持相同优先级的线程，对于不同优先级的线程采用可抢占式调度的形式，而对于同一优先级的不同线程采用时间片轮转进行调度。

 既然RT-Thread以线程作为基本调度单位，首先看下线程控制块的结构：

``` c
/**
 * Thread structure
 */
struct rt_thread
{
    /* rt object */
    char        name[RT_NAME_MAX];                      /**< the name of thread */
    rt_uint8_t  type;                                   /**< type of object */
    rt_uint8_t  flags;                                  /**< thread's flags */

#ifdef RT_USING_MODULE
    void       *module_id;                              /**< id of application module */
#endif

    rt_list_t   list;                                   /**< the object list */
    rt_list_t   tlist;                                  /**< the thread list */

    /* stack point and entry */
    void       *sp;                                     /**< stack point */
    void       *entry;                                  /**< entry */
    void       *parameter;                              /**< parameter */
    void       *stack_addr;                             /**< stack address */
    rt_uint16_t stack_size;                             /**< stack size */

    /* error code */
    rt_err_t    error;                                  /**< error code */

    rt_uint8_t  stat;                                   /**< thread stat */

    /* priority */
    rt_uint8_t  current_priority;                       /**< current priority */
    rt_uint8_t  init_priority;                          /**< initialized priority */
#if RT_THREAD_PRIORITY_MAX > 32
    rt_uint8_t  number;
    rt_uint8_t  high_mask;
#endif
    rt_uint32_t number_mask;

#if defined(RT_USING_EVENT)
    /* thread event */
    rt_uint32_t event_set;
    rt_uint8_t  event_info;
#endif

    rt_ubase_t  init_tick;                              /**< thread's initialized tick */
    rt_ubase_t  remaining_tick;                         /**< remaining tick */

    struct rt_timer thread_timer;                       /**< built-in thread timer */

    void (*cleanup)(struct rt_thread *tid);             /**< cleanup function when thread exit */

    rt_uint32_t user_data;                              /**< private user data beyond this thread */
};
typedef struct rt_thread *rt_thread_t;

```

其中的init_priority指的是初始化线程时设定的任务优先级，在运行过程中系统不会动态的调整优先级的大小，但可通过线程控制函数手动更改线程优先级。在实时系统中任务执行的确定性是一个重要因素，动态优先级算法会增加代码实现复杂性，会为系统引入更多的不确定性，并增加CPU的占用率，并不利于移植到低端的嵌入式平台。

### 基于位图的优先级查找算法

RT-Thread中定义了跟位图相关的变量：

``` c
(1)rt_list_t rt_thread_priority_table[RT_THREAD_PRIORITY_MAX];
struct rt_thread *rt_current_thread;

rt_uint8_t rt_current_priority;

#if RT_THREAD_PRIORITY_MAX > 32
/* Maximum priority level, 256 */
rt_uint32_t rt_thread_ready_priority_group;
rt_uint8_t rt_thread_ready_table[32];
#else
/* Maximum priority level, 32 */
rt_uint32_t rt_thread_ready_priority_group;
#endif
```

其含义将在接下来一步步讲解：

算法1：语句(1)定义了线程控制块数组，其线程最大优先级为256，一般的我们会想到如下的调度算法：

``` c
for(i = 0; i < RT_THREAD_PRIORITY_MAX; i++)
{
    if(rt_thread_priority_table[i] != NULL)
	    break;
}
highest_ready_priority = i;
```
显而易见的是该算法的时间复杂度为O(N)(N为最大线程优先级)，这在实时系统中显然是无法接受的。

算法2：算法1中用一个字节来表示一个优先级，这不免有些浪费，因为某个优先级上是否有就绪任务是一个是否问题，只需一个bit位就可表示。对于RT-Thread来说她支持256个优先级，在C语言中不支持256bit变量的定义，可用rt_uint8_t rt_thread_ready_table[32]来定义。于是乎我们有了算法2：
``` c
for(i = 0; i < 32; i++)
{
	for(j = 0; j < 8; j++)
	{
		if(rt_thread_priority_table[i] & (1 << j))
			break;
	}
	highest_ready_priority = i * 8 + j;
}
```
分析下算法2我们发现其时间复杂度还是为O(N)，这种算法还是不能使用的。

算法3：rt_uint8_t rt_thread_ready_table[32]存储的是一个变量值，当rt_thread_ready_table[RANDOM]=255时，所有的bit都为1，则8个优先级都存在就绪任务，因而最高优先级为bit0对应的那个线程，类似的可以推广到其他情况，我们可以通过维护一张表，程序只要查表就能找到最高的优先级，这是一种以空间换时间的做法，RTT中维护了这样一张表：
``` c
const rt_uint8_t __lowest_bit_bitmap[] =
{
    /* 00 */ 0, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 10 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 20 */ 5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 30 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 40 */ 6, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 50 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 60 */ 5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 70 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 80 */ 7, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* 90 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* A0 */ 5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* B0 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* C0 */ 6, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* D0 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* E0 */ 5, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0,
    /* F0 */ 4, 0, 1, 0, 2, 0, 1, 0, 3, 0, 1, 0, 2, 0, 1, 0
};
```
这个表只包括了8个优先级的情况，可想而知当优先级增长到256时这个表将存储2**32=4G的空间，嵌入式芯片将无法存放。所以仅仅依靠查表的做法是不可行的。

现在考虑32个优先级的情况，32个优先级分为4组，每组有8个优先级，每组用一个rt_uint8_t变量表示，查表时先确定最高优先级在那个字节中，当确定出了是哪个字节后再进行查表，以确定最高优先级对应的bit位，具体代码如下：
``` c
 if (value == 0) return 0;

    if (value & 0xff)
        return __lowest_bit_bitmap[value & 0xff] + 1;

    if (value & 0xff00)
        return __lowest_bit_bitmap[(value & 0xff00) >> 8] + 9;

    if (value & 0xff0000)
        return __lowest_bit_bitmap[(value & 0xff0000) >> 16] + 17;

    return __lowest_bit_bitmap[(value & 0xff000000) >> 24] + 25;
``` 

以上代码中解决了32个优先级的问题，当优先级扩展到256位的时候，仅仅简单的改造以上的代码并不能奏效，因为在256/8=32个字节中查找最高优先级所属的字节并不简单。聪明的RT-Thread开发人员使用了二级位图的方法来解决这个问题。

超过32个优先级对应的位图相关变量定义如下：
``` c
#if RT_THREAD_PRIORITY_MAX > 32
/* Maximum priority level, 256 */
rt_uint32_t rt_thread_ready_priority_group;
rt_uint8_t rt_thread_ready_table[32];
#else 
``` 
256个优先级有256个bit位由32个字节存储，每一个字节的8个bit代表8个优先级，在使用二级位图中，我们首先对这32个字节（需要引入32个bit的位图变量）进行查找，确定所属的字节，再对该字节内的bit位进行查找，找出最高优先级对应的bit位。上述定义中rt_thread_ready_table为位图变量数组，而为了提高查找效率又引入了rt_thread_ready_priority_group的位图变量，其实现如下：
``` c
#if RT_THREAD_PRIORITY_MAX > 32
    register rt_ubase_t number;

    number = __rt_ffs(rt_thread_ready_priority_group) - 1;
    highest_ready_priority = (number << 3) + __rt_ffs(rt_thread_ready_table[number]) - 1;
#else
```
其中__rt_ffs()实现如下： 
``` c
 if (value == 0) return 0;

    if (value & 0xff)
        return __lowest_bit_bitmap[value & 0xff] + 1;

    if (value & 0xff00)
        return __lowest_bit_bitmap[(value & 0xff00) >> 8] + 9;

    if (value & 0xff0000)
        return __lowest_bit_bitmap[(value & 0xff0000) >> 16] + 17;

    return __lowest_bit_bitmap[(value & 0xff000000) >> 24] + 25;
``` 

用二级位图的方法轻松的实现了256个优先级的查找，轻松的达到了O(1)的时间复杂度。

###基于时间片的调度算法

RT-Thread中支持相同优先级的不同任务，对于同优先级的任务调度其采用了基于时间片的调度算法。当系统中存在多个相同的最高优先级线程时，调度器会根据创建线程时设定的时间片进行调度。

我们知道对于任务调度时机存在着以下几种情况，第一种就是任务主动放弃CPU资源，任务主动调用rt_schedule()进行任务切换，第二种就是外部中断发生时，有更高优先级的任务进入就绪状态，这时也会进行任务切换，第三种情况就是时间中断，时间中断发生时，若任务运行到了其规定的时间片时也会进行任务切换。

首先看看经过了一个滴答后，系统做了哪些处理（在src/clock.c下）
``` c
/**
 * This function will notify kernel there is one tick passed. Normally,
 * this function is invoked by clock ISR.
 */
void rt_tick_increase(void)
{
    struct rt_thread *thread;

    /* increase the global tick */
    ++ rt_tick;

    /* check time slice */
    thread = rt_thread_self();

  （1）  -- thread->remaining_tick;
  （2）if (thread->remaining_tick == 0)
    {
        /* change to initialized tick */
        thread->remaining_tick = thread->init_tick;

        /* yield */
        （3）rt_thread_yield();
    }

    /* check timer */
    rt_timer_check();
}
``` 
[3]可见主要的工作为判断任务的时间片是否执行完毕(1),(2)，然后任务执行让出处理器动作(3)，具体见rt_thread_yield（）函数：
``` c
/**
 * This function will let current thread yield processor, and scheduler will
 * choose a highest thread to run. After yield processor, the current thread
 * is still in READY state.
 *
 * @return RT_EOK
 */
rt_err_t rt_thread_yield(void)
{
    register rt_base_t level;
    struct rt_thread *thread;

    /* disable interrupt */
    level = rt_hw_interrupt_disable();

    /* set to current thread */
    thread = rt_current_thread;

    /* if the thread stat is READY and on ready queue list */
   （1） if (thread->stat == RT_THREAD_READY &&
        thread->tlist.next != thread->tlist.prev)
    {
        /* remove thread from thread list */
       （2） rt_list_remove(&(thread->tlist));

        /* put thread to end of ready queue */
       （3）rt_list_insert_before(&(rt_thread_priority_table[thread->current_priority]),
                              &(thread->tlist));

        /* enable interrupt */
        rt_hw_interrupt_enable(level);

        rt_schedule();

        return RT_EOK;
    }

    /* enable interrupt */
    rt_hw_interrupt_enable(level);

    return RT_EOK;
}
RTM_EXPORT(rt_thread_yield);
```
从rt_thread_yield()函数可以看出执行完时间片的任务只要其仍然处于就绪状态就从该优先级的链表头部中取出放到链表尾部(看(1),(2),(3)处代码)然后执行任务切换函数(看rt_schedule()函数，rt_schedule()执行的是基于位图的最高优先级任务查找与调度)，在rt_schedule()函数中CPU会切换到最高优先级的任务运行，如果没有更高优先级的任务就绪，那么调度器会在原先的优先级中执行下一个任务，运行至指定的时间片后重复相同的动作。

## 参考文献
[1]. 维基百科.http://zh.wikipedia.org/zh-cn/%E5%AE%9E%E6%97%B6%E6%93%8D%E4%BD%9C%E7%B3%BB%E7%BB%9F

[2]. RT-Thread编程指南

[3]. GitHub.https://github.com/RT-Thread
