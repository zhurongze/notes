1.  编写scull模块, scull01.tar.gz,包括的函数有

*   scull_llseek,
*   scull_read,
*   scull_write,
*   scull_open,
*   scull_release.

1.  添加proc接口，用于debug，查看驱动程序(设备)的状态，源代码是 scull02.tar.gz,新加了

*   scull\_read\_procmem，
*   scull\_create\_proc，
*   scull\_remove\_proc函数。

1.  添加 seq_file(proc)的接口，解决proc的不足，源代码是scull03.tar.gz

2.  添加ioctl调用，源代码是scull04.tar.gz

3.  编写新的模块(LDD3中的第151页)，实现了简单的休眠。源代码是sleepy.tar.gz

4.  编写新的模块(LDD3中的第153页)，实现了阻塞IO。源代码是piepe.tar.gz

5.  在piepe.tar.gz的基础上添加了poll函数(LDD3中的第165页。源代码是piepe.02.tar.gz

6.  编写时间、延迟及延缓操作模块。源码是myjit.tar.gz。

<pre name="code" class="bash">#head -8  /proc/currentime             #获取当前时间
    #dd   bs=20  count=5  &lt;  /proc/jitbusy    #忙等待
    #dd bs=20 count=5 &lt; /proc/jitsched        #使用schedule()函数，让出CPU
    #dd bs=20 count=5 &lt; /proc/jitschedto     #使用schedule_timeout()函数，等待超时
    #dd bs=20 count=5 &lt; /proc/jitqueue         #使用wait_event_interruptible_timeout()函数，等待超时
    #cat  /proc/jitimer            #使用内核定时器
    #cat  /proc/jitasklet           #使用tasklet
    #cat /proc/jitasklethi         #使用tasklethi
</pre>

1.  编写内存分配模块，使用slab，源码是scullc.tar.gz。
2.  学习中断处理，源码是shortp.tar.gz。
3.  在shortp的基础上添加tasklet，源码是shortp02.tar.gz。
4.  编写PCI驱动，源码是pci.tar.gz。
5.  编写了一个总线类型、一个总线设备、一个总线驱动。源码是lddbus.tar.gz。
6.  编写一个总线下的设备。源码是ldddevice.tar.gz。
7.  编写scullv设备，该设备使用vmalloc获取内存空间。并完成mmap内存映射功能。主要是完成了fault函数。源码是scullv.tar.gz。
8.  编写块设备驱动，就像ramdisk一样。源码是sbull.tar.gz。
