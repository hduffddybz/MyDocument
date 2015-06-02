# ���������ֵ�ź���������RT-ThreadΪ����

��������һ��������ź���������ֵ�ź������������������߲�����һ���¡���������һ��������RT-Thread����һ���̣߳���ӵ�У�����һ��������ֻ����ӵ���߲����ͷţ�����֧�ֻ������ݹ���ʣ���ֹ���ȼ���ת�����ԡ�

����������RTT���ź����������ṹ����ʲô����

## �ź����ṹ�壺

``` c

/**
 * Semaphore structure
 */
struct rt_semaphore
{
    struct rt_ipc_object parent;                        /**< inherit from ipc_object */

    rt_uint16_t          value;                         /**< value of semaphore. */
};
typedef struct rt_semaphore *rt_sem_t;

```

## �������ṹ�壺

``` c

/**
 * Mutual exclusion (mutex) structure
 */
struct rt_mutex
{
    struct rt_ipc_object parent;                        /**< inherit from ipc_object */

    rt_uint16_t          value;                         /**< value of mutex */

    rt_uint8_t           original_priority;             /**< priority of last thread hold the mutex */
    rt_uint8_t           hold;                          /**< numbers of thread hold the mutex */

    struct rt_thread    *owner;                         /**< current owner of mutex */
};
typedef struct rt_mutex *rt_mutex_t;

```

## ���������ź������̳е�ipc�ṹ�����£�

``` c

struct rt_ipc_object
{
    struct rt_object parent;                            /**< inherit from rt_object */

    rt_list_t        suspend_thread;                    /**< threads pended on this resource */
};

```

�����Ե��ǻ������ṹ����ź����ṹ������orginal_priority,��owener���ֶΣ�����original_priority�����Ƿ�ֹ���ȼ���ת֮�ã���owener�ֶ�ָ���Ǹû������������ߡ�

����mutex�ǿ�����ģ���semaphore�ǲ�������ģ�����һ���߳������һ����ֵ�ź������еݹ���лᵼ�µ�ǰ�̹߳��𡣿��Կ���rt_sem_take()�е�ʵ�֣�

``` c

rt_err_t rt_sem_take(rt_sem_t sem, rt_int32_t time)
{
    register rt_base_t temp;
    struct rt_thread *thread;

    RT_ASSERT(sem != RT_NULL);

    RT_OBJECT_HOOK_CALL(rt_object_trytake_hook, (&(sem->parent.parent)));

    /* disable interrupt */
    temp = rt_hw_interrupt_disable();

    RT_DEBUG_LOG(RT_DEBUG_IPC, ("thread %s take sem:%s, which value is: %d\n",
                                rt_thread_self()->name,
                                ((struct rt_object *)sem)->name,
                                sem->value));

    if (sem->value > 0)
    {
        /* semaphore is available */
        sem->value --;

        /* enable interrupt */
        rt_hw_interrupt_enable(temp);
    }
    else
    {
        /* no waiting, return with timeout */
        if (time == 0)
        {
            rt_hw_interrupt_enable(temp);

            return -RT_ETIMEOUT;
        }
        else
        {
            /* current context checking */
            RT_DEBUG_IN_THREAD_CONTEXT;

            /* semaphore is unavailable, push to suspend list */
            /* get current thread */
            thread = rt_thread_self();

            /* reset thread error number */
            thread->error = RT_EOK;

            RT_DEBUG_LOG(RT_DEBUG_IPC, ("sem take: suspend thread - %s\n",
                                        thread->name));

            /* suspend thread */
            rt_ipc_list_suspend(&(sem->parent.suspend_thread),
                                thread,
                                sem->parent.parent.flag);

            /* has waiting time, start thread timer */
            if (time > 0)
            {
                RT_DEBUG_LOG(RT_DEBUG_IPC, ("set thread:%s to timer list\n",
                                            thread->name));

                /* reset the timeout of thread timer and start it */
                rt_timer_control(&(thread->thread_timer),
                                 RT_TIMER_CTRL_SET_TIME,
                                 &time);
                rt_timer_start(&(thread->thread_timer));
            }

            /* enable interrupt */
            rt_hw_interrupt_enable(temp);

            /* do schedule */
            rt_schedule();

            if (thread->error != RT_EOK)
            {
                return thread->error;
            }
        }
    }

    RT_OBJECT_HOOK_CALL(rt_object_take_hook, (&(sem->parent.parent)));

    return RT_EOK;
}
RTM_EXPORT(rt_sem_take);

```

��һ���̻߳�ȡһ����ֵ�ź���ʱ��value����ٵ�0�������߳��ٴλ�ȡ���ź���ʱ������vauleֵС��0ʱ���Ὣ��ǰ�̹߳��𣬲�������뵽�ȴ������У����ڸ��ź������ᱻ�ͷţ����Ը��̻߳�һֱִ�У�ֱ����ʱʱ�䵽
���ڸ��߳�Ϊ����״̬�������޷�������������У����Ը��߳̽��޷��õ����С�

���������Ĳ����Ͳ��������ˣ�

``` c

rt_err_t rt_mutex_take(rt_mutex_t mutex, rt_int32_t time)
{
    register rt_base_t temp;
    struct rt_thread *thread;

    /* this function must not be used in interrupt even if time = 0 */
    RT_DEBUG_IN_THREAD_CONTEXT;

    RT_ASSERT(mutex != RT_NULL);

    /* disable interrupt */
    temp = rt_hw_interrupt_disable();

    /* get current thread */
    thread = rt_thread_self();

    RT_OBJECT_HOOK_CALL(rt_object_trytake_hook, (&(mutex->parent.parent)));

    RT_DEBUG_LOG(RT_DEBUG_IPC,
                 ("mutex_take: current thread %s, mutex value: %d, hold: %d\n",
                  thread->name, mutex->value, mutex->hold));

    /* reset thread error */
    thread->error = RT_EOK;

    if (mutex->owner == thread)
    {
        /* it's the same thread */
        mutex->hold ++;
    }
    else
    {
        /* The value of mutex is 1 in initial status. Therefore, if the
         * value is great than 0, it indicates the mutex is avaible.
         */
        if (mutex->value > 0)
        {
            /* mutex is available */
            mutex->value --;

            /* set mutex owner and original priority */
            mutex->owner             = thread;
            mutex->original_priority = thread->current_priority;
            mutex->hold ++;
        }
        else
        {
            /* no waiting, return with timeout */
            if (time == 0)
            {
                /* set error as timeout */
                thread->error = -RT_ETIMEOUT;

                /* enable interrupt */
                rt_hw_interrupt_enable(temp);

                return -RT_ETIMEOUT;
            }
            else
            {
                /* mutex is unavailable, push to suspend list */
                RT_DEBUG_LOG(RT_DEBUG_IPC, ("mutex_take: suspend thread: %s\n",
                                            thread->name));

                /* change the owner thread priority of mutex */
                if (thread->current_priority < mutex->owner->current_priority)
                {
                    /* change the owner thread priority */
                    rt_thread_control(mutex->owner,
                                      RT_THREAD_CTRL_CHANGE_PRIORITY,
                                      &thread->current_priority);
                }

                /* suspend current thread */
                rt_ipc_list_suspend(&(mutex->parent.suspend_thread),
                                    thread,
                                    mutex->parent.parent.flag);

                /* has waiting time, start thread timer */
                if (time > 0)
                {
                    RT_DEBUG_LOG(RT_DEBUG_IPC,
                                 ("mutex_take: start the timer of thread:%s\n",
                                  thread->name));

                    /* reset the timeout of thread timer and start it */
                    rt_timer_control(&(thread->thread_timer),
                                     RT_TIMER_CTRL_SET_TIME,
                                     &time);
                    rt_timer_start(&(thread->thread_timer));
                }

                /* enable interrupt */
                rt_hw_interrupt_enable(temp);

                /* do schedule */
                rt_schedule();

                if (thread->error != RT_EOK)
                {
                    /* return error */
                    return thread->error;
                }
                else
                {
                    /* the mutex is taken successfully. */
                    /* disable interrupt */
                    temp = rt_hw_interrupt_disable();
                }
            }
        }
    }

    /* enable interrupt */
    rt_hw_interrupt_enable(temp);

    RT_OBJECT_HOOK_CALL(rt_object_take_hook, (&(mutex->parent.parent)));

    return RT_EOK;
}
RTM_EXPORT(rt_mutex_take);

```

��Ҫ�������������øû��������߳��Ǹû�������ӵ����ʱ��Ҳ����һ���̵߳ݹ���л�����������ô��ȡ�Ĳ����ǽ�holdֵ��1��
���ͷ�����ʱ��ֻ�е�holdֵ���ٵ�0��ʱ���������ִ��valueֵ��1�Ķ�����

��Ȼmutex���и�������ԣ�����֧�����ȼ��̳�Э�飬�����д���if(thread->current_priority < mutex->owener->current_priority),
�������ľ��ǽ������ȼ���ӵ�л��������߳���ߵ������ȼ�����Ҫ�ù�����Դ���߳����С�