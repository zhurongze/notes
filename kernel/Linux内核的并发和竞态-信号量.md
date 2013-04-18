
# 在include/linux/semaphore.h中


<pre name="code" class="c"> 
/* Please don't access any members of this structure directly */
struct semaphore {
    spinlock_t        lock;
    unsigned int        count;
    struct list_head    wait_list;
};

#define __SEMAPHORE_INITIALIZER(name, n)                \
{                                    \
    .lock        = __SPIN_LOCK_UNLOCKED((name).lock),        \
    .count        = n,                        \
    .wait_list    = LIST_HEAD_INIT((name).wait_list),        \
}

#define DECLARE_MUTEX(name)    \
    struct semaphore name = __SEMAPHORE_INITIALIZER(name, 1)

static inline void sema_init(struct semaphore *sem, int val)
{
    static struct lock_class_key __key;
    *sem = (struct semaphore) __SEMAPHORE_INITIALIZER(*sem, val);
    lockdep_init_map(&sem->lock.dep_map, "semaphore->lock", &__key, 0);
}

#define init_MUTEX(sem)        sema_init(sem, 1)
#define init_MUTEX_LOCKED(sem)    sema_init(sem, 0)
</pre> 
 
# 在kernel/semaphore.c中


<pre name="code" class="c"> 
/**
 * down - acquire the semaphore
 * @sem: the semaphore to be acquired
 *
 * Acquires the semaphore. If no more tasks are allowed to acquire the
 * semaphore, calling this function will put the task to sleep until the
 * semaphore is released.
 *
 * Use of this function is deprecated, please use down_interruptible() or
 * down_killable() instead.
 */
void down(struct semaphore *sem)
{
    unsigned long flags;

    spin_lock_irqsave(&sem->lock, flags);
    if (likely(sem->count > 0))
        sem->count--;
    else
        __down(sem);
    spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(down);

/**
 * down_interruptible - acquire the semaphore unless interrupted
 * @sem: the semaphore to be acquired
 *
 * Attempts to acquire the semaphore. If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the sleep is interrupted by a signal, this function will return -EINTR.
 * If the semaphore is successfully acquired, this function returns 0.
 */
int down_interruptible(struct semaphore *sem)
{
    unsigned long flags;
    int result = 0;

    spin_lock_irqsave(&sem->lock, flags);
    if (likely(sem->count > 0))
        sem->count--;
    else
        result = __down_interruptible(sem);
    spin_unlock_irqrestore(&sem->lock, flags);

    return result;
}
EXPORT_SYMBOL(down_interruptible);

/**
 * down_killable - acquire the semaphore unless killed
 * @sem: the semaphore to be acquired
 *
 * Attempts to acquire the semaphore. If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the sleep is interrupted by a fatal signal, this function will return
 * -EINTR. If the semaphore is successfully acquired, this function returns
 * 0.
 */
int down_killable(struct semaphore *sem)
{
    unsigned long flags;
    int result = 0;

    spin_lock_irqsave(&sem->lock, flags);
    if (likely(sem->count > 0))
        sem->count--;
    else
        result = __down_killable(sem);
    spin_unlock_irqrestore(&sem->lock, flags);

    return result;
}
EXPORT_SYMBOL(down_killable);

/**
 * down_trylock - try to acquire the semaphore, without waiting
 * @sem: the semaphore to be acquired
 *
 * Try to acquire the semaphore atomically. Returns 0 if the mutex has
 * been acquired successfully or 1 if it it cannot be acquired.
 *
 * NOTE: This return value is inverted from both spin_trylock and
 * mutex_trylock! Be careful about this when converting code.
 *
 * Unlike mutex_trylock, this function can be used from interrupt context,
 * and the semaphore can be released by any task or interrupt.
 */
int down_trylock(struct semaphore *sem)
{
    unsigned long flags;
    int count;

    spin_lock_irqsave(&sem->lock, flags);
    count = sem->count - 1;
    if (likely(count >= 0))
        sem->count = count;
    spin_unlock_irqrestore(&sem->lock, flags);

    return (count < 0);
}
EXPORT_SYMBOL(down_trylock);

/**
 * down_timeout - acquire the semaphore within a specified time
 * @sem: the semaphore to be acquired
 * @jiffies: how long to wait before failing
 *
 * Attempts to acquire the semaphore. If no more tasks are allowed to
 * acquire the semaphore, calling this function will put the task to sleep.
 * If the semaphore is not released within the specified number of jiffies,
 * this function returns -ETIME. It returns 0 if the semaphore was acquired.
 */
int down_timeout(struct semaphore *sem, long jiffies)
{
    unsigned long flags;
    int result = 0;

    spin_lock_irqsave(&sem->lock, flags);
    if (likely(sem->count > 0))
        sem->count--;
    else
        result = __down_timeout(sem, jiffies);
    spin_unlock_irqrestore(&sem->lock, flags);

    return result;
}
EXPORT_SYMBOL(down_timeout);

/**
 * up - release the semaphore
 * @sem: the semaphore to release
 *
 * Release the semaphore. Unlike mutexes, up() may be called from any
 * context and even by tasks which have never called down().
 */
void up(struct semaphore *sem)
{
    unsigned long flags;

    spin_lock_irqsave(&sem->lock, flags);
    if (likely(list_empty(&sem->wait_list)))
        sem->count++;
    else
        __up(sem);
    spin_unlock_irqrestore(&sem->lock, flags);
}
EXPORT_SYMBOL(up);

/* Functions for the contended case */

struct semaphore_waiter {
    struct list_head list;
    struct task_struct *task;
    int up;
};

/*
 * Because this function is inlined, the 'state' parameter will be
 * constant, and thus optimised away by the compiler. Likewise the
 * 'timeout' parameter for the cases without timeouts.
 */
static inline int __sched __down_common(struct semaphore *sem, long state,
                                long timeout)
{
    struct task_struct *task = current;
    struct semaphore_waiter waiter;

    list_add_tail(&waiter.list, &sem->wait_list);
    waiter.task = task;
    waiter.up = 0;

    for (;;) {
        if (signal_pending_state(state, task))
            goto interrupted;
        if (timeout <= 0)
            goto timed_out;
        __set_task_state(task, state);
        spin_unlock_irq(&sem->lock);
        timeout = schedule_timeout(timeout);
        spin_lock_irq(&sem->lock);
        if (waiter.up)
            return 0;
    }

 timed_out:
    list_del(&waiter.list);
    return -ETIME;

 interrupted:
    list_del(&waiter.list);
    return -EINTR;
}

static noinline void __sched __down(struct semaphore *sem)
{
    __down_common(sem, TASK_UNINTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}

static noinline int __sched __down_interruptible(struct semaphore *sem)
{
    return __down_common(sem, TASK_INTERRUPTIBLE, MAX_SCHEDULE_TIMEOUT);
}

static noinline int __sched __down_killable(struct semaphore *sem)
{
    return __down_common(sem, TASK_KILLABLE, MAX_SCHEDULE_TIMEOUT);
}

static noinline int __sched __down_timeout(struct semaphore *sem, long jiffies)
{
    return __down_common(sem, TASK_UNINTERRUPTIBLE, jiffies);
}

static noinline void __sched __up(struct semaphore *sem)
{
    struct semaphore_waiter *waiter = list_first_entry(&sem->wait_list,
                        struct semaphore_waiter, list);
    list_del(&waiter->list);
    waiter->up = 1;
    wake_up_process(waiter->task);
}
</pre> 
