# Flat Datacenter Storage

OSDI'2012

## Abstract

Flat Datacenter Storage (FDS) is a high-performance, fault-tolerant, large-scale, locality-oblivious **blob store**.
**Using a novel combination of full bisection bandwidth networks**, data and metadata striping, and ﬂow control,
FDS multiplexes an application’s large-scale I/O across the available throughput and latency budget of every disk
in a cluster. **FDS therefore makes many optimizations around data locality unnecessary.** **Disks also commu-
nicate with each other at their full bandwidth**, making recovery from disk failures extremely fast.

> * disk之间的传输速度是全带宽的，这得益于使用一种新的网络。



## 1 Introduction

**Counterintuitively, locality constraints can sometimes even hinder efﬁcient resource utilization.**

recently developed CLOS networks--large numbers of small commodity switches with
redundant interconnections—have made it economical to build non-oversubscribed full bisection bandwidth networks at the scale of a datacenter for the ﬁrst time. 

Flat Datacenter Storage (FDS) is a datacenter storage system designed from ﬁrst principles under the formerly unrealistic assumption that datacenter bandwidth is abundant.

Unconventionally for a system of this scale, FDS returns to the ﬂat storage model: all compute nodes can access 
all storage with equal throughput. Even when computation is co-located with storage, all storage is treated
as remote; in FDS, there are no “local” disks.

> * 依赖新的网络结构，消除局部性，并提供一个简单和灵活的存储系统。所有存储都是remote的，没有local的disks。

## 2 Design Overview

FDS’ main goal is to expose all of a cluster’s disk bandwidth to applications. Blobs are divided into tracts
(§2.1), parallelizing both storage of data (§2.2) and handling of metadata (§2.3) across all disks.

FDS provides the best of both worlds: a scale-out system with aggregate
I/O throughput equal to systems that exploit local disks combined with the conceptual simplicity and ﬂexibility of a logically centralized storage array.

> * FDS的主要目标是释放整个集群中所有磁盘的带宽。Blobs被分成tracts，data 和 metadata 被在所有disck中paralleizing/handling。

### 2.1 Blobs and Tracts

In FDS, data is logically stored in blobs. A blob is a byte sequence named with a 128-bit GUID. 
Blobs can be any length up to the system’s storage capacity. Reads from and writes to a blob are done in units called tracts.
Tracts are sized such that random and sequential access achieves nearly the same throughput. In our cluster, tracts are 8MB.

> *  每个blob的有一个唯一标识 128-bit GUID. blob被分为定长的tracts.
> *  Blobs可以是任意大小(不超过系统存储的容量)，读写的单位都是tracts。
> *  Tracts的定长的，它所设定的长度保证Blob的随机访问和顺序访问都保持同样的速度。


**Every disk is managed by a process called a tractserver that services read and write requests that arrive
over the network from clients.** Tractservers do not use a ﬁle system. Instead, they lay out tracts directly to disk by
using the raw disk interface. Since there are only about 10^6 tracts per disk (for a 1TB disk), all tracts’ metadata
is cached in memory, eliminating many disk accesses.

> * 上面介绍每个磁盘都有一个叫做tractserver的进程进行管理。tracts保存在磁盘中.

In addition to the listed parameters, each function takes a callback function and an associated context pointer. **All
calls in FDS are non-blocking; the library invokes the application’s callback when the operation completes.**
The application’s callback functions must be reentrant; they are called from the library’s threadpool and
may overlap. Tract reads are not guaranteed to arrive in order of issue. Writes are not guaranteed to be commit-
ted in order of issue. Applications with ordering requirements are responsible for issuing operations after pre-
vious acknowledgments have been received, rather than concurrently. **FDS guarantees atomicity: a write is either
committed or failed completely.**

The non-blocking API helps applications achievegood performance. The FDS API GetSimultaneousLimit() tells the application how many reads and writes to issue concurrently.

> *  每个API都不是阻塞的。
> *  每个API都有callback函数。
> *  每个写操作都是原子性的。
> *  不保证读操作是有序的。
> *  不保证写操作是有序的。

**FDS API**

* Getting access to a blob
    * CreateBlob(UINT128 blobGuid)
    * OpenBlob(UNIT128 blobGuid)
    * CloseBlob(UNIT128 blobGuid)
    * DeleteBlob(UNIT128 blobGuid)

* Interacting with a blob
    * GetBlobSize()
    * ExtendBlobSize(UNIT64 numberOfTracts)
    * WriteTract(UINT64 tractNumber, BYTE \*buf)
    * ReadTract(UINT64 tractNumber, BYTE \*buf)
    * GetSimultaneousLimit()

### 2.2 Deterministic data placement

A key issue in parallel storage systems is data placement and rendezvous, that is: how does a writer know
where to send data? How does a reader ﬁnd data that has been previously written? 

> * 在存储系统中，关键问题是数据是如何放置的.

Many systems solve this problem using a metadata server that stores the location of data blocks.

> * 许多系统使用元数据服务器去存放blocks的locations.

This method has the advantage of allowing maximum ﬂexibility of data placement and visibility into the system’s state.
However, it has drawbacks: the metadata server is a central point of failure, usually implemented as a replicated
state machine, that is on the critical path for all reads and writes.

> *  优点是灵活的data placement和可见的system'state
> *  缺点是元数据服务器是所有读写操作的关键路径，容易成为瓶颈

In FDS, we took a different approach. FDS uses a metadata server, but its role during normal operations
is simple and limited: collect a list of the system’s active tractservers and distribute it to clients. We call this
list the tract locator table, or TLT. In a single-replicated system, each TLT entry contains the address of a sin-
gle tractserver. With k-way replication, each entry has k tractservers; 

> *  FDS也使用元数据服务器，但是它减少了元数据的总量。
> *  client要找到某个block的location，它需要先算一步，再查表一步.

**Tract Locator = (Hash(g)+i) mod TLT Length**

Once clients ﬁnd the proper tractserver address in the TLT, they send read and write requests containing the
blob GUID, tract number, and (in the case of writes) the data payload. Readers and writers rendezvous because
tractserver lookup is deterministic: as long as a reader has the same TLT the writer had when it wrote a tract, a
reader’s TLT lookup will point to the same tractserver.


In a single-replicated system, the TLT is constructed by concatenating m random permutations of the tract server list. 

> * 防止clients读取blob时，它们访问Tract Locator的顺序是一致的.

In the case of non-uniform disk speeds, the TLT is weighted so that different tractservers appear in proportion to the measured speed of the disk

> * TLT还有一个功能是，它可以表示磁盘的权重，比如磁盘的速度等。

To be clear, the TLT does not contain complete information about the location of individual tracts in the sys-
tem. It is not modiﬁed by tract reads and writes. The only way to determine if a tract exists is to contact the tract-
server that would be responsible for the tract if it does exist. Since the TLT changes only in response to clus-
ter reconﬁguration or failures it can be cached by clients for a long time. Its size in a single-replicated system is
proportional to the number of tractservers in the system (hundreds, or thousands), not the number of tracts stored
(millions or billions).

> *  TLT只有当集群重新配置时或者failures时才会发生改变，所以它可以长时间cached在clients上。
> *  TLT的大小和磁盘数量和副本数有关，和tracts的数量无光。

When the system is initialized, tractservers locally store their position in the TLT. This means the metadata
server does not need to store durable state, simplifying its implementation. In case of a metadata server failure, the
TLT is reconstructed by collecting the table assignments from each tractserver.

> *  tractservers在本地也存了和它相关的部分TLT，这意味着元数据服务器不需要保证TLT的持久行，这简化了元数据服务器的实现。
> *  当元数据服务器failure时，TLT可以遍历所有tractserver进行重建.

To summarize, our metadata scheme has a number of nice properties:

* The metadata server is in the critical path only when a client process starts.

* The TLT can be cached long-term since it changes only on cluster conﬁguration, not each read and
write, eliminating all trafﬁc to the metadata server in a running system under normal conditions

* The metadata server stores metadata only about the hardware conﬁguration, not about blobs. Since traf-
ﬁc to it is low, its implementation is simple and lightweight.

* Since the TLT contains random permutations of the list of tractservers, sequential reads and writes by
independent clients are highly likely to utilize all tractservers uniformly and are unlikely to organize
into synchronized convoys.

> *  元数据服务器不会成为读写操作的关键路径
> *  TLT可以长时间缓存在client上
> *  元数据服务器保存的元数据包括TLT和一些硬件配置，不包括blobs，所以它的traffic很低，这使得它的实现非常简单和轻型.
> *  TLT上保存的tractservers列表要足够随机


### 2.3 Per-Blob Metadata

Each blob has metadata such as its length. FDS stores it in each blob’s special metadata tract (“tract −1”).
Clients ﬁnd a blob’s metadata on a tractserver using the same TLT used to ﬁnd regular data. 

When a blob is created, the tractserver responsible for its metadata tract creates that tract on disk and initializes
the blob’s size to 0. When a blob is deleted, that tractserver deletes the metadata. A scrubber application scans
each tractserver for orphaned tracts with no associated metadata, making them eligible for garbage collection.

Newly created blobs have a length of 0 tracts. Applications must extend a blob before writing past the end
of it. The extend operation is atomic, is safe to execute concurrently with other clients, and returns the new size
of the blob as a result of the client’s call. A separate APItells the client the blob’s current size. Extend operations
for a blob are sent to the tractserver that owns that blob’s metadata tract.

> * blob的元数据也被当成tract保存在tactserver上，这使得处理blob的元数据和处理其他tract没啥区别。

### 2.4 Dynamic Work Allocation

## 3 Replication and Failure Recovery

In an n-disk cluster where one disk fails, roughly 1/n th of the replicated data will be found on
all n of the other disks. All remaining disks send the under-replicated data to each other in parallel, restoring
the cluster to full replication very quickly.

> * 优秀的data placement策略保证1/n的副本数据可以在所有n个disk中找到，所有的disk开始并行传输副本，Recovery会非常快.

Such fast failure recovery significantly improves durability because it reduces the win-
dow of vulnerability during which additional failures can cause unrecoverable data loss.

> * 快速的的failure recovery能够显著提供数据的持久性，因为它减少了vulnerability的窗口时间.

### 3.1 Replication

When an application writes a tract, the client library ﬁnds the appropriate row of the TLT and sends the write to every tract-
server it contains. Reads select a single tractserver at random. Applications are notiﬁed that their writes have
completed only after the client library receives write acknowledgments from all replicas.

> * 副本策略。同时向多个副本发起写操作。只想一个副本发起读操作。
> * 可以根据一致性的要求，改变副本策略。

Replication also requires changes to CreateBlob, ExtendBlobSize, and DeleteBlob. Each mutates the
state of the metadata tract and must guarantee that updates are serialized. Clients send these operations only
to the tractserver acting as the primary replica, marked as such in the TLT. When a tractserver receives one of
these operations, it executes a two-phase commit with the other replicas. The primary replica does not commit the
change until all other replicas have completed successfully. Should the prepare fail, the operation is aborted.

> * 在 CreateBlob, ExtendBlobSize，DeleteBlob操作中，primary replica使用two-phase commit保证一致性.

FDS also supports per-blob variable replication, for example, to single-replicate intermediate computations
for write performance, triple-replicate archival data for durability, and over-replicate popular blobs to increase
read bandwidth. The maximum possible replication level is determined when the cluster is created and drives the
number of tractservers listed in each TLT entry. Each blob’s actual replication level is stored in the blob’smeta-
data tract and retrieved when a blob is opened. For an n-way replicated blob, the client uses only the ﬁrst n tract-
servers in each TLT entry

> * FDS支持blob可变的副本数，每个blob的实际副本等级是保存在blob的元数据tract中，对于n副本的系统，client只使用TLT entry中地一个tractserver。

### 3.2 Failure recovery


We begin with the simplest failure recovery case: the failure of a single tractserver.

> * 简单的故障恢复：只有一个tractserver发生故障了。

Tractservers send heartbeat messages to the metadata server. When the metadata server detects a tractserver
timeout, it declares the tractserver dead. Then, it:

* invalidates the current TLT by incrementing the version number of each row in which the failed tractserver appears;

* picks random tractservers to ﬁll in the empty spaces in the TLT where the dead tractserver appeared;

* sends updated TLT assignments to every server affected by the changes; and

* waits for each tractserver to ack the new TLT assignments, and then begins to give out the new TLT to clients when queried for it.

> * 重新生成一个新的TLT

When a tractserver receives an assignment of a new entry in the TLT, it contacts the other replicas and begins
copying previously written tracts. When a failure occurs, clients must wait only for the TLT to be updated; operations can continue while re-replication is still in progress.

> * 开始复制tracts

All operations are tagged with the client’s TLT version. If a client attempts an operation using a stale TLT
entry, the tractserver detects the inconsistency and rejects the operation. This prompts the client to retrieve an updated TLT from the metadata server.

> * 每个TLT entry都有一个version号，每个operation也有一个version号，当tractserver检查到version不一致时，它会拒绝执行operation。

Table versioning prevents a tractserver that failed and then returned, e.g., due to a transient network outage,
from continuing to interact with applications as soon as the client receives a new TLT. Further, any attempt by a
failed tractserver to contact the metadata server will result in the metadata server ordering its termination.

After a tractserver failure, the TLT immediately converges to provide applications the current location to read or write data.

> * 当一个tractserver发生故障时， TLT可以立即收敛。

#### 3.2.1 Additional failure scenarios

We now extend our description to concurrent and cascading tractserver failures as well as metadata server failures.

> * 现在扩大故障范围

When multiple tractservers fail, the metadata server’s only new task is to ﬁll more slots in the TLT. Similarly, if
failure recovery is underway and additional tractservers fail, the metadata server executes another round of the
protocol by ﬁlling in empty table entries and incrementing their version. Data loss occurs when all the tractservers within a row fail within the recovery window.

> * 当多个tarctservers发生故障时。
> * 当failure recovry进行时，又有tractserver发生故障。

A simple TLT might have n rows with each row listing disks i and i+1. While
data will be double-replicated, the cluster will not meet our goal of fast failure recovery: when a disk fails, its
backup data is stored on only two other disks (i+1 and i−1). Recovery time will be limited by the bandwidth of
just two disks. A cluster with 1TB disks with 100MB/s read performance would require 90 minutes for recovery.
A second failure within that time would have roughly a 2/n chance of losing data permanently.



A better TLT has O(n2) entries. Each possible pair of disks (ignoring failure domains; §3.3.1) appears in an
entry of the TLT. Since the generation of tract locators is pseudo-random (§2.2), any data written to a disk will
have high diversity in the location of its replica. **When a disk fails, replicas of 1/nth of its data resides on the
other n disks in the cluster. When a disk fails, all n disks can exchange data in parallel over FDS’ full bisection
bandwidth network. Since all disks recover in parallel, larger clusters recover from disk loss more quickly.**
While such a TLT recovers from single-disk failure quickly, a second failure while recovery is in progress is
guaranteed to lose data. Since all pairs of disks appear as TLT entries, any pair of failures will lose the tracts whose
TLT entry contained the pair of failed disks. Replicated FDS clusters therefore have a minimum replication level
of 3. Perhaps counterintuitively, no level of replication ever needs a TLT larger than O(n2). For any replica-
tion level k > 2, FDS starts with the “all-pairs” TLT, then expands each entry with k−2 additional random disks
(subject to failure domain constraints).

> * 最好的TLT构成方式,当一个磁盘故障时，所有的磁盘并行recover。

Constructing the replicated TLT this way has several important properties. First, performance during recovery
still involves every disk in the cluster since every pair of disks is still represented in the TLT.
Second, a triple disk failure within the recovery window now has only about a 2/n chance1 of causing permanent data loss.

To understand why, imagine two disks fail. Find the entries in the TLT that contain those two
disks. We expect to ﬁnd 2 such entries. There is a 1/n chance that a third disk failure will match the random
third disk in that TLT entry. Finally, adding more replicas decreases the probability of data loss. Consider now a 4-way replicated clus-
ter. Each entry in the O(n2)-length TLT has two random disks added instead of one. 3 or fewer simultaneous fail-
ures are safe; 4 simultaneous failures have a 1/n^2 chance of losing data. Similarly, 5-way replication means that
4 or fewer failures are safe and 5 simultaneous failures havea1/n^3 chance of loss.

> * 好的TLT还能提高数据持久性


One possible disadvantage to a TLT with O(n2) entries is its size. In our 1,000-disk cluster, the in-memory TLT
is about 13MB. However, on larger clusters, quadratic growth is cumbersome: 10,000 disks would require a
600MB TLT. We have two (unimplemented) strategies to mitigate TLT size. First, a tractserver can manage multiple disks;
this reduces n by a factor of 5–10. Second, we can limit the number of disks that participate in failure re-
covery. An O(n2) TLT uses every disk for recovery, but 3,000 disks are expected to recover 1TB in less than
20s (§5.3). The marginal utility of involving more disks may be small. To build an n-disk cluster where m disks
are involved in recovery, the TLT only needs O(XXX) entries. For 10,000 to 100,000 disks, this also reduces
table size by a factor of 5–10. Using both optimizations,a 100,000 disk cluster’s TLT would be a few dozen MB.

> * TLT的大小，TLT可以缓存在内存中，TLT的大小和磁盘的数量有关，但是可以进行优化.

#### 3.3.1 Failure domains

A failure domain is a set of machines that have a high probability of experiencing a correlated failure. Com-
mon failure domains include machines within a rack,since they will often share a single power source, or ma-
chines within a container, as they may share common cooling or power infrastructure.
FDS leaves it up to the administrator to deﬁne a failure domain policy for a cluster. Once that policy is de-
ﬁned, FDS adheres to that policy when constructing the tract locator table. FDS guarantees that none of the disks
in a single row of the TLT share the same failure domain. This policy is also followed during failure recov-
ery: when a disk is replaced, the new disk must be in a different failure domain than the other tractservers in that
particular row.

> * 故障域，同一个机箱，同一个机架上的磁盘都是在同一个故障域中，每个TLT的entry中不能包含在同一个故障域的磁盘.

### 3.4 Gluster growth

FDS supports the dynamic growth of a cluster through the addition of new disks and machines. For simplicity,
we ﬁrst consider cluster growth in the absence of failures. Cluster growth adds both storage capacity and
throughput to the system. The FDS metadata server re-balances the assignment of table entries so that both ex-
isting data and new workloads are uniformly distributed. When a tractserver is added to the cluster, TLT entries are
taken away from existing tractservers and given to the new server. These assignments happen in two phases.
First, the new tractserver is given the assignments but they are marked as “pending” and the TLT version for
each entry is incremented. The new tractserver then be-gins copying data from other replicas. During this phase,
clients write data to both the existing servers and the new server so that the new tractserver is kept up-to-date. Once
the tractserver has completed copying old data, the meta-data server ‘commits’ the TLT entry by incrementing its
version and changing its assignment to the new tract-server. 

It also notiﬁes the now replaced tractserver that it can safely garbage collect all tracts associated with that TLT entry. If a new tractserver fails while its TLT entries are
pending, the metadata server increments the TLT entry version and expunges it from the list of new tractservers.

If an existing server fails, the failure recovery protocol executes. However, tractservers with pending TLT en-
tries are not eligible to take over for failed servers as they are already busy copying data.

> * 如何处理集群扩展

### 3.5 Consistency guarantees

The current protocol for replication depends upon the client to issue all writes to all replicas. This decision
means that FDS provides weak consistency guarantees to clients. For example, if a client writes a tract to 1
of 3 replicas and then crashes, other clients reading dif- The current protocol for replication depends upon the
client to issue all writes to all replicas. This decision means that FDS provides weak consistency guarantees
to clients. For example, if a client writes a tract to 1 of 3 replicas and then crashes, other clients reading dif-
cas of that tract will observe differing state. Weak consistency guarantees are not uncommon; for ex-
ample, clients of the Google File System must handle garbage entries in ﬁles. However, if strong consis-
tency guarantees are desired, FDS could be modiﬁed to use chain replication to provide strong consistency
guarantees for all updates to individual tracts. Tractservers may also be inconsistent during failure
recovery. A tractserver recently assigned to a TLT entry will not have the same state as the entry’s other repli-
cas until data copying is complete. While in this state, tractservers reject read requests; clients use other repli-
cas instead.

> * 目前client发起的写操作会向所有的副本写数据，这说明FDS目前提供的是弱一致性。
> * 可以使用chain replication提供强一致性.

## 4 Networking

## 5 Microbenchmarks

<table border="1">
<tr>
    <th>Disk count</th>
    <th colspan="2">100</th>
    <th colspan="3">1,000</th>
</tr>
<tr>
    <th>Disks failed</th>
    <td>1</td>
    <td>1</td>
    <td>1</td>
    <td>1</td>
    <td>7</td>
</tr>
<tr>
    <th>Total(TB)</th>
    <td>4.7</td>
    <td>9.2</td>
    <td>47</td>
    <td>92</td>
    <td>92</td>
</tr>
<tr>
    <th>GB/disk</th>
    <td>47</td>
    <td>92</td>
    <td>47</td>
    <td>92</td>
    <td>92</td>
</tr>
<tr>
    <th>GB recov.</th>
    <td>47</td>
    <td>92</td>
    <td>47</td>
    <td>92</td>
    <td>655</td>
</tr>
<tr>
    <th>Recovery time(s)</th>
    <td>19+-0.7</td>
    <td>50.5+-16.0</td>
    <td>3.3+-0.6</td>
    <td>6.2+-0.4</td>
    <td>33.7+-1.5</td>
</tr>
</table>

## 个人总结

### 解决问题的步骤

1. 分析目前数据中心网络的问题，没有一个Flat的网络，各个节点间的带宽/速度不一致。
2. 从利用局部性到抛弃局部性，没有local，只有remote。
3. 根据这种新的模型，设计新的存储系统

### 研究的方法

自己对存储系统的抽象


<table border="1">
    <tr>
        <td>data view</td>
    </tr>
    <tr>
        <td>data operator</td>
    </tr>
    <tr>
        <td>data flow</td>
    </tr>
    <tr>
        <td>data store</td>
    </tr>
    <tr>
        <td>data unit</td>
    </tr>
</table>


1. 根据网络模型设计存储系统
2. 设计概述
    1. Blobs and tracts: data view, data operator
    2. Deterministic data placement: data flow
        * 从data view到data store需要映射
        * 有两种方法得到这个映射：算法，查表
        * 算法有一致性哈系算法等，操作路径不会经过元数据服务器，纯使用算法不够灵活
        * 查表更灵活，但是元数据服务器会成为瓶颈，破解办法在于减少元数据的数量,而且元数据不经常变更，这使得元数据可以缓存在client上
        * 最好的映射办法是：算法+查表
        * FDS使用TLT的办法，TLT的构造很关键，TLT会影响data flow，强大的data flow可以让所有disk并行工作(提高性能)，让数据分布在所有disk上(快速恢复)
    3. Per-blob metadata: 如何处理data view的元数据
3. 副本与故障恢复
    1. Replication
        * 基本的副本策略
        * 副本操作
        * 针对每个Blog可变的副本级别(存在blob的metadata上)
    2. Failure Recovery
        * 先定义简单的Failure 场景和Recovery策略
        * 再定义复杂的Failure 场景
        * TLT的构造对Recovery的速度有影响，Recovery的速度对数据持久行有影响
    3. Failure domains
        * 防止同一个机箱、机架上的磁盘在同一个TLT entry上。
    4. Gluster growth
        * 扩容的方法
    5. Consistency guarentees
        * 一致性的策略应该是可以调整的，以便满足用户的需求


### 技术应用

1. TLT的设计与构造值得借鉴
2. 任何功能的策略应该是可以替换的
