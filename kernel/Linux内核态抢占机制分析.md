# Linux内核态抢占机制分析

**【摘要】**本文首先介绍非抢占式内核(Non-Preemptive Kernel)和可抢占式内核(Preemptive Kernel)的区别。接着分析Linux下有两种抢占：用户态抢占(User Preemption)、内核态抢占(Kernel Preemption)。然后分析了在内核态下：如何判断能否抢占内核(什么是可抢占的条件)；何时触发重新调度(何时设置可抢占条件)；抢占发生的时机(何时检查可抢占的条件)；什么时候不能抢占内核。最后分析了2.6kernel中如何支持抢占内核。

**【关键字】**内核态抢占 用户态抢占 中断 实时性 自旋锁 linux kernel schedule preemption reentrant

## 1 非抢占式和可抢占式内核的区别

为了简化问题，我使用嵌入式实时系统uC/OS作为例子。首先要指出的是，uC/OS只有内核态，没有用户态，这和Linux不一样。

多任务系统中，内核负责管理各个任务，或者说为每个任务分配CPU时间，并且负责任务之间的通讯。内核提供的基本服务是任务切换。调度（Scheduler）,英文还有一词叫dispatcher，也是调度的意思。这是内核的主要职责之一，就是要决定该轮到哪个任务运行了。多数实时内核是基于优先级调度法的。每个任务根据其重要程度的不同被赋予一定的优先级。基于优先级的调度法指，CPU总是让处在就绪态的优先级最高的任务先运行。然而，究竟何时让高优先级任务掌握CPU的使用权，有两种不同的情况，这要看用的是什么类型的内核，是不可剥夺型的还是可剥夺型内核

**非抢占式内核**

非抢占式内核是由任务主动放弃CPU的使用权。非抢占式调度法也称作合作型多任务，各个任务彼此合作共享一个CPU。异步事件还是由中断服务来处理。中断服务可以使一个高优先级的任务由挂起状态变为就绪状态。但中断服务以后控制权还是回到原来被中断了的那个任务，直到该任务主动放弃CPU的使用权时，那个高优先级的任务才能获得CPU的使用权。非抢占式内核如下图所示。

[图1]


非抢占式内核的优点有：

* 中断响应快(与抢占式内核比较)；
* 允许使用不可重入函数；
* 几乎不需要使用信号量保护共享数据。运行的任务占有CPU，不必担心被别的任务抢占。这不是绝对的，在打印机的使用上，仍需要满足互斥条件。
* 非抢占式内核的缺点有：
* 任务响应时间慢。高优先级的任务已经进入就绪态，但还不能运行，要等到当前运行着的任务释放CPU。
* 非抢占式内核的任务级响应时间是不确定的，不知道什么时候最高优先级的任务才能拿到CPU的控制权，完全取决于应用程序什么时候释放CPU。


**抢占式内核**

使用抢占式内核可以保证系统响应时间。最高优先级的任务一旦就绪，总能得到CPU的使用权。当一个运行着的任务使一个比它优先级高的任务进入了就绪态，当前任务的CPU使用权就会被剥夺，或者说被挂起了，那个高优先级的任务立刻得到了CPU的控制权。如果是中断服务子程序使一个高优先级的任务进入就绪态，中断完成时，中断了的任务被挂起，优先级高的那个任务开始运行。抢占式内核如下图所示。

[图2]


抢占式内核的优点有：

* 使用抢占式内核，最高优先级的任务什么时候可以执行，可以得到CPU的使用权是可知的。使用抢占式内核使得任务级响应时间得以最优化。

抢占式内核的缺点有：

* 不能直接使用不可重入型函数。调用不可重入函数时，要满足互斥条件，这点可以使用互斥型信号量来实现。如果调用不可重入型函数时，低优先级的任务CPU的使用权被高优先级任务剥夺，不可重入型函数中的数据有可能被破坏。

## 2 Linux下的用户态抢占和内核态抢占

Linux除了内核态外还有用户态。用户程序的上下文属于用户态，系统调用和中断处理例程上下文属于内核态。在2.6 kernel以前，Linux kernel只支持用户态抢占。

### 2.1 用户态抢占(User Preemption)

在kernel返回用户态(user-space)时，并且need\_resched标志为1时，scheduler被调用，这就是用户态抢占。当kernel返回用户态时，系统可以安全的执行当前的任务，或者切换到另外一个任务。当中断处理例程或者系统调用完成后，kernel返回用户态时，need\_reched标志的值会被检查，假如它为1，调度器会选择一个新的任务并执行。中断和系统调用的返回路径(return path)的实现在entry.S中(entry.S不仅包括kernel entry code，也包括kernel exit code)。

### 2.2 内核态抢占(Kernel Preemption)

在2.6 kernel以前，kernel code(中断和系统调用属于kernel code)会一直运行，直到code被完成或者被阻塞(系统调用可以被阻塞)。在 2.6 kernel里，Linux kernel变成可抢占式。当从中断处理例程返回到内核态(kernel-space)时，kernel会检查是否可以抢占和是否需要重新调度。kernel可以在任何时间点上抢占一个任务(因为中断可以发生在任何时间点上)，只要在这个时间点上kernel的状态是安全的、可重新调度的。


## 3 内核态抢占的设计

### 3.1 可抢占的条件

要满足什么条件，kernel才可以抢占一个任务的内核态呢？


* 没持有锁。锁是用于保护临界区的，不能被抢占。
* Kernel code可重入(reentrant)。因为kernel是SMP-safe的，所以满足可重入性。

如何判断当前上下文(中断处理例程、系统调用、内核线程等)是没持有锁的？Linux在每个每个任务的thread\_info结构中增加了preempt\_count变量作为preemption的计数器。这个变量初始为0，当加锁时计数器增一，当解锁时计数器减一。

### 3.2 内核态需要抢占的触发条件

内核提供了一个need\_resched标志(这个标志在任务结构thread\_info中)来表明是否需要重新执行调度。


### 3.3 何时触发重新调度

* set\_tsk\_need\_resched()：设置指定进程中的need\_resched标志
* clear\_tsk need\_resched()：清除指定进程中的need\_resched标志
* need\_resched()：检查need\_ resched标志的值;如果被设置就返回真，否则返回假

什么时候需要重新调度：

* 时钟中断处理例程检查当前任务的时间片，当任务的时间片消耗完时，scheduler\_tick()函数就会设置need\_resched标志；
* 信号量、等到队列、completion等机制唤醒时都是基于waitqueue的，而waitqueue的唤醒函数为default\_wake\_function，其调用try\_to\_wake\_up将被唤醒的任务更改为就绪状态并设置need\_resched标志。
* 设置用户进程的nice值时，可能会使高优先级的任务进入就绪状态；
* 改变任务的优先级时，可能会使高优先级的任务进入就绪状态；
* 新建一个任务时，可能会使高优先级的任务进入就绪状态；
* 对CPU(SMP)进行负载均衡时，当前任务可能需要放到另外一个CPU上运行；


### 3.4 抢占发生的时机(何时检查可抢占条件)

* 当一个中断处理例程退出，在返回到内核态时(kernel-space)。这是隐式的调用schedule()函数，当前任务没有主动放弃CPU使用权，而是被剥夺了CPU使用权。
* 当kernel code从不可抢占状态变为可抢占状态时(preemptible again)。也就是preempt\_count从正整数变为0时。这也是隐式的调用schedule()函数。
* 一个任务在内核态中显式的调用schedule()函数。任务主动放弃CPU使用权。
* 一个任务在内核态中被阻塞，导致需要调用schedule()函数。任务主动放弃CPU使用权。


### 3.5 禁用/使能可抢占条件的操作

* 对preempt\_count操作的函数有add\_preempt\_count()、sub\_preempt\_count()、inc\_preempt\_count()、dec\_preempt\_count()。
* 使能可抢占条件的操作是preempt\_enable()，它调用dec\_preempt\_count()函数，然后再调用preempt\_check\_resched()函数去检查是否需要重新调度。
* 禁用可抢占条件的操作是preempt\_disable()，它调用inc\_preempt\_count()函数。
* 在内核中有很多函数调用了preempt\_enable()和preempt\_disable()。比如spin\_lock()函数调用了preempt\_disable()函数，spin\_unlock()函数调用了preempt\_enable()函数。

### 3.6 什么时候不允许抢占

preempt\_count()函数用于获取preempt\_count的值，preemptible()用于判断内核是否可抢占。
有几种情况Linux内核不应该被抢占，除此之外，Linux内核在任意一点都可被抢占。这几种情况是：

* 内核正进行中断处理。在Linux内核中进程不能抢占中断(中断只能被其他中断中止、抢占，进程不能中止、抢占中断)，在中断例程中不允许进行进程调度。进程调度函数schedule()会对此作出判断，如果是在中断中调用，会打印出错信息。
* 内核正在进行中断上下文的Bottom Half(中断的下半部)处理。硬件中断返回前会执行软中断，此时仍然处于中断上下文中。
* 内核的代码段正持有spinlock自旋锁、writelock/readlock读写锁等锁，处干这些锁的保护状态中。内核中的这些锁是为了在SMP系统中短时间内保证不同CPU上运行的进程并发执行的正确性。当持有这些锁时，内核不应该被抢占，否则由于抢占将导致其他CPU长期不能获得锁而死等。
* 内核正在执行调度程序Scheduler。抢占的原因就是为了进行新的调度，没有理由将调度程序抢占掉再运行调度程序。
* 内核正在对每个CPU“私有”的数据结构操作(Per-CPU date structures)。在SMP中，对于per-CPU数据结构未用spinlocks保护，因为这些数据结构隐含地被保护了(不同的CPU有不一样的per-CPU数据，其他CPU上运行的进程不会用到另一个CPU的per-CPU数据)。但是如果允许抢占，但一个进程被抢占后重新调度，有可能调度到其他的CPU上去，这时定义的Per-CPU变量就会有问题，这时应禁抢占。

## 4 Linux内核态抢占的实现
### 4.1 数据结构

在thread\_info.h中

<pre name="code" class="c">
    struct thread_info {  
        struct task_struct  *task;      /* main task structure */  
        struct exec_domain  *exec_domain;   /* execution domain */  
        __u32           flags;      /* low level flags */  
        __u32           status;     /* thread synchronous flags */  
        __u32           cpu;        /* current CPU */  
        int         preempt_count;  /* 0 => preemptable, 
                               <0 => BUG */  
        mm_segment_t        addr_limit;  
        struct restart_block    restart_block;  
        void __user     *sysenter_return;  
    #ifdef CONFIG_X86_32  
        unsigned long           previous_esp;   /* ESP of the previous stack in 
                               case of nested (IRQ) stacks 
                            */  
        __u8            supervisor_stack[0];  
    #endif  
    };  
</pre>

### 4.2 代码流程

禁用/使能可抢占条件的函数

<pre name="code" class="c">
#if defined(CONFIG_DEBUG_PREEMPT) || defined(CONFIG_PREEMPT_TRACER)  
  
extern void add_preempt_count(int val);  
  
extern void sub_preempt_count(int val);  
  
#else  
  
# define add_preempt_count(val) do { preempt_count() += (val); } while (0)  
  
# define sub_preempt_count(val) do { preempt_count() -= (val); } while (0)  
  
#endif  
  
#define inc_preempt_count() add_preempt_count(1)  
  
#define dec_preempt_count() sub_preempt_count(1)  
  
#define preempt_count() (current_thread_info()->preempt_count)  
  
#define preempt_disable() \  
  
do { \  
  
    inc_preempt_count(); \  
  
    barrier(); \  
  
} while (0)  
  
#define preempt_enable_no_resched() \  
  
do { \  
  
    barrier(); \  
  
    dec_preempt_count(); \  
  
} while (0)  
  
#define preempt_check_resched() \  
  
do { \  
  
    if (unlikely(test_thread_flag(TIF_NEED_RESCHED))) \  
  
    preempt_schedule(); \  
  
} while (0)  
  
#define preempt_enable() \  
  
do { \  
  
    preempt_enable_no_resched(); \  
  
    barrier(); \  
  
    preempt_check_resched(); \  
  
} while (0)  

//检查可抢占条件  

# define preemptible() (preempt_count() == 0 && !irqs_disabled())  
   
//自旋锁的加锁与解锁

void __lockfunc _spin_lock(spinlock_t *lock)  
{  
    preempt_disable();  
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);  
    LOCK_CONTENDED(lock, _raw_spin_trylock, _raw_spin_lock);  
}  
  
void __lockfunc _spin_unlock(spinlock_t *lock)  
{  
    spin_release(&lock->dep_map, 1, _RET_IP_);  
    _raw_spin_unlock(lock);  
    preempt_enable();  
}  

//设置need_resched标志的函数

static inline void set_tsk_need_resched(struct task_struct *tsk)  
{  
    set_tsk_thread_flag(tsk,TIF_NEED_RESCHED);  
}  
  
static inline void clear_tsk_need_resched(struct task_struct *tsk)  
{  
    clear_tsk_thread_flag(tsk,TIF_NEED_RESCHED);  
}  
  
static inline int test_tsk_need_resched(struct task_struct *tsk)  
{  
    return unlikely(test_tsk_thread_flag(tsk,TIF_NEED_RESCHED));  
}

//时钟中断时调用的task_tick()函数，当时间片消耗完之后，设置need_resched标志  

static void task_tick_rt(struct rq *rq, struct task_struct *p, int queued)  
{  
    update_curr_rt(rq);  
  
    watchdog(rq, p);  
  
    /* 
     * RR tasks need a special form of timeslice management. 
     * FIFO tasks have no timeslices. 
     */  
    if (p->policy != SCHED_RR)  
        return;  
  
    if (--p->rt.time_slice)  
        return;  
  
    p->rt.time_slice = DEF_TIMESLICE;  
  
    /* 
     * Requeue to the end of queue if we are not the only element 
     * on the queue: 
     */  
    if (p->rt.run_list.prev != p->rt.run_list.next) {  
        requeue_task_rt(rq, p, 0);  
        set_tsk_need_resched(p);  
    }  
}

//设置任务的need_resched标志，并触发任务所在CPU的调度器。

static void resched_task(struct task_struct *p)  
{  
    int cpu;  
  
    assert_spin_locked(&task_rq(p)->lock);  
  
    if (unlikely(test_tsk_thread_flag(p, TIF_NEED_RESCHED)))  
        return;  
  
    set_tsk_thread_flag(p, TIF_NEED_RESCHED);  
  
    cpu = task_cpu(p);  
    if (cpu == smp_processor_id())  
        return;  
  
    /* NEED_RESCHED must be visible before we test polling */  
    smp_mb();  
    if (!tsk_is_polling(p))  
        smp_send_reschedule(cpu);  
}
</pre>

## 参考资料

* http://blog.csdn.net/sailor_8318/archive/2008/09/03/2870184.aspx  
* uC/OS-II源码公开的嵌入式实时多任务操作系统内核
* Linux 2.6.29内核源码



