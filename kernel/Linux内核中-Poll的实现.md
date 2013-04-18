 
# 先描述Poll函数的功能和定义
 
在APUE第384页中，有Poll函数的定义

    #include <poll.h>
    int poll(struct pollfd fdarray[], nfds_t nfds, int timeout);

返回值：准备就绪的描述符数，若超时则返回0，若出错则返回-1；
 
    struct pollfd{
        int  fd;                //file descriptor to check, or < 0 to ignore
        short  events;    // events of interest on fd
        short revents;    // events that occurred on fd
    };
 
> nfds说明了fdarray数组中的元素数。

> events 的标志有 POLLIN   POLLOUT 等等

> revents 的标志有 POLLIN POLLOUT POLLERR POLLHUP POLLNVAL等等

> poll的最后一个参数说明我们愿意等待多少时间。

>   * timeout == -1  永远等待。
>   * timeout == 0   不等待，测试所有描述符并立即返回。
>   * timeout > 0    等待timeout毫秒。
 
 
## 在linux/poll.h中定义了

<pre name="code" class="c"> 
typedef void (*poll_queue_proc)(struct file *, wait_queue_head_t *, struct poll_table_struct *);

typedef struct poll_table_struct {
    poll_queue_proc qproc;
} poll_table;

static inline void poll_wait(struct file * filp, wait_queue_head_t * wait_address, poll_table *p)
{
    if (p && wait_address)
        p->qproc(filp, wait_address, p);
}

static inline void init_poll_funcptr(poll_table *pt, poll_queue_proc qproc)
{
    pt->qproc = qproc;
}

struct poll_table_entry {
    struct file *filp;                //文件描述符
    wait_queue_t wait;        //__pollwait对这个wait进行初始化，把它添加到等待队列中。
    wait_queue_head_t *wait_address;        //等待队列，一般是文件上的等待队列
};

/*
 * Structures and helpers for sys_poll/sys_poll
 */
struct poll_wqueues {
    poll_table pt;
    struct poll_table_page *table;     //poll_table_page的定义在select.c中，假如内部空间不够，可以用于分配空间
    struct task_struct *polling_task;
    int triggered;
    int error;
    int inline_index;
    struct poll_table_entry inline_entries[N_INLINE_POLL_ENTRIES];        //内部空间
};

extern void poll_initwait(struct poll_wqueues *pwq);
extern void poll_freewait(struct poll_wqueues *pwq);
extern int poll_schedule_timeout(struct poll_wqueues *pwq, int state,
                 ktime_t *expires, unsigned long slack);

static inline int poll_schedule(struct poll_wqueues *pwq, int state)
{
    return poll_schedule_timeout(pwq, state, NULL, 0);
}
 
### poll_table_page的定义如下所示。
struct poll_table_page {
    struct poll_table_page * next;
    struct poll_table_entry * entry;
    struct poll_table_entry entries[0];
};
</pre>
 
## 在fs/select.c中

### do_sys_poll的定义如下所示
 
<pre name="code" class="c"> 
#define N_STACK_PPS ((sizeof(stack_pps) - sizeof(struct poll_list)) / \
            sizeof(struct pollfd))
#define POLLFD_PER_PAGE ((PAGE_SIZE-sizeof(struct poll_list)) / sizeof(struct pollfd))
 
int do_sys_poll(struct pollfd __user *ufds, unsigned int nfds,
        struct timespec *end_time)
{
    struct poll_wqueues table;            //重要的数据结构
     int err = -EFAULT, fdcount, len, size;
    /* Allocate small arguments on the stack to save memory and be
       faster - use long to make sure the buffer is aligned properly
       on 64 bit archs to avoid unaligned access */
    long stack_pps[POLL_STACK_ALLOC/sizeof(long)];            //POLL_STACK_ALLOC = 256
    struct poll_list *const head = (struct poll_list *)stack_pps;        //poll_list 的定义在下面
     struct poll_list *walk = head;
     unsigned long todo = nfds;

    if (nfds > current->signal->rlim[RLIMIT_NOFILE].rlim_cur)    //进程打开文件数目的限制
        return -EINVAL;
    //先用栈中的空间，假如栈中的空间不够用，则使用堆中的空间。这样既能节省空间也能提高效率
   //把struct pollfd __user *ufds中的元素都复制到poll_list *head的链表中
    len = min_t(unsigned int, nfds, N_STACK_PPS);
    for (;;) {
        walk->next = NULL;
        walk->len = len;
        if (!len)
            break;

        if (copy_from_user(walk->entries, ufds + nfds-todo,
                    sizeof(struct pollfd) * walk->len))
            goto out_fds;

        todo -= walk->len;
        if (!todo)
            break;

        len = min(todo, POLLFD_PER_PAGE);
        size = sizeof(struct poll_list) + sizeof(struct pollfd) * len;
        walk = walk->next = kmalloc(size, GFP_KERNEL);
        if (!walk) {
            err = -ENOMEM;
            goto out_fds;
        }
    }
    poll_initwait(&table);         //初始化table , poll_iniwait定义在下面
    fdcount = do_poll(nfds, head, &table, end_time);    //do_poll的定义在下面
    poll_freewait(&table);        //处理table上的信息，得到事件信息，poll_freewait函数的定义在下面
    //把事件信息复制到用户空间上。
    for (walk = head; walk; walk = walk->next) {
        struct pollfd *fds = walk->entries;
        int j;

        for (j = 0; j < walk->len; j++, ufds++)
            if (__put_user(fds[j].revents, &ufds->revents))
                goto out_fds;
      }

    err = fdcount;
out_fds:
    walk = head->next;
    while (walk) {
        struct poll_list *pos = walk;
        walk = walk->next;
        kfree(pos);
    }

    return err;
}
</pre>

### poll_list 的定义如下所示

<pre name="code" class="c"> 
struct poll_list {
    struct poll_list *next;
    int len;                            //数组entries的长度
    struct pollfd entries[0];    //数组指针
};
</pre>

### poll_initwait的定义如下所示

<pre name="code" class="c"> 
void poll_initwait(struct poll_wqueues *pwq)
{
    init_poll_funcptr(&pwq->pt, __pollwait);
    pwq->polling_task = current;        //important
    pwq->error = 0;
    pwq->table = NULL;
    pwq->inline_index = 0;
}
</pre>

### __pollwait的定义如下所示

<pre name="code" class="c"> 
static void __pollwait(struct file *filp, wait_queue_head_t *wait_address,
                poll_table *p)
{
    struct poll_wqueues *pwq = container_of(p, struct poll_wqueues, pt);
    struct poll_table_entry *entry = poll_get_entry(pwq);
    if (!entry)
        return;
    get_file(filp);
    entry->filp = filp;
    entry->wait_address = wait_address;
    init_waitqueue_func_entry(&entry->wait, pollwake);    //init_waitqueue_func_entry函数的定义如下所示，pollwake函数的定义如下所示
    entry->wait.private = pwq;
    add_wait_queue(wait_address, &entry->wait);
}
</pre>

### init_waitqueue_func_entry的定义如下所示

<pre name="code" class="c"> 
static inline void init_waitqueue_func_entry(wait_queue_t *q,
                    wait_queue_func_t func)
{
    q->flags = 0;
    q->private = NULL;
    q->func = func;
}
</pre>

### poll_wake的定义如下所示

<pre name="code" class="c"> 
static int pollwake(wait_queue_t *wait, unsigned mode, int sync, void *key)
{
    struct poll_wqueues *pwq = wait->private;
    DECLARE_WAITQUEUE(dummy_wait, pwq->polling_task);

    /*
     * Although this function is called under waitqueue lock, LOCK
     * doesn't imply write barrier and the users expect write
     * barrier semantics on wakeup functions. The following
     * smp_wmb() is equivalent to smp_wmb() in try_to_wake_up()
     * and is paired with set_mb() in poll_schedule_timeout.
     */
    smp_wmb();
    pwq->triggered = 1;

    /*
     * Perform the default wake up operation using a dummy
     * waitqueue.
     *
     * TODO: This is hacky but there currently is no interface to
     * pass in @sync. @sync is scheduled to be removed and once
     * that happens, wake_up_process() can be used directly.
     */
    return default_wake_function(&dummy_wait, mode, sync, key);
}
</pre>

### default_wake_function的定义如下所示

<pre name="code" class="c"> 
int default_wake_function(wait_queue_t *curr, unsigned mode, int sync,
              void *key)
{
    return try_to_wake_up(curr->private, mode, sync);
}
</pre>

### poll_get_entry的定义如下所示

<pre name="code" class="c"> 
#define POLL_TABLE_FULL(table) \
    ((unsigned long)((table)->entry+1) > PAGE_SIZE + (unsigned long)(table))
 
static struct poll_table_entry *poll_get_entry(struct poll_wqueues *p)
{
    struct poll_table_page *table = p->table;

    if (p->inline_index < N_INLINE_POLL_ENTRIES)    //先用内部的空间
        return p->inline_entries + p->inline_index++;

    if (!table || POLL_TABLE_FULL(table)) {            //空间不足，分配空间
        struct poll_table_page *new_table;

        new_table = (struct poll_table_page *) __get_free_page(GFP_KERNEL);
        if (!new_table) {
            p->error = -ENOMEM;
            return NULL;
        }
        new_table->entry = new_table->entries;
        new_table->next = table;
        p->table = new_table;
        table = new_table;
    }

    return table->entry++;        //使用分配的空间
}
</pre> 
 
 
### poll_freewait的定义如下所示

<pre name="code" class="c"> 
static void free_poll_entry(struct poll_table_entry *entry)
{
    remove_wait_queue(entry->wait_address, &entry->wait);
    fput(entry->filp);
}
 
void poll_freewait(struct poll_wqueues *pwq)
{
    struct poll_table_page * p = pwq->table;
    int i;
    for (i = 0; i < pwq->inline_index; i++)
        free_poll_entry(pwq->inline_entries + i);
    while (p) {
        struct poll_table_entry * entry;
        struct poll_table_page *old;

        entry = p->entry;
        do {
            entry--;
            free_poll_entry(entry);
        } while (entry > p->entries);
        old = p;
        p = p->next;
        free_page((unsigned long) old);
    }
}
</pre>

### do_poll的定义如下所示

<pre name="code" class="c"> 
static int do_poll(unsigned int nfds, struct poll_list *list,
           struct poll_wqueues *wait, struct timespec *end_time)
{
    poll_table* pt = &wait->pt;
    ktime_t expire, *to = NULL;
    int timed_out = 0, count = 0;
    unsigned long slack = 0;

    /* Optimise the no-wait case */
    if (end_time && !end_time->tv_sec && !end_time->tv_nsec) {//假如超时时间设置为0，则pt = NULL，poll调用就不用阻塞休眠
        pt = NULL;
        timed_out = 1;
    }

    if (end_time && !timed_out)
        slack = estimate_accuracy(end_time);

    for (;;) {
        struct poll_list *walk;

        for (walk = list; walk != NULL; walk = walk->next) {
            struct pollfd * pfd, * pfd_end;

            pfd = walk->entries;
            pfd_end = pfd + walk->len;
            for (; pfd != pfd_end; pfd++) {
                /*
                 * Fish for events. If we found one, record it
                 * and kill the poll_table, so we don't
                 * needlessly register any other waiters after
                 * this. They'll get immediately deregistered
                 * when we break out and return.
                 */
                if (do_pollfd(pfd, pt)) {
                    count++;                //假如找到一个事件，poll就不用阻塞了。
                    pt = NULL;            
                }
            }
        }
        /*
         * All waiters have already been registered, so don't provide
         * a poll_table to them on the next loop iteration.
         */
        pt = NULL;
        if (!count) {
            count = wait->error;
            if (signal_pending(current))
                count = -EINTR;
        }
        if (count || timed_out)    //假如检查到event发生或者是时间超时，则跳出循环
            break;

        /*
         * If this is the first loop and we have a timeout
         * given, then we convert to ktime_t and set the to
         * pointer to the expiry value.
         */
        if (end_time && !to) {
            expire = timespec_to_ktime(*end_time);
            to = &expire;
        }

        if (!poll_schedule_timeout(wait, TASK_INTERRUPTIBLE, to, slack))    //进入休眠，等待被唤醒
            timed_out = 1;
    }
    return count;
}
</pre>

### do_pollfd的定义如下所示

<pre name="code" class="c"> 
/*
 * Fish for pollable events on the pollfd->fd file descriptor. We're only
 * interested in events matching the pollfd->events mask, and the result
 * matching that mask is both recorded in pollfd->revents and returned. The
 * pwait poll_table will be used by the fd-provided poll handler for waiting,
 * if non-NULL.
 */
static inline unsigned int do_pollfd(struct pollfd *pollfd, poll_table *pwait)
{
    unsigned int mask;
    int fd;

    mask = 0;
    fd = pollfd->fd;
    if (fd >= 0) {
        int fput_needed;
        struct file * file;

        file = fget_light(fd, &fput_needed);    //获取文件描述符
        mask = POLLNVAL;
        if (file != NULL) {
            mask = DEFAULT_POLLMASK;
            if (file->f_op && file->f_op->poll)
                mask = file->f_op->poll(file, pwait);    //执行设备驱动的poll函数，在poll函数中，可能调用poll_wait函数(在poll.h中定义)。
            /* Mask out unneeded events. */
            mask &= pollfd->events | POLLERR | POLLHUP;   //设置状态
            fput_light(file, fput_needed);
        }
    }
    pollfd->revents = mask;        //设置已发生事件的状态

    return mask;
}
</pre> 

