# Hypervisor-based Fault-tolerance

1995

## Abstrace

Protocols to implement a fault-tolerant computing system are described.
These protocols augment the hypervisor of a virtual machine manager and coordinate a primary virtual machine with its backup.
No modification to hardware, operating system, or application programs is required.

## 1. Introduction

One popular scheme for implementing fault tolerance involves replicating a computation on processors that fail independently.
Replicas are coordinated so that they execute the same sequence of instructions and produce the same results.
Use of a hypervisor to implement replica coordination is attractive.

This paper address these issuse by describing the protocols and performance of a prototype implementation of hypervisor-base fault-tolerance.

In $2, we describe the protocols. These protocols ensure that the sequence of instructions executed by two virtual machines running on diffenrent physical processors are identical.
The protocols also coordinate I/O issued by these virtual machines.

## 2. Replica-Coorination Protocols

In the primary/backup approach to fault-tolerance, n processors implement a system that can tolerate n-1 faults. 
One processor is designated the primary and the others are designated backups. To obtain service, clients make requests of the primary. 
The primary responds to each request and informs the backups of its actions so that a bakcup can take over if the primary fails.

Our implementation of fault-tolerant virtual machines uses the primary/backup approach in the hypervisor.

Our protocols use a single backup and implement a 1-fault-tolerant virtual machine. The protocols cause the backup virtual machine to execute exactly the same 
sequence of instructions as the primary virtual machine, where each instruction executed by the backup has the same effect as when it executed by the primary. 
The protocols also ensure that the environment does not see an anomalous sequence of I/O requests if the primary fails and the backup takes over while an I/O 
operation is in progress.

One obvisou assumption, required so that the backup virtual machine can **take over** for the primary:

*  **I/O Device Accessibility Assumption**:I/O device accessible to the processor executing the primary virtual machine are also accessible to the processor executing the backup virtual machine.
*  **Vitual Machine Assumption**: System and user software executes correctly under the hypervisor.

### 2.1 Indentical Instruction Streams

In our scheme, a given instruction must have the same effect whether it is executed by the primary virtual machine or the backup.

Define the virtual machine state to include the memory and registers that change only with execution of instructions by that virtual machine.

We partition the instruction set into **ordinary instructions**, whose behavior is completely determined by virtual-machine state, and **enviroment instructions**, whose
behavior is not.Examples of ordinary instructions include those for arithmetic and data movement; examples of enviroment instructions include those for reading the
time-of-day clock, loding the interval timer, and for performing I/O.

* ordinary instructions:只在系统内，只影响寄存器和内存.
* enviroment instructions: 和外部关联，从外部读取数据，或者向外写出数据，不可控.

In order that the primary and backup virtual machines execute exactly the same sequence of instructions, both virtual machines are started in the same state. 
By definition, the effects of executing ordinary instructions depend only on the virtual-machine state.

* **Ordinary Instruction Assumption**: Executing the same ordinary instruction on two processors in the same virtual-machine state has exaclty the same effect.

Another assumption ensures that when executing an environment instruction. instruction, the hypervisor at the primary and backup virtual machines have an opportunity to
communicate. This allows both hypervisors to change the virtual-machine state in the same way.

* **Enviroment Instruction Assumption**: Environment instructions are simulated by the hypervisor (and not executed directly by the hardware). The simulation
ensures that a given environment instruction executed on two processors in the same virtual-machine state has exactly the same effect on the virtual-machine state.

To guarantee that the primary and backup virtual machines execute the same sequence of instructions, we must
ensure that **identical interrupts are delivered to each and at the same points in their instruction streams**. The presence of a 
hypervisor helps here. **The primary’s hypervisor can buffer and forward I/O interrupts it receives to the backup’s hypervisor.
And, the primary’s hypervisor can send to the backup’s hypervisor information about the value of the interval timer at the pro-
cessor executing the primary virtual machine**. Thus, by communicating with the primary’s hypervisor, the backup’s hypervisor
learns what interrupts it must deliver to the backup virtual machine,

This is because instruction-execution timing on most modem processors is unpredictable, **Yet, interrupts must be
delivered at the same points in the primary and backup virtual machine instruction streams.** We must employ some other mech-
anism for transferring control to the hypervisor when a virtual machine reaches a specified point in its instruction stream.

The recovery register on HP’s PA-RISC processors is a register that is decremented each time an instruction completes;
an interrupt is caused when the recovery register becomes negative.

**hypervisor that uses the recovery register can thus ensure that epochs at the primary and backup virtual machines
each begin and end at exactly the same point in the instruction stream.** Interrupts are delivered only on epoch boundaries.
A recovery register or some similar mechanism is, therefore, assumed.

* **Instruction-Stream Interrupt Assumption**: A mechanism is available to invoke the hypervisor when a specified point in the instruction stream is reached.

By virtue of the Instruction-Stream Interrupt Assumption, execution of a virtual machine is partitioned into epochs, and
**corresponding epochs at the primary and the backup virtual machines comprise the same sequences of instructions.**

**We have only to ensure that the same interrupts are delivered at the backup as at the primary
when each epoch ends.** The solution to this is for the primary and backup hypervisor to communicate, and at the end of an
epoch i to have the backup’s hypervisor deliver copies of the interrupts that primary’s hypervisor delivered at the end of its
epoch i.

We now summarize the protocol that ensures the primary and backup virtual machines each performs the same sequence
of instructions and receives the same interrupts.

The protocol is presented as a set of routines that are implemented in the hypervisor. These routines may be activated concurrently.

* P0
* P1
* P2
* P3
* P4
* P5
* P6
* P7

through P7 ensure that the backup virtual machine executes the same sequence of instructions (each having
the same effect) as the primary virtual machine. PO through P7also ensure that if the primary virtual machine fails, then instruc-
tions executed by the backup extend the sequence of instructions executed by the primary.

PO through P7 do not guarantee that interrupts from I/O devices are not lost.

### 2.1 Interaction with an Environment

The state of the environment is affected by executing I/O instructions. We must ensure that the sequence of I/O instruc-
tions seen by the environment is consistent with what could be observed were a single processor in use, even though our l-fault-
tolerant virtual machine is built using two processors. The problem is best divided into two cases.

> 这里遇到了一个问题，因为指令是原子性的，不可能同时发起两条指令，因此执行I/O指令和通知Backup指令不能同时进行
> Our solution is to exploit the reality that I/O devices are themselves subject to transient failures, and device drivers already cope with these faihrres.
> 这里需要硬件设备的支持

* P8

**The effect of P8 is to cause certain I/O instructions to be repeated**.

## A Prototype System

设计与开发原型系统.

## Performance of the Prototype

对原型系统进行测试。对CPU-Intensive Workload和I/O Workloads进行了测试，以Epoch Length为维度。

## Summary


## 个人总结

### 解决问题的步骤

1. 用某种新方法解决某种问题
2. 并设计模型
3. 开发出原型系统
4. 对原型系统进行测试，得到测试数据
5. 根据测试数据进行总结

### 研究的方法

1. 定义问题，因为真实的世界很复杂，你需要先对问题有清晰的定义。
2. 定义你的模型，你的模型只能不断逼近真实世界，只能是真实世界的子集。
3. 定义你模型的假设条件(公理系统？？？)
4. 定义你的模型的状态集(state set)。
5. 定义能够改变状态的操作和指令集。
6. 定义你要的容错系统(Fault-tolerance)。
7. 定义你要的一致性。(操作顺序，不同client发起不同的操作看到的视图)
8. 实现你模型的假设条件，实现你的模型。
9. 优化

### 看论文的方法

1. 找出问题(Fault-tolerant)
2. 找出方法(Hypervisor replicas)
3. 找出关键点(5个假设)
4. 找出测试数据,找出影响测试的因素(Epoch Length)
5. 找出方法中的问题
6. **思考这种方法还可以用在哪些地方**

### 技术应用

1. 研究的方法能用在分布式存储系统上。



