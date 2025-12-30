# [Softlockup&Hardlockup检测机制](https://www.cnblogs.com/wodemia/p/17967979)
Linux自身具备一定的异常检测机制，softlockup和hardlockup是典型的两种，softlockup检测内核是否出现了长时间不调度其他任务执行的异常情况。hardlockup则更进一步检测内核是否出现了长时间不响应中断的异常情况。softlockup和hardlockup的定义如下：

    A 'softlockup' is defined as a bug that causes the kernel to loop in kernel mode for more than 20 seconds, without giving other tasks a chance to run.
    A 'hardlockup' is defined as a bug that causes the CPU to loop in kernel mode for more than 10 seconds, without letting other interrupts have a chance to run.

这两种异常检测机制具有一定的相似性，因此设计的思路是一体的。但是在检测的目标上又存在差异，所以实现上有一些不同。
## watchdog
watchdog机制是一种常见的keep-alive方法，其原理是周期性的执行一个任务检查某个值是否已经更新，这个检查过程称之为watch dog，而更新值的动作被称为touch dog。
softlockup和hardlockup机制针对的是单核的检测，因此对于每一个CPU内核都有两个dog分别对应softlockup和hardlockup。
- softlockup的dog是watchdog_touch_ts，记录了上一次touch dog的时间戳。
- hardlockup的dog是hrtimer_interrupts，记录hrtimer高精度定时器中断发生的次数。

```
static DEFINE_PER_CPU(unsigned long, watchdog_touch_ts);
static DEFINE_PER_CPU(unsigned long, hrtimer_interrupts);
```
在内核中存在三类程序可以被执行的，按照优先级从高到底分别是NMI处理函数、Normal Interrupt处理函数和Task。从本质上来说，softlockup检测的是NMI和Normal Interrupt正常响应的情况下，Task之间的调度能否正常发生，hardlockup检测的是NMI正常响应的情况下，Normal Interrupt能否正常响应和被调度执行。
Note：NMI作为不可屏蔽中断，保证了任何条件下都能执行。
### softlockup
为了满足检测目标，softlockup需要有一个内核线程能够touch dog（更新watchdog_touch_ns），并且该线程必须在softlockup检查时启动。同时还需要一个周期定时器任务，检查watchdog_touch_ts与now之间的距离是否超过门限，如果超过就认为发生了softlockup。默认超时时长softlockup_thresh是20s(2 * watchdog_thresh)。softlockup检查在is_softlockup（kernel/watchdog.c）中实现：

```
static int is_softlockup(unsigned long touch_ts)
{
    unsigned long now = get_timestamp();

    if ((watchdog_enabled & SOFT_WATCHDOG_ENABLED) && watchdog_thresh){
        /* Warn about unreasonable delays. */
        if (time_after(now, touch_ts + get_softlockup_thresh()))
            return now - touch_ts;
    }
    return 0;
}

```
为了保证softlockup的有效性，更新watchdog_touch_ns的Task必须拥有最高的任务优先级，否则即使正常发生调度低优先级任务也无法及时更新时间戳。因此在老的内核版本更新watch_touch_ns的Task是[watchdog/x]，随着STOP调度类（比实时任务的优先级更高）的引入，更新线程变成了[migration/x]。

migration线程作为内核中优先级最高的线程，负责内核热插拔、停止CPU运行等工作。migration线程管理了一个work_queue，当有任务需要执行时migration就会进入RUNNABLE状态等待调度，一旦发生调度migration一定能够拿到执行权更新watchdog_touch_ns，保证了softlockup检查的有效性。

而检查softlockup的任务必须交给优先级更高的中断，内核中的hrtimer可以周期性的触发中断，在hrtimer的处理函数watchdog_timer_fn中可以检查[migration/x]是否正常更新了watchdog_touch_ns，hrtimer定时器的触发周期是softlockup_thresh / 5（默认值是4s）。

softlockup检查机制的整体流程如下：

- hrtimer周期性的触发执行中断处理程序watchdog_timer_fn：
    1. 向work_queue插入任务softlockup_fn
    2. 检查watchdog_touch_ns是否异常
    3. 睡眠，等待下一次触发
- migration线程
    1. 被work_queue唤醒
    2. 检查队列，取出softlockup_fn执行
    3. 更新watchdog_touch_ns
    4. work_queue为空，进入睡眠

如果migration线程在任务队列中长时间没有被调度执行（核上的任务长时间的占据了CPU），则说明出现了softlockup异常，需要对现场进行dump。
                    
### hardlockup
hardlockup的检测机制和softlockup类似，但是检测的目标不同，hardlockup检测的是普通中断长时间不响应，hardlockup的检查在kernel/watchdog.c的is_hardlockup中实现，判断hrtimer_interrupts是否在进行递增，如果没有递增则认为发生了hardlockup。

```
/* watchdog detector functions */
bool is_hardlockup(void)
{
    unsigned long hrint = __this_cpu_read(hrtimer_interrupts);

    if (__this_cpu_read(hrtimer_interrupts_saved) == hrint)
        return true;

    __this_cpu_write(hrtimer_interrupts_saved, hrint);
    return false;
}
```
hardlockup的默认超时时长watchdog_thresh是10s，是softlockup的一半。和softlockup不一样的是hrtimer_interrupts没有记录时间戳信息，如何判断是否超时呢？

Linux使用的是周期性的NMI。基于perf subsystem的cycles事件，perf的counter可以设置溢出阈值，当perf event的发生次数达到阈值时会触发一次NMI中断，同时cycles与时间存在一定的关系，具体可以看kernel/watchdog.c的watchdog_nmi_enable函数。顺着调用链可以看到hardlockup_detector_event_create函数（在kernel/watchdog_hld.c中）调用了hw_nmi_get_sample_period（在arch/x86/kernel/apic/hw_nmi.c中），这个函数是一个体系结构相关的函数，在这里获取了cycles溢出的NMI中断的触发周期watchdog_thresh。

```
u64 hw_nmi_get_sample_period(int watchdog_thresh)
{
    return (u64)(cpu_khz) * 1000 * watchdog_thresh;
}
```
周期性的NMI触发执行回调函数进行watch（检查hrtimer_interrupts是否递增），hrtimer则负责定期的touch（增加hrtimer_interrupts）。

hardlockup和softlockup之间通过hrtimer产生了交集，所以hrtiemr的处理函数不仅要watch watchdog_touch_ts进行softlockup检查，同时还需要touch hrtimer_interrupts更新中断触发次数。



## 知识补充
### 迁移线程migration:
是每个处理器核对应的内核线程，主要用于执行进程迁移操作和负载均衡。每个处理器都有一个名为“migration/<cpu_id>”的迁移线程，它可以抢占所有其他进程，并处理迁移请求。
- 处理调度器发出的迁移请求，将进程迁移到目标处理器。 
- 执行主动负载均衡。 
- 提供API以支持工作分配，支持同步和异步方式。 
- 在Linux内核启动过程中创建。 
- 参与CPU负载均衡、软锁定检测等机制。

