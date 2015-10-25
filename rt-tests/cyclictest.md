Cyclictest 是实时 Linux 中最广泛使用的基准测试工具，它的主要概念十分简单，然而其测试选项扩展性很强。Cyclictest 的测试结果看起来可以很简单，但也可以相当复杂,本文将介绍 Cyclictest 的方方面面以帮助读者理解为什么 Cyclictest 的测试结果可以代表一个系统的实时性能。

## Cyclictest 测量什么

简单来说 Cyclictest 就是测量系统响应刺激的延迟，这里的刺激指的是中断触发，时钟到期等等，那么造成延迟的因素有哪些呢？

关中断期间以及执行中断处理函数造成的延迟, Cyclictest 本身唤醒造成的延迟，任务抢占造成的延迟，不合理的设置 Cyclictest 为最高优先级造成的延迟，调度器本身的调度开销，Cyclictest 会将以上所列所有可能造成的延迟原因测量进去。

当然上面所列的造成延迟的情况还较为理想，因为这些情况发生的顺序不同，每个情况可能发生若干次，并且还有其他因素可能会造成延迟。比如响应外部刺激时处理器从睡眠状态唤醒带来的延迟，由于唤醒任务造成的 cache 冲刷带来的延迟，待唤醒的任务阻塞在睡眠锁上造成的延迟(在实时系统中若有一个任务想要获得一个锁，而另外一个任务已经占据了这个锁时，锁的拥有者可能会提高优先级而跟锁的申请者相同，直到锁的拥有者完成任务了才释放这个锁并恢复到原先的优先级)。

## Cyclictest 是如何测量的

Cyclictest 是 rt-tests 项目下的一个主要工具，最初由 Thomas Gleixner 开发,现在由 RedHat 的 Clark Williams 和 John Kacur 来维护，最新代码可从 [kernel.org](https://git.kernel.org/cgit/utils/rt-tests/rt-tests.git/) 获取。

Cyclictest 代码行数超过了 2000 行，然而他的算法却是比较简单的，可用如下的伪码表示：

```
test_loop(){
clock_gettime(&now)
next = now + par->interval

while(!shutdown)
{
    clock_nanosleep(&next)
    clock_gettime(&now)
    diff = calcdiff(now, next)
    
    #update stat->min,max,total latency...
    
    next += par->interval
}
}
```

## Cyclictest 存在的问题一

这种算法能够将造成延迟的总总因素考虑进去,然而接下来会马上指出它的不足。

下面用伪码简单表示下整个程序,分为 main(), timerthread(),thread\_setup(),test loop() 四个程序片段：

```
main()
{
    for(i = 0; i < num_threads; i++)
    {
        pthread_create(timerthread)
    } 
    while(!shutdown)
    {
        for(i = 0; i < num_threads; i++)
            print_stat(stats[i], i)
        usleep(10000)
    }
}
```

```
timerthread(void *par)
{
    thread_setup()
    test_loop()
}
```

```
thread_setup()
{
pthread_setaffinity_np(pthread_self())
setscheduler({par->policy, par->priority})
}
```

test\_loop() 函数如先前例子所示。

这段程序问题在哪里呢：为能够造成更大的并发可能性，多个定时器线程应该同时启动，而非在 Cyclictest 中所展示的那样多个线程相继启动，并且多个线程独立运行并不可能出现相互干扰的情况。

所以在 2013-09-09 Nicholas Mc Guire 提交了一个补丁修复了该问题：

```
This patch provides additional -A/--align flag to cyclictest to align thread wakeup time of all threads as closly defined as possible.

We need both.
same period + "random" start time
same period + synced start time

it makes a difference on some boxes that is significant.
```

## Cyclictest 存在的问题二

测试出来的最大延迟只是可能的最大延迟的最小值，可基于如下理由判断：
- 由于定时器中断的发生，一些可能的延迟就结束了，是因为中断具有最高优先级能抢占其他任务运行。
- 由于定时器中断处理函数都在中断上下中，而实时系统中执行中断是在一个很短的中断上下文中唤醒中断线程让实时任务在进程上下文中执行，因而 Cyclictest 这种方式不是实时系统中最典型的方式
- Cyclictest 没有将其他的实时和非实时任务干扰产生的延迟测量在内，如模块装载，卸载，其他核的 hotplug。

如何解决上述问题呢？

- 不要使用 Cyclictest，而是自己编写实时程序，并使用示波器等测量工具来测量延迟
- 增加实时程序和非实时程序作为系统的负载,运行多组实验，每组实验中 Cyclictest 的优先级不同来测量不同情景下的延迟。
