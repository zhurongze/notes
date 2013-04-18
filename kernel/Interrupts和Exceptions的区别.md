# 1 Interrupts事件与Exceptions事件的区别

Interrupts事件和Exceptions事件的共同点是改变当前CPU的控制流。

**Interrupts：异步事件(跟当前指令无关)。事件是由CPU外部产生的。**

Interrupts事件产生源有：

* I/O中断
* 时钟中断
* 定时器

 
Interrupts可以分成可屏蔽Interrupts、不可屏蔽Interrupts。分类依据是否可屏蔽。

* 可屏蔽Interrupts：有两种状态，已屏蔽、未屏蔽。CPU会忽略已屏蔽的Interrupt。
* 不可屏蔽Interrupts ：硬件失效时不可屏蔽的Interrupts  。CPU不能忽略。

当Interrupts事件发生时，在内核态(一般都是kernel mode)的堆栈中保存的EIP的值是下一条指令的地址。

 
**Exceptions：同步事件(跟当前指令有关，被CPU侦测到的不正常情况)。事件是由CPU内部产生的。可以重现的事件。**

Exceptions事件产生源有：

* 内存访问错误
* DEBUG指令（INT3）
* 除0操作
* INTO（检查溢出）
* BOUND（检查地址边界）
* 软中断(INT )：一般用于系统服务请求

Exceptions的分类依据是EIP的值（这个EIP是保存在kernelmode下的堆栈中）。分类如下所示：

* Faults：可以被修正(由事件处理程序修正)。一旦修正后，原来的程序会重新执行引起Fault的指令（因为EIP的值是引起Fault指令的地址）。例如内存缺页错误。
* Traps：当执行引起Trap的指令后，事件处理程序会被立即执行。当事件处理程序执行完后，原来的程序执行下一条指令（引起Trap指令的下一条指令）。这是因为EIP的值下一条指令的地址。Trap的主要用途是用于DEBUG（设置断点）。
* Aborts：严重的错误发生。CPU不能保存引起Abort指令的精确地址到EIP中。Abort用于报告严重的错误，比如硬件故障、无效不一致的系统表。事件处理程序必须停止引起Abort的程序。

可编程的Exceptions（也就是软中断）。程序可以请求Exceptions。它们由int、int 3、into、bound引起。CPU把软中断都当成Trap事件。软中断的两个主要用途是：实现系统调用和DEBUG。

 

 

 
# 2 Intel处理Interrupts事件和Exceptions事件的过程

![过程](http://hi.csdn.net/attachment/201109/4/0_1315136813ZSR3.gif)

注意：

* 发生事件的分类与中断向量表中门的分类无关，无对应关系（比如NMI是不可屏蔽的硬件中断，但是它指向的门可以是Trap Gate – “set_trap_gate(2,&nmi);”）。
* 发生事件的类型影响“CPU捕获信号”和“保存EIP”。可屏蔽中断事件被CPU忽略。在Fault事件中，EIP保存的是引起Fault事件的指令的地址。在Trap事件中，EIP保存的是引起Trap事件的指令的下一条指令的地址。
* 中断向量表中门的不同影响事件处理程序中TF和IF的值。当通过Interrupt gate或Trap gate访问事件处理程序时,在将EFLAGS寄存器的内容保存进栈后，处理器会清除EFLAGS寄存器的TF位（还会清VM、RF、NT位）。清TF位则可以禁止指令跟踪，以使中断响应不受影响，后继的IRET指令则使用保存在栈中的EFLAGS寄存器中的值，恢复TF（和VM、RF、NT）位。Interrupt gate和Trap gate的唯一区别在于处理器处理EFLAGS寄存器的IF位的方式。当通过Interrupt gate访问事件处理程序时，处理器清楚IF位，以阻止另外的中断干扰当前的中断事件处理程序。后继的IRET指令用存储在栈中的EFLAGS的内容恢复IF的值。而通过Trap gate访问事件处理程序时，IF位不受影响。

 
# 3 Linux如何使用Interrupts和Exceptions

INT指令允许用户模式下的程序发送Interrupt signal（中断向量从0到255）。因此初始化IDT必须非常小心。要防止用户模式下的程序通过INT指令访问关键的事件处理程序。设置Interrupt gate Descriptor和Trap gate Descriptor中DPL的值为0后，当程序尝试发送某个Interrupt signal时，CPU会检查CPL和DPL的值，假如CPL的值比DPL的值高（也就是CPL的优先级比DPL的低），这会触发“General protection” exception。

当然在某些情况，用户模式下的程序必须能触发一个可编程的exception。因此，必须设置对应的Interrupt gate Descriptor和Trap gate Descriptor中DPL的值为3。

Intel提供三种interrupt descriptors：Task、Interrupt、Trap Gate Descriptors。因为Linux没有使用到Task GateDescriptors。所以IDT中断向量表只包含Interrupt Gate Descriptors和Trap Gate Descriptors。Linux定义了如下概念，术语跟Intel稍微有些不同。


<table>
    <tr>
        <td>Interrupt gate</td>
        <td>一个Intel Interrupt gate（这个gate的DPL的值是0）不能被用户模式下的程序访问。Linux Interrupt gate指向的事件处理程序(Linux interrupt handlers)只能在内核模式下运行。</td>
    </tr>
    <tr>
        <td>System gate</td>
        <td>一个Inter trap gate（这个gate的DPL的值是3）可以被用户模式下的程序访问。四个Linux exception handlers对应着中断向量3、4、5、128(system gate)。所以4个汇编指令int 3、 into、bound、 int $0x80可以在用户模式下执行。</td>
    </tr>
    <tr>
        <td>Trap gate</td>
        <td>一个Intel trap gate(这个gate的DPL的值是0)不能被用户模式下的程序访问。大部分的Linux exceptionhandlers对应着Trap gate。</td>
    </tr>
</table>



