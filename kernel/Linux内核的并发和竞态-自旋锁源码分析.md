
# 在include/linux/spinlock_types.h中
<pre name="code" class="c"> 
typedef struct{
    raw_spinlock_t  raw_lock;
    ...
}spinlock_t;
 
在arch/x86/include/asm/spinlock_types.h中
typedef struct raw_spinlock{
    unsigned int slock;            //初始值是0
}raw_spinlock_t;
 
在include/linux/spinlock.h中
#define  spin_lock(lock)    _spin_lock(lock)
#define spin_lock_bh(lock)  _spin_lock_bh(lock)
#define spin_lock_irq(lock) _spin_lock_irq(lock)
#if defined(CONFIG_SMP || defined(CONFIG_DEBUG_SPINLOCK))
#define spin_lock_irqsave(lock, flags)  \
    do{        \
        typecheck(unsigned long , flags);    \
        flags = _spin_lock_irqsave(lock);    \
    }while(0)
#endif
 
#define spin_unlock(lock) _spin_unlock(lock)
#define spin_unlock_bh(lock) _spin_unlock_bh(lock)
#define spin_unlock_irq(lock) _spin_unlock_irq(lock)
 
#define spin_unlock_irqsave(lock, flags) \
do{ \
typecheck(unsigned long , flags); \
 _spin_unlock_irqrestore(lock, flags); \
}while(0)
 </pre>
 
# 在kernel/spinlock.c中
 <pre name="code" class="c"> 
void __lockfunc _spin_lock(spinlock_t *lock)
{
    preempt_disable();
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    LOCK_CONTENDED(lock, _raw_spin_trylock, _raw_spin_lock);
}
 
void __lockfunc _spin_lock_bh(spinlock_t *lock)
{
    local_bh_disable();
    preempt_disable();
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    LOCK_CONTENDED(lock, _raw_spin_trylock, _raw_spin_lock);
}
 
void __lockfunc _spin_lock_irq(spinlock_t *lock)
{
    local_irq_disable();
    preempt_disable();
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    LOCK_CONTENDED(lock, _raw_spin_trylock, _raw_spin_lock);
}
 
unsigned long __lockfunc _spin_lock_irqsave(spinlock_t *lock)
{
    unsigned long flags;

    local_irq_save(flags);
    preempt_disable();
    spin_acquire(&lock->dep_map, 0, 0, _RET_IP_);
    /*
     * On lockdep we dont want the hand-coded irq-enable of
     * _raw_spin_lock_flags() code, because lockdep assumes
     * that interrupts are not re-enabled during lock-acquire:
     */
#ifdef CONFIG_LOCKDEP
    LOCK_CONTENDED(lock, _raw_spin_trylock, _raw_spin_lock);
#else
    _raw_spin_lock_flags(lock, &flags);
#endif
    return flags;
}
void __lockfunc _spin_unlock(spinlock_t *lock)
{
    spin_release(&lock->dep_map, 1, _RET_IP_);
    _raw_spin_unlock(lock);
    preempt_enable();
}
 
void __lockfunc _spin_unlock_bh(spinlock_t *lock)
{
    spin_release(&lock->dep_map, 1, _RET_IP_);
    _raw_spin_unlock(lock);
    preempt_enable_no_resched();
    local_bh_enable_ip((unsigned long)__builtin_return_address(0));
}
 
void __lockfunc _spin_unlock_irq(spinlock_t *lock)
{
    spin_release(&lock->dep_map, 1, _RET_IP_);
    _raw_spin_unlock(lock);
    local_irq_enable();
    preempt_enable();
}
 
void __lockfunc _spin_unlock_irqrestore(spinlock_t *lock, unsigned long flags)
{
    spin_release(&lock->dep_map, 1, _RET_IP_);
    _raw_spin_unlock(lock);
    local_irq_restore(flags);
    preempt_enable();
}
 </pre>
 
# 在include/linux/lockdep.h中

<pre name="code" class="c"> 
#define LOCK_CONTENDED(_lock, try, lock) \
    lock(_lock)
 </pre>
 
# 在linux/spinlock.h中

<pre name="code" class="c"> 
# define _raw_spin_lock(lock)        __raw_spin_lock(&(lock)->raw_lock)
 
在arch/x86/include/asm/spinlock.h中

static __always_inline void __raw_spin_lock(raw_spinlock_t *lock)
{
    __ticket_spin_lock(lock);
}
 
#if (NR_CPUS < 256)
#define TICKET_SHIFT 8

static __always_inline void __ticket_spin_lock(raw_spinlock_t *lock)
{
    short inc = 0x0100;

    asm volatile (
        LOCK_PREFIX "xaddw %w0, %1\n"
        "1:\t"
        "cmpb %h0, %b0\n\t"
        "je 2f\n\t"
        "rep ; nop\n\t"
        "movb %1, %b0\n\t"
        /* don't need lfence here, because loads are in-order */
        "jmp 1b\n"
        "2:"
        : "+Q" (inc), "+m" (lock->slock)
        :
        : "memory", "cc");
}

static __always_inline int __ticket_spin_trylock(raw_spinlock_t *lock)
{
    int tmp, new;

    asm volatile("movzwl %2, %0\n\t"
             "cmpb %h0,%b0\n\t"
             "leal 0x100(%" REG_PTR_MODE "0), %1\n\t"
             "jne 1f\n\t"
             LOCK_PREFIX "cmpxchgw %w1,%2\n\t"
             "1:"
             "sete %b1\n\t"
             "movzbl %b1,%0\n\t"
             : "=&a" (tmp), "=&q" (new), "+m" (lock->slock)
             :
             : "memory", "cc");

    return tmp;
}

static __always_inline void __ticket_spin_unlock(raw_spinlock_t *lock)
{
    asm volatile(UNLOCK_LOCK_PREFIX "incb %0"
             : "+m" (lock->slock)
             :
             : "memory", "cc");
}
#else
#define TICKET_SHIFT 16

static __always_inline void __ticket_spin_lock(raw_spinlock_t *lock)
{
    int inc = 0x00010000;
    int tmp;

    asm volatile(LOCK_PREFIX "xaddl %0, %1\n"
             "movzwl %w0, %2\n\t"
             "shrl $16, %0\n\t"
             "1:\t"
             "cmpl %0, %2\n\t"
             "je 2f\n\t"
             "rep ; nop\n\t"
             "movzwl %1, %2\n\t"
             /* don't need lfence here, because loads are in-order */
             "jmp 1b\n"
             "2:"
             : "+r" (inc), "+m" (lock->slock), "=&r" (tmp)
             :
             : "memory", "cc");
}

static __always_inline int __ticket_spin_trylock(raw_spinlock_t *lock)
{
    int tmp;
    int new;

    asm volatile("movl %2,%0\n\t"
             "movl %0,%1\n\t"
             "roll $16, %0\n\t"
             "cmpl %0,%1\n\t"
             "leal 0x00010000(%" REG_PTR_MODE "0), %1\n\t"
             "jne 1f\n\t"
             LOCK_PREFIX "cmpxchgl %1,%2\n\t"
             "1:"
             "sete %b1\n\t"
             "movzbl %b1,%0\n\t"
             : "=&a" (tmp), "=&q" (new), "+m" (lock->slock)
             :
             : "memory", "cc");

    return tmp;
}

static __always_inline void __ticket_spin_unlock(raw_spinlock_t *lock)
{
    asm volatile(UNLOCK_LOCK_PREFIX "incw %0"
             : "+m" (lock->slock)
             :
             : "memory", "cc");
}
#endif
 </pre>

