# Lecture 4: FDS Case Study

    Flat Datacenter Storage
    Nightingale, Elson, Fan, Hofmann, Howell, Suzue
    OSDI 2012

## what is FSD?

    a cluster storage system
    stores giant blobs -- 128-bit ID, multi-megabyte content
    for big-data processing (like MapReduce)
    cluster of 1000s of computers processing data in parallel

## Hign-level design -- a common pattern

    lots of clients
	lots of storage servers ("tractservers")
	partition the data
	master ("metadata server") controls partitioning
	replica groups for reliability

## why is this high-level design useful?

	1000s of disks of space
	store giant blobs, or many big blobs
	1000s of servers/disks/arms of parallel throughput
	can expand over time -- reconfiguration
	large pool of storage servers for instance replacement after failure

## what should we want to know from the paper?

* API?
* layout?
* finding data?
* add a server?
* replication?
* failure handling?
* failure model?
* consistent reads/writes? (i.e. does a read see latest write?)
* config mgr failure handling?
* good performance?
* useful for apps?

## Q&A

### API

*   does the abstract's 2 GByte/sec per client seem impressive?

    how fast can you read a file from Athena AFS? (abt 10 MB/sec)
    how fast can you read a typical hard drive?
    how fast can typical networks move data?

> * 每个磁盘的速度大概是100MB/s,千兆网卡的速度可以达到100MB/s, 万兆网卡的速度可以达到1000MB/s。
> * 这是因为FDS最大并行化磁盘。

*   abstract claims recover from lost disk (92 GB) in 6.2 seconds

    that's 15 GByte / sec
    impressive?
    how is that even possible? that's 30x the speed of a disk!
    who might care about this metric?

> * lost dick上每个数据的副本都保存在不同的磁盘上，Recovery在多个磁盘上同时进行。150个磁盘读，150个磁盘写，可以达到15GB/s.
> * 当recover的速度越快，数据的持久性就越高.

*   why are 128-bit blob IDs a nice interface? why not file names?

> * 使用128-bit最为blob ID，做HASH之后离散性更好，接口比较统一，可以被其他系统对接。


*   why do 8 MB tracts make sense?

> * 在文中的系统中以8M为tracts的大小，则对blob的随机操作和顺序操作的的响应速度几乎没有差别。

*	what kinds of client applications is the API aimed at? and not aimed at?

> * 分布式应用，处理大数据的应用，而不是使用POSIX接口的应用

### Layout


*	why have tracts at all? why not store each blob on just one server? what kinds of apps will benefit from striping? what kinds of apps won't?

> * 条带化
> * 把每个blob只放在一个server会造成性能瓶颈，不能利用并行化
> * 条带化利于需要读取大量数据的应用

*	how fast will a client be able to read a single tract?

> * 因为每个tract保存在磁盘中，所以读取一个tract的速度跟磁盘的速度有关，最快是100MB/S

*	where does the abstract's single-client 2 GB number come from?

> * 读取多个tracts，多个磁盘并行操作。

*	why not the UNIX i-node approach?   

    store an array per blob, indexed by tract #, yielding tractserver
    so you could make per-tract placement decisions
     e.g. write new tract to most lightly loaded server

> * 导致元数据太多


*	why not hash(b + t)?

> * This effectively selected a TLT entry independently at random for each tract, producing a binomial rather than a uniform distribution of tracts on tractservers.

*	how many TLT entries should there be?
   how about n = number of tractservers?
   why do they claim this works badly? Section 2.2

> * Section 3.3中
> * 不仅要考虑tract是否均匀分布在所有的TLT entries上，也要考虑tractserver是否均匀分布在所有的TLT entries上。
> * 最好的TLT应该有O(n^2)个entries，每一对disks都会出现在TLT的entry上。
> * **要保证在每个tractserver上在更多的TLT entry上，这样才能使得recovery速度更快。**



*	why is the paper's n^2 scheme better?

    TLT with n^2 entries, with every server pair occuring once
    How long will repair take?
    What are the risks if two servers fail?

> * 有n个disk
> * 在n^2 shceme中，在TLT中有n^2个TLT  entries。每个disk在2n个TLT entries中。
> * 设所有tracts的总容量是V，则每个disk上有 2V/n 的数据。
> * 当一个disk故障时，需要系统需要复制 2V/n 的数据，才能收敛到原来的状态。
> * 在其他的每块disk上，都含有n分之一这2V/n 的数据。
> * 重新生成2n个TLT entries，则剩下的disk就会在 2n+2 个TLT entries中，每个disk需要传输 4 个TLT entries的数据(写入2个，写出2个)，
大概需要是 4V/n^2 的数据量，大概需要 4V/(m*n^2) 的时间。

*	why do they actually use a minimum replication level of 3?

    same n^2 table as before, third server is randomly chosen
    What effect on repair time?
    What effect on two servers failing?
    What if three disks fail?

> * 假设TLT的使用n^2 scheme,且副本数为2，则任意坏两块磁盘，则必定有两个TLT entry上的数据丢失(思考一下为什么)。
> * 当副本数为3时，不影响recovery时间。
> * 当两个disk故障时，不会有TLT entry上的数据丢失。
> * 当两个disk故障时，在recovery window坏第三块disk，会有2/n的概率导致数据丢失。(思考为什么是2/n)


### Adding a tractserver

    To increase the amount of disk space / parallel throughput
    Metadata server picks some random TLT entries
    Substitutes new server for an existing server in those TLT entries

*	how long will adding a tractserver take?

> * 假设TLT使用的是n^2 scheme，则每个disk需要有2V/n的数据，则需要花费 2V/(n*m) 的时间。(所有tracts的总容量是V, n是磁盘个数，m是磁盘的传输速率)

*	what about client writes while tracts are being transferred?

    receiving tractserver may have copies from client(s) and from old srvr
    how does it know which is newest?

> * client的数据是最新的


*	what if a client reads/writes but has an old tract table?

> * TLT entry有version号，假如tractserver发现version不对，会拒绝请求。

### Replication

    A writing client sends a copy to each tractserver in the TLT.
    A reading client asks one tractserver.

*	why don't they send writes through a primary?

> * 牺牲一致性，提高性能.

*	what problems are they likely to have because of lack of primary?
   why weren't these problems show-stoppers?

> * blob的元数据操作需要primary。因为这些操作需要强一致性。

*	why not just pick one replacement server?

> * 假如选择一个替换的disk, 数据都会写到这个disk中，disk的传输速度就是瓶颈。
> * 最完美的recovery是需要所有disk并行传输数据。

*	how long will it take to copy all the tracts?

> * 需要 4V/(m*n^2) 的时间，详细解释看上文。

*	if a tractserver's net breaks and is then repaired, might srvr serve old data?

> * 看不懂问题....


*	if a server crashes and reboots with disk intact, can contents be used?

    e.g. if it only missed a few writes?
    3.2.1's "partial failure recovery"
    but won't it have already been replaced?
    how to know what writes it missed?

> * 在内存中记录tract的version号

*	when is it better to use 3.2.1's partial failure recovery?

> * tansient failures, the tractserver later returns to service.

*  What happens when the metadata server crashes?

> * 等待metadata server重启，并且重构TLT

*	while metadata server is down, can the system proceed?

> * 可以, client上有TLT的缓存.

*	is there a backup metadata server?

> * Network partitions complicate recovery from  metadata server failures. A simple primary/backup scheme is not safe because two active metadata servers
will cause cluster corruption. Our current solution is simple: only one metadata server is allowed to execute at a time. If it fails, an operator must 
ensure the old one is decommissioned and start a new one. We are experimenting with using Paxos leader election to safely make this process
automatic and reduce the impact to availability. Once a new metadata server starts, tractservers contact it with
their TLT assignments. After a timeout period, the new metadata server reconstructs the old TLT.

*	how does rebooted metadata server get a copy of the TLT?

> * 遍历所有的tractservers，得到TLT entry的内容

*	does their scheme seem correct?

### Random issues

*	is the metadata server likely to be a bottleneck?

> * 不是operation的关键路径，不是瓶颈.

*	why do they need the scrubber application mentioned in 2.3? why don't they delete the tracts when the blob is deleted?can a blob be written after it is deleted?

> * 因为元数据记录blob中每个tract是否存在. 当删除blob时，需要查询blob所有的tract是否存在，这会消耗很多时间。

### Performance

*	how do we know we're seeing "good" performance? what's the best you can expect?

*	limiting resource for 2 GB / second single-client?

> * Each computer in our cluster has a dual-port 10Gbps NIC.

*	Figure 4a: why starts low? why goes up? why levels off? why does it level off at that particular performance?

> * 总的吞吐量和磁盘的数量有关.

*	Figure 4b shows random r/w as fast as sequential (Figure 4a). is this what you'd expect?

> * 因为每个tract的大小是8MB，所以随机操作和顺序操作已经没有差别了(单副本)。

*	why are writes slower than reads with replication in Figure 4c?

> * 因为是三副本，每次写操作需要写三份数据.

*	where does the 92 GB in 6.2 seconds come from?

    Table 1, 4th column
    that's 15 GB / second, both read and written
    1000 disks, triple replicated, 128 servers?
    what's the limiting resource? disk? cpu? net?

> * 理论上的时间是 4V/（m*n^2）= 2*92TB/(100MB*1000^2) = 3.64 second
> * 每个磁盘输出输入的 364MB, 假设每个server上有10个disk，则每个server输出输入的总流量是 3640MB
> * 瓶颈在于网络







