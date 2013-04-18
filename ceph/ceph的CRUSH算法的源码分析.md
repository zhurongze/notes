# 1 源文件分析

分析的ceph版本是ceph-0.41。

 
## 1.1 rule与bucket的关系

http://ceph.newdream.net/wiki/Custom_data_placement_with_CRUSH

 
## 1.2 crush目录下的文件

* builder.c
* builder.h
* crush.c
* crush.h
* CrushWrapper.cc
* CrushWrapper.h
* CrushWrapper.i
* grammar.h
* hash.c
* hash.h
* mapper.c
* mapper.h
* sample.txt
* types.h

 
**CRUSH头文件的包含关系**

![CRUSH头文件的包含关系](http://way4ever.com/wp-content/uploads/2013/04/CRUSH.png)

 
## 1.3 crush.h中

定义了crush_rule、crush_bucket、crush_bucket_uniform、crush_bucket_list、crush_bucket_tree、crush_bucket_straw、crush_map等数据结构。

其中crush_bucket结构

<pre name="code" class="cpp"> 
struct crush_bucket {
    __s32 id;        /* this'll be negative */
    __u16 type;      /* non-zero; type=0 is reserved for devices */
    __u8 alg;        /* one of CRUSH_BUCKET_* */
    __u8 hash;       /* which hash function to use, CRUSH_HASH_* */
    __u32 weight;    /* 16-bit fixed point */
    __u32 size;      /* num items */ //假如size为0，说明它不包含item。
    __s32 *items;   //数组，它包含item的id，这些id可能都是负数，也可能都是自然数。
                    //假如是负数，表示它包含的item都是bucket，假如是自然数，表示它的item都是device。

    /*
     * cached random permutation: used for uniform bucket and for
     * the linear search fallback for the other bucket types.
     */
    __u32 perm_x;  /* @x for which *perm is defined */
    __u32 perm_n;  /* num elements of *perm that are permuted/defined */
    __u32 *perm;
};

</pre>
 
## 1.4 crush.c中

* crush_get_bucket_item_weight函数用于获取指定bucket中的item的weight。
* crush_calc_parents函数计算device和bucket的parent，并记录。
* crush_destroy_bucket函数是用于销毁bucket数据结构。
* crush_destroy函数用于销毁crush_map结构。

 
## 1.5 build.c中

构造、操作 crush_map、rule、bucket。

* crush_craete函数用于创建一个crush_map结构。也就是申请crush_map结构的空间。
* crush_finalize函数用于最后初始化一个crush_map结构。当所有的bucket都被添加到crush_map之后才调用该函数。该函数执行三个操作
    1. 计算max_devices的值。
    2. 分配device_parents数组和bucket_parents数组的空间。
    3. 为crush_map中的device和bucket创建parent map(调用的是crush_calc_parents函数)。
* crush_make_rule函数用于创建一个crush_rule结构。
* crush_rule_set_step函数对一个crush_rule结构设置step(包括op、arg1、arg2)。
* crush_add_rule函数往crush_map中添加rule。
* crush_get_next_bucket_id函数寻找空槽位(返回未使用的bucket id)。
* crush_add_bucket函数添加bucket到crush_map上。
* crush_remove_bucket函数从crush_map中移除指定的bucket。在移除bucket前，bucket中的所有item都已被移除，它的weight为0，因此它不需要再调整weight。
* crush_make_bucket函数创建一个bucket结构。需要指定CRUSH算法(uniform、list、tree、straw)、hash算法、bucket类型(自定义的type)、size(包括多少个item)、这些items的id数组、这些items的weights。
* crush_bucket_add_item函数往bucket中添加item。
* crush_bucket_adjust_item_weight函数调整bucket中指定item的weight，并相应调整bucket的weight。
* crush_reweight_bucket函数计算crush_map中指定bucket的weight。
* crush_bucket_remove_item函数从bucket中移除指定的item。并调整bucket的weight。

 
## 1.6 在hash.h、hash.c中

使用Robert Jenkins的HASH算法，地址是http://burtleburtle.net/bob/hash/evahash.html

* crush_hash32函数计算1个32位参数的哈希值，返回的哈希值也是32位的。
* crush_hash32_2函数计算2个32位参数的哈希值，返回的哈希值也是32位的。
* crush_hash32_3函数计算3个32位参数的哈希值，返回的哈希值也是32位的。
* crush_hash32_4函数计算4个32位参数的哈希值，返回的哈希值也是32位的。
* crush_hash32_5函数计算5个32位参数的哈希值，返回的哈希值也是32位的。

 
## 1.7 在mapper.h、mapper.c中

* crush_find_rule函数是根据指定的ruleset、type、size在crush_map中找到相应的的crush_rule id。
* crush_do_rule函数根据指定的输入(哈希值)、rule在crush_map中执行rule，并返回相应的数组(包含一组OSD集合)。本文会重点分析这个函数。

 
## 1.8 在CrushWrapper.h、CrushWrapper.c中

包含了CrushWrapper类(这个类是Crush算法的包装类)，主要包括对item、rule、map、bucket的操作，还有encode和decode操作，把一些参数编码成为bufferlist，或者把bufferlist解码成原参数。最重要的是do_rule函数，该函数根据rule、PG号等一些参数返回多个OSD。

 
# 2 CRUSH算法流程

假设我组建一套存储系统，有3个机架(host)，每个机架上有4台主机(host)，每个主机上有2个磁盘(device)，则一共有24个磁盘。预计的扩展方式是添加主机或者添加机架。

![存储系统物理架构举例](http://way4ever.com/wp-content/uploads/2013/04/racks.png)
 
## 2.1 创建crush_map

我们的bucket有三种: root、rack、host。root包含的item是rack，root的结构是straw。rack包含的item是host，rack的结构是tree。host包括的item是device，host的结构式uniform。这是因为每个host包括的device的数量和权重是一定的，不会改变，因此要为host选择uniform结构，这样计算速度最快。

**执行下面的命令，得到crush_map**

    # crushtool --num_osds 24 -o crushmap --build host uniform 2 rack tree 4 root straw 0
    # crushtool -d crushmap -o flat.txt

 

**查看crush_map**

    root@ceph-01:~# cat flat.txt
    
<pre name="code" class="shell"> 
# begin crush map

# devices
device 0 device0
device 1 device1
device 2 device2
device 3 device3
device 4 device4
device 5 device5
device 6 device6
device 7 device7
device 8 device8
device 9 device9
device 10 device10
device 11 device11
device 12 device12
device 13 device13
device 14 device14
device 15 device15
device 16 device16
device 17 device17
device 18 device18
device 19 device19
device 20 device20
device 21 device21
device 22 device22
device 23 device23

# types
type 0 device
type 1 host
type 2 rack
type 3 root

# buckets
host host0 {
    id -1        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device0 weight 1.000 pos 0
    item device1 weight 1.000 pos 1
}
host host1 {
    id -2        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device2 weight 1.000 pos 0
    item device3 weight 1.000 pos 1
}
host host2 {
    id -3        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device4 weight 1.000 pos 0
    item device5 weight 1.000 pos 1
}
host host3 {
    id -4        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device6 weight 1.000 pos 0
    item device7 weight 1.000 pos 1
}
host host4 {
    id -5        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device8 weight 1.000 pos 0
    item device9 weight 1.000 pos 1
}
host host5 {
    id -6        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device10 weight 1.000 pos 0
    item device11 weight 1.000 pos 1
}
host host6 {
    id -7        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device12 weight 1.000 pos 0
    item device13 weight 1.000 pos 1
}
host host7 {
    id -8        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device14 weight 1.000 pos 0
    item device15 weight 1.000 pos 1
}
host host8 {
    id -9        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device16 weight 1.000 pos 0
    item device17 weight 1.000 pos 1
}
host host9 {
    id -10        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device18 weight 1.000 pos 0
    item device19 weight 1.000 pos 1
}
host host10 {
    id -11        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device20 weight 1.000 pos 0
    item device21 weight 1.000 pos 1
}
host host11 {
    id -12        # do not change unnecessarily
    # weight 2.000
    alg uniform    # do not change bucket size (2) unnecessarily
    hash 0    # rjenkins1
    item device22 weight 1.000 pos 0
    item device23 weight 1.000 pos 1
}
rack rack0 {
    id -13        # do not change unnecessarily
    # weight 8.000
    alg tree    # do not change pos for existing items unnecessarily
    hash 0    # rjenkins1
    item host0 weight 2.000 pos 0
    item host1 weight 2.000 pos 1
    item host2 weight 2.000 pos 2
    item host3 weight 2.000 pos 3
}
rack rack1 {
    id -14        # do not change unnecessarily
    # weight 8.000
    alg tree    # do not change pos for existing items unnecessarily
    hash 0    # rjenkins1
    item host4 weight 2.000 pos 0
    item host5 weight 2.000 pos 1
    item host6 weight 2.000 pos 2
    item host7 weight 2.000 pos 3
}
rack rack2 {
    id -15        # do not change unnecessarily
    # weight 8.000
    alg tree    # do not change pos for existing items unnecessarily
    hash 0    # rjenkins1
    item host8 weight 2.000 pos 0
    item host9 weight 2.000 pos 1
    item host10 weight 2.000 pos 2
    item host11 weight 2.000 pos 3
}
root root {
    id -16        # do not change unnecessarily
    # weight 24.000
    alg straw
    hash 0    # rjenkins1
    item rack0 weight 8.000
    item rack1 weight 8.000
    item rack2 weight 8.000
}

# rules
rule data {
    ruleset 1
    type replicated
    min_size 2
    max_size 2
    step take root
    step chooseleaf firstn 0 type host
    step emit
}

# end crush map
</pre>


## 2.2 CRUSH_MAP结构解析

crush_map结构中的**buckets**成员是bucket结构指针数组，buckets成员保存了上面这些bucket结构的指针。上面这些bucket结构的指针在buckets中的下标是 [-1-id]。buckets数组的元素如下所示。

{ &host0, &host1, &host2, ... , &host11, &rack0, &rack1, &rack2, &root}

<table>
    <tr>
        <th>pos</th>
        <td>0</td>
        <td>1</td>
        <td>...</td>
        <td>11</td>
        <td>12</td>
        <td>13</td>
        <td>14</td>
        <td>15</td>
    </tr>
    <tr>
        <th>&bucket</th>
        <td>&host0</td>
        <td>&host1</td>
        <td>...</td>
        <td>&host11</td>
        <td>&rack0</td>
        <td>&rack1</td>
        <td>&rack2</td>
        <td>&root</td>
    </tr>
    <tr>
        <th>&bucked_id</th>
        <td>-1</td>
        <td>-2</td>
        <td>...</td>
        <td>-12</td>
        <td>-13</td>
        <td>-14</td>
        <td>-15</td>
        <td>-16</td>
    </tr>
</table>

bucket的id使用负数是为了和device区分，因为bucket的item可以是device，也可以是bucket。比如host0的item数组中包含的元素是{0, 1}，它们是device0、device1。而rack2的item数组中包含的元素是{-9, -10, -11, -12}，它们是host8、host9、host10、host11。
 

## 2.3 数据分配rule

它的数据分配rule包括3条step。

> step take root

> step chooseleaf firstn 0 type host

> step emit

它的副本数是2。

 
## 2.4 crush_do_rule函数的解析

根据参数x(object_id或者是pg_id)，crush_do_rule返回一组device的id(这组device用于保存object的副本)。crush_do_rule还有其他参数。

* const struct crush_map *map ，反映了当前存储系统的架构。
* int ruleno， 选择哪种rule。
* int x
* int force， 指定副本在哪个device上，假如为-1，则不指定。
* int *result，crush_do_rule函数会把device的id放到这个数组中。
* int result_max， 最大副本数。
* const __u32 *weight， 当前所有device权重的数组。

crush_do_rule函数在mapper.cc中。在第509行中，CRUSH依次执行每条step。

第一个step是"step take root"，因此CRUSH会执行512~523行，因为"root"所对应的id是-16，因此w[0] = -16，wsize = 1。

然后CRUSH执行第二条 "step chooseleaf firstn 0 type host"， CRUSH会执行525~588行代码。CRUSH执行到541行时，firstn = 1， recurse_to_leaf = 1。

因为wsize = 1， 因此542行的循环只执行一次。

执行到568行时，numrep = 2， j = 0， i = 0 。

执行到569行时，会调用crush_choose函数。map->buckets[-1-w[i]] = &root。

 

**crush_choose函数的参数如下所示**

* const struct crush_map *map，反映了当前存储系统的架构。
* struct crush_bucket *bucket，在这个bucket中选择item，crush_do_rule函数传给crush_choose的bucket是&root。&root中的item是{&rack0，&rack1，&rack2}，它们的id是{-13，-14，-15}。
* const __u32 *weight， 所有device的权重
* int x
* int numrep，crush_do_rule函数传给crush_choose的numrep是2。
* int type，crush_do_rule函数传给crush_choose的type是"host"。
* int *out，输出。crush_choose会在bucket中选择numrep - outpos个item，并把它们放到out数组中。
* int outpos，被选择的items放到 out[outpos]、out[outpos+1]、... 、out[numrep - 1]。
* int firstn，"first n"选项，决定了r'的值。
* int recurse_to_leaf，假如recurse_to_leaf为真，则CRUSH会继续遍历这些item中的每个item(对每个item递归调用crush_choose)，并从每个item中选出一个device。
* int *out2，假如recurse_to_leaf为真，则CRUSH会把找到的devices放到out2数组中。

crush_choose会找到2个host，并在每个host中找到一个device。

当crush_choose第一次执行到356行时，in是&root bucket，r = 0。调用crush_bucket_choose函数。

crush_bucket_choose函数根据root的类型，选择调用bucket_straw_choose函数。假设bucket_straw_choose函数经过计算在&root中选择了&rack0，并返回rack0的id(-13)。

crush_choose函数执行到367行时，判断rack0的type是否是"host"，因为rack0的type是"rack"，因此CRUSH会再次执行322~356行的代码，并调用crush_bucket_choose函数。crush_bucket_choose函数根据rack0的类型，选择调用bucket_tree_choose函数。假设bucket_tree_choose函数经过计算在rack0中选择host2，并返回host2的id(-3)。

因为recurse_to_leaf是1，因此CRUSH会执行384~399行，在host2中选择一个device(假设是device4)，并把device的id放到out2数组中。

CRUSH第二次执行365行时，in是&root bucket，r = 1。这次过程不再复述，假设CRUSH在root中选择了rack1，在rack1中选择了host6，在host6中选择了device13。

返回crush_do_rule函数，执行579~588行，则w数组中的元素是{4, 13}。

最后CRUSH会执行第三个 "step emit"，执行591~597行，把复制w数组到result数组上。

{4，13}代表device4和device13，这表明x对应的设备是{device4, device13}。

CRUSH算法完成了x到{device4, device13}的映射。
