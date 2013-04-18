**目录**

* 1 测试目的
* 2 测试环境
* 3 测试工具
* 4 测试方案
* 5 测试脚本
* 6 测试结果
* 7 结论分析

# 1. 测试目的 #

测试由SAS磁盘组成的RAID5和SSD组成RAID5的性能，并进行对比。最后分析结果。

# 2. 测试环境 #
## 2.1 测试对象 ##

测试对象是两个RAID5。
* 第一个RAID5由7块SAS硬盘组成。
* 第二个RAID5由9块SSD硬盘组成。

## 2.2 硬件配置 ##
### 2.2.1 服务器 ###

CPU是 Intel(R) Xeon(R) CPU E5620  @ 2.40GHz

内存 48GB

### 2.2.2 RAID卡 ###

是HP服务器上集成的RAID卡，型号是Dell PowerEdge RAID Controller H700 Integrated，序列号是1BQ03ZY。

### 2.2.3 SAS的RAID5信息 ###

<pre name="code" class="shell">
RAID Level          : Primary-5, Secondary-0, RAID Level Qualifier-3
Size                : 1.633 TB
Parity Size         : 278.875 GB
State               : Optimal
Strip Size          : 64 KB
Number Of Drives    : 7
Span Depth          : 1
Default Cache Policy: WriteBack, ReadAdaptive, Direct, No Write Cache if Bad BBU
Current Cache Policy: WriteBack, ReadAdaptive, Direct, No Write Cache if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Disk's Default
</pre>

### 2.2.4 SSD的RAID5信息 ###

<pre name="code" class="shell">
RAID Level          : Primary-5, Secondary-0, RAID Level Qualifier-3
Size                : 1.160 TB
Parity Size         : 148.5 GB
State               : Optimal
Strip Size          : 64 KB
Number Of Drives    : 9
Span Depth          : 1
Default Cache Policy: WriteBack, ReadAdaptive, Direct, No Write Cache if Bad BBU
Current Cache Policy: WriteBack, ReadAdaptive, Direct, No Write Cache if Bad BBU
Default Access Policy: Read/Write
Current Access Policy: Read/Write
Disk Cache Policy   : Disk's Default
</pre>

### 2.2.5 硬盘 ###

#### 2.2.5.1 SAS硬盘 ####

<pre name="code" class="shell">
PD Type: SAS
Raw Size: 279.396 GB [0x22ecb25c Sectors]
Non Coerced Size: 278.896 GB [0x22dcb25c Sectors]
Coerced Size: 278.875 GB [0x22dc0000 Sectors]
Firmware state: Online, Spun Up
Device Firmware Level: ES64
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x5000c50043e6d731
SAS Address(1): 0x0
Connected Port Number: 0(path0) 
Inquiry Data: SEAGATE ST3300657SS     ES646SJ3TLCJ            
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 6.0Gb/s 
Link Speed: 6.0Gb/s 
Media Type: Hard Disk Device
Drive Temperature :45C (113.00 F)
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Drive's write cache : Disabled
Port-0 :
Port status: Active
Port's Linkspeed: 6.0Gb/s 
Port-1 :
Port status: Active
Port's Linkspeed: Unknown 
Drive has flagged a S.M.A.R.T alert : No
</pre>


#### 2.2.5.2 SSD硬盘 ####

<pre name="code" class="shell">
PD Type: SATA
Raw Size: 149.049 GB [0x12a19eb0 Sectors]
Non Coerced Size: 148.549 GB [0x12919eb0 Sectors]
Coerced Size: 148.5 GB [0x12900000 Sectors]
Firmware state: Online, Spun Up
Device Firmware Level: 0362
Shield Counter: 0
Successful diagnostics completion on :  N/A
SAS Address(0): 0x500065b36789abe8
Connected Port Number: 0(path0) 
Inquiry Data: BTPR1455017J160DGN  INTEL SSDSA2CW160G3                     4PC10362
FDE Enable: Disable
Secured: Unsecured
Locked: Unlocked
Needs EKM Attention: No
Foreign State: None 
Device Speed: 3.0Gb/s 
Link Speed: 3.0Gb/s 
Media Type: Solid State Device
Drive:  Not Certified
Drive Temperature : N/A
PI Eligibility:  No 
Drive is formatted for PI information:  No
PI: No PI
Drive's write cache : Disabled
Drive's NCQ setting : Disabled
Port-0 :
Port status: Active
Port's Linkspeed: 3.0Gb/s 
Drive has flagged a S.M.A.R.T alert : No
</pre>

## 2.3 软件配置 ##

由于被测试的的RAID5已经被正在使用，因此不能直接测试裸设备。

RAID5上创建有ext4文件系统。

# 3 测试工具 #

一般的测试工具有iometer、fio、bonnie++、iozone、orion。

最后选择**fio**。

fio的《HOWTO》文档在 http://www.bluestop.org/fio/HOWTO.txt

使用fio的例子有 http://marcitland.blogspot.com/2011/06/scst-ssd-arrays.html


# 4 测试方案 #

主要测试的是IOPS。

SAS和SSD都有的测试项目：

* 顺序读
* 顺序写
* CFQ随机读
* CFQ随机写

SSD中还要测试deadline,noop的影响，SSD还要单独加的测试项目有：

* deadline随机读
* deadline随机写
* noop随机读
* noop随机写

SSD为了和其他测评(http://marcitland.blogspot.com/2011/06/scst-ssd-arrays.html) 进行对比，还需要进行的测试项目有：

* noop libaio  4k随机读
* noop libaio  4k随机写

通过改变fio参数配置可以测试出RAID组的IOPS峰值，但是这次是对比测试，所以应该每个测试项目都有规定的参数配置。

# 5 测试脚本 #
fio的测试脚本。

## 5.1 顺序读 ##

<pre name="code" class="shell">
[global]
runtime=120
time_based
group_reporting
directory=/var/fio-test
ioscheduler=cfq
refill_buffers
[innodb-data]
filename=test-innodb.dat
bs=16K
ioengine=psync
rw=read
size=10G
direct=1
numjobs=16
</pre>
## 顺序写 ##
<pre name="code" class="shell">
[global]
runtime=120
time_based
group_reporting
directory=/var/fio-test
ioscheduler=cfq
refill_buffers
[innodb-data]
filename=test-innodb.dat
bs=16K
ioengine=psync
rw=write
size=10G
direct=1
numjobs=1
</pre>

## 5.2 CFQ随机读 ##

<pre name="code" class="shell">
[global]
runtime=120
time_based
group_reporting
directory=/var/fio-test
ioscheduler=cfq
refill_buffers
[innodb-data]
filename=test-innodb.dat
bs=16K
ioengine=psync
rw=randread
size=1G
direct=1
numjobs=128
</pre>

## 5.3 CFQ随机写 ##

<pre name="code" class="shell">
[global]
runtime=120
time_based
group_reporting
directory=/var/fio-test
ioscheduler=cfq
refill_buffers
[innodb-data]
filename=test-innodb.dat
bs=16K
ioengine=psync
rw=randwrite
size=10G
direct=1
numjobs=32
</pre>

## 5.4 deadline随机读 ##

<pre name="code" class="shell">
[global]
runtime=120
time_based
group_reporting
directory=/var/fio-test
ioscheduler=deadline
refill_buffers
[innodb-data]
filename=test-innodb.dat
bs=16K
ioengine=psync
rw=randread
size=10G
direct=1
numjobs=128
</pre>

## 5.5 deadline随机写 ##

<pre name="code" class="shell">
[global]
runtime=120
time_based
group_reporting
directory=/var/fio-test
ioscheduler=deadline
refill_buffers
[innodb-data]
filename=test-innodb.dat
bs=16K
ioengine=psync
rw=randwrite
size=10G
direct=1
numjobs=32
</pre>

## 5.6 noop随机读 ##

<pre name="code" class="shell">
[global]
runtime=120
time_based
group_reporting
directory=/var/fio-test
ioscheduler=noop
refill_buffers
[innodb-data]
filename=test-innodb.dat
bs=16K
ioengine=psync
rw=randread
size=10G
direct=1
numjobs=128
</pre>

## 5.7 noop随机写 ##

<pre name="code" class="shell">
[global]
runtime=120
time_based
group_reporting
directory=/var/fio-test
ioscheduler=noop
refill_buffers
[innodb-data]
filename=test-innodb.dat
bs=16K
ioengine=psync
rw=randwrite
size=10G
direct=1
numjobs=32
</pre>

## 5.8 noop libaio  4k随机读 ##

<pre name="code" class="shell">
[global]
runtime=120
time_based
group_reporting
directory=/var/fio-test
ioscheduler=noop
refill_buffers
[innodb-data]
filename=test-innodb.dat
bs=4K
ioengine=libaio
rw=randread
size=10G
direct=1
numjobs=1
iodepth=64
</pre>

## 5.9 noop libaio  4k随机写 ##

<pre name="code" class="shell">
[global]
runtime=120
time_based
group_reporting
directory=/var/fio-test
ioscheduler=noop
refill_buffers
[innodb-data]
filename=test-innodb.dat
bs=4K
ioengine=libaio
rw=randwrite
size=10G
direct=1
numjobs=1
iodepth=64
</pre>
 

# 6 测试结果 #

## 6.1 SAS RAID5 ##
### 6.1.1 顺序读 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=16): err= 0: pid=22420
  read : io=42586MB, bw=363387KB/s, iops=22711 , runt=120003msec
    clat (usec): min=41 , max=632714 , avg=702.41, stdev=11985.50
     lat (usec): min=42 , max=632714 , avg=702.58, stdev=11985.51
    clat percentiles (usec):
     |  1.00th=[   48],  5.00th=[   49], 10.00th=[   50], 20.00th=[   51],
     | 30.00th=[   52], 40.00th=[   53], 50.00th=[   55], 60.00th=[   57],
     | 70.00th=[   62], 80.00th=[   64], 90.00th=[   75], 95.00th=[  107],
     | 99.00th=[  253], 99.50th=[  524], 99.90th=[228352], 99.95th=[259072],
     | 99.99th=[321536]
    bw (KB/s)  : min=   29, max=116274, per=6.34%, avg=23023.26, stdev=13540.79
    lat (usec) : 50=7.63%, 100=86.77%, 250=4.58%, 500=0.51%, 750=0.05%
    lat (usec) : 1000=0.01%
    lat (msec) : 2=0.03%, 4=0.03%, 10=0.06%, 20=0.01%, 50=0.02%
    lat (msec) : 100=0.02%, 250=0.22%, 500=0.06%, 750=0.01%
  cpu          : usr=0.51%, sys=2.40%, ctx=2737544, majf=0, minf=471
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=2725474/w=0/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
   READ: io=42586MB, aggrb=363387KB/s, minb=363387KB/s, maxb=363387KB/s, mint=120003msec, maxt=120003msec

Disk stats (read/write):
  sdb: ios=2722328/233, merge=1552/64, ticks=1637980/90548, in_queue=1728388, util=99.99%
</pre>

### 6.1.2 顺序写 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=1): err= 0: pid=22985
  write: io=22942MB, bw=195766KB/s, iops=12235 , runt=120001msec
    clat (usec): min=44 , max=24093 , avg=74.27, stdev=58.43
     lat (usec): min=44 , max=24093 , avg=74.46, stdev=58.43
    clat percentiles (usec):
     |  1.00th=[   52],  5.00th=[   53], 10.00th=[   54], 20.00th=[   55],
     | 30.00th=[   56], 40.00th=[   58], 50.00th=[   67], 60.00th=[   81],
     | 70.00th=[   85], 80.00th=[   89], 90.00th=[  100], 95.00th=[  108],
     | 99.00th=[  139], 99.50th=[  258], 99.90th=[  278], 99.95th=[  282],
     | 99.99th=[  326]
    bw (KB/s)  : min=172320, max=247680, per=99.94%, avg=195645.72, stdev=14009.57
    lat (usec) : 50=0.03%, 100=89.07%, 250=10.39%, 500=0.52%, 750=0.01%
    lat (usec) : 1000=0.01%
    lat (msec) : 10=0.01%, 20=0.01%, 50=0.01%
  cpu          : usr=10.10%, sys=16.97%, ctx=1469742, majf=0, minf=21
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=1468257/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=22942MB, aggrb=195765KB/s, minb=195765KB/s, maxb=195765KB/s, mint=120001msec, maxt=120001msec

Disk stats (read/write):
  sdb: ios=0/1468191, merge=0/93, ticks=0/91048, in_queue=90156, util=72.75%
</pre>

### 6.1.3 CFQ随机读 ###
<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=128): err= 0: pid=24728
  read : io=10040MB, bw=85589KB/s, iops=5349 , runt=120117msec
    clat (usec): min=51 , max=3238.9K, avg=23905.90, stdev=45355.49
     lat (usec): min=51 , max=3238.9K, avg=23906.07, stdev=45355.49
    clat percentiles (usec):
     |  1.00th=[   70],  5.00th=[   89], 10.00th=[ 1816], 20.00th=[ 3376],
     | 30.00th=[ 5088], 40.00th=[ 7776], 50.00th=[11328], 60.00th=[16320],
     | 70.00th=[23680], 80.00th=[35584], 90.00th=[59136], 95.00th=[86528],
     | 99.00th=[164864], 99.50th=[207872], 99.90th=[342016], 99.95th=[464896],
     | 99.99th=[2056192]
    bw (KB/s)  : min=    5, max= 1682, per=0.80%, avg=683.02, stdev=211.68
    lat (usec) : 100=5.28%, 250=1.16%, 500=0.60%, 750=0.26%, 1000=0.12%
    lat (msec) : 2=3.85%, 4=12.44%, 10=23.20%, 20=18.56%, 50=21.67%
    lat (msec) : 100=9.20%, 250=3.38%, 500=0.24%, 750=0.02%, 1000=0.01%
    lat (msec) : 2000=0.01%, >=2000=0.01%
  cpu          : usr=0.03%, sys=0.11%, ctx=667321, majf=0, minf=3773
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=642543/w=0/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
   READ: io=10040MB, aggrb=85588KB/s, minb=85588KB/s, maxb=85588KB/s, mint=120117msec, maxt=120117msec

Disk stats (read/write):
  sdb: ios=642520/46, merge=0/13, ticks=15337688/11268, in_queue=17222388, util=99.95%
</pre>

### 6.1.4 CFQ随机写 ###
<pre name="code" class="shell">
innodb-data: (g=0): rw=randwrite, bs=16K-16K/16K-16K, ioengine=psync, iodepth=1
fio 2.0.7
Starting 32 processes
Jobs: 32 (f=32): [wwwwwwwwwwwwwwwwwwwwwwwwwwwwwwww] [100.0% done] [0K/17235K /s] [0 /1052  iops] [eta 00m:00s]
innodb-data: (groupid=0, jobs=32): err= 0: pid=32245
  write: io=2044.2MB, bw=17406KB/s, iops=1087 , runt=120256msec
    clat (usec): min=50 , max=1439.2K, avg=29404.01, stdev=109807.76
     lat (usec): min=50 , max=1439.2K, avg=29404.22, stdev=109807.77
    clat percentiles (usec):
     |  1.00th=[   61],  5.00th=[   63], 10.00th=[   64], 20.00th=[   68],
     | 30.00th=[   77], 40.00th=[   95], 50.00th=[  126], 60.00th=[  266],
     | 70.00th=[ 4320], 80.00th=[ 6752], 90.00th=[41216], 95.00th=[191488],
     | 99.00th=[610304], 99.50th=[790528], 99.90th=[1044480], 99.95th=[1155072],
     | 99.99th=[1236992]
    bw (KB/s)  : min=   16, max= 6464, per=3.26%, avg=566.74, stdev=575.48
    lat (usec) : 100=42.49%, 250=17.43%, 500=0.18%, 750=0.04%, 1000=0.08%
    lat (msec) : 2=0.28%, 4=7.75%, 10=17.52%, 20=3.39%, 50=1.16%
    lat (msec) : 100=1.91%, 250=3.93%, 500=2.35%, 750=0.93%, 1000=0.40%
    lat (msec) : 2000=0.16%
  cpu          : usr=0.04%, sys=0.12%, ctx=261706, majf=0, minf=769
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=130823/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=2044.2MB, aggrb=17405KB/s, minb=17405KB/s, maxb=17405KB/s, mint=120256msec, maxt=120256msec

Disk stats (read/write):
  sdb: ios=1/131084, merge=0/108, ticks=0/195496, in_queue=195580, util=96.76%
</pre>

## 6.2 SSD RAID5 ##

### 6.2.1 顺序读 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=16): err= 0: pid=16054
  read : io=44100MB, bw=376310KB/s, iops=23519 , runt=120004msec
    clat (usec): min=41 , max=560014 , avg=678.19, stdev=11623.99
     lat (usec): min=41 , max=560014 , avg=678.36, stdev=11623.99
    clat percentiles (usec):
     |  1.00th=[   48],  5.00th=[   49], 10.00th=[   50], 20.00th=[   51],
     | 30.00th=[   52], 40.00th=[   53], 50.00th=[   56], 60.00th=[   59],
     | 70.00th=[   62], 80.00th=[   64], 90.00th=[   75], 95.00th=[   87],
     | 99.00th=[  173], 99.50th=[  540], 99.90th=[224256], 99.95th=[259072],
     | 99.99th=[321536]
    bw (KB/s)  : min=   48, max=116305, per=6.35%, avg=23905.54, stdev=15041.24
    lat (usec) : 50=8.34%, 100=88.21%, 250=2.81%, 500=0.11%, 750=0.12%
    lat (usec) : 1000=0.02%
    lat (msec) : 2=0.02%, 4=0.01%, 10=0.01%, 20=0.01%, 50=0.03%
    lat (msec) : 100=0.03%, 250=0.22%, 500=0.06%, 750=0.01%
  cpu          : usr=0.53%, sys=2.38%, ctx=2829040, majf=0, minf=455
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=2822416/w=0/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
   READ: io=44100MB, aggrb=376309KB/s, minb=376309KB/s, maxb=376309KB/s, mint=120004msec, maxt=120004msec

Disk stats (read/write):
  sdb: ios=2816078/158, merge=698/148, ticks=1612092/77104, in_queue=1689012, util=99.98%
</pre>

### 6.2.2 顺序写 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=1): err= 0: pid=16469
  write: io=25757MB, bw=219787KB/s, iops=13736 , runt=120001msec
    clat (usec): min=46 , max=12234 , avg=65.28, stdev=37.39
     lat (usec): min=46 , max=12234 , avg=65.52, stdev=37.39
    clat percentiles (usec):
     |  1.00th=[   52],  5.00th=[   53], 10.00th=[   54], 20.00th=[   55],
     | 30.00th=[   55], 40.00th=[   55], 50.00th=[   56], 60.00th=[   57],
     | 70.00th=[   58], 80.00th=[   78], 90.00th=[   88], 95.00th=[  104],
     | 99.00th=[  143], 99.50th=[  237], 99.90th=[  258], 99.95th=[  266],
     | 99.99th=[  286]
    bw (KB/s)  : min=203168, max=251648, per=100.00%, avg=219814.36, stdev=5652.50
    lat (usec) : 50=0.01%, 100=93.74%, 250=5.94%, 500=0.30%, 750=0.01%
    lat (usec) : 1000=0.01%
    lat (msec) : 10=0.01%, 20=0.01%
  cpu          : usr=12.65%, sys=17.75%, ctx=1649968, majf=0, minf=21
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=1648416/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=25757MB, aggrb=219786KB/s, minb=219786KB/s, maxb=219786KB/s, mint=120001msec, maxt=120001msec

Disk stats (read/write):
  sdb: ios=0/1648261, merge=0/82, ticks=0/84016, in_queue=83260, util=67.70%
</pre>

### 6.2.3 CFQ随机读 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=128): err= 0: pid=16785
  read : io=130409MB, bw=1086.7MB/s, iops=69547 , runt=120007msec
    clat (usec): min=57 , max=9275.1K, avg=1837.55, stdev=13507.19
     lat (usec): min=58 , max=9275.1K, avg=1837.74, stdev=13507.19
    clat percentiles (usec):
     |  1.00th=[  916],  5.00th=[ 1064], 10.00th=[ 1144], 20.00th=[ 1240],
     | 30.00th=[ 1320], 40.00th=[ 1384], 50.00th=[ 1464], 60.00th=[ 1560],
     | 70.00th=[ 1672], 80.00th=[ 1864], 90.00th=[ 2768], 95.00th=[ 4704],
     | 99.00th=[ 6496], 99.50th=[ 7072], 99.90th=[ 8160], 99.95th=[ 8512],
     | 99.99th=[ 9536]
    bw (KB/s)  : min=    3, max=11378, per=0.79%, avg=8793.23, stdev=583.36
    lat (usec) : 100=0.01%, 250=0.01%, 500=0.01%, 750=0.17%, 1000=2.43%
    lat (msec) : 2=81.55%, 4=8.49%, 10=7.33%, 20=0.01%, 50=0.01%
    lat (msec) : 100=0.01%, 250=0.01%, 500=0.01%, 2000=0.01%, >=2000=0.01%
  cpu          : usr=0.25%, sys=3.12%, ctx=10784579, majf=0, minf=3754
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=8346173/w=0/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
   READ: io=130409MB, aggrb=1086.7MB/s, minb=1086.7MB/s, maxb=1086.7MB/s, mint=120007msec, maxt=120007msec

Disk stats (read/write):
  sdb: ios=8346017/46, merge=0/16, ticks=14542576/2012, in_queue=15500384, util=100.00%
</pre>

### 6.2.4 CFQ随机写 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=32): err= 0: pid=17598
  write: io=12963MB, bw=110615KB/s, iops=6913 , runt=120006msec
    clat (usec): min=48 , max=255977 , avg=4619.16, stdev=4684.01
     lat (usec): min=49 , max=255977 , avg=4619.37, stdev=4684.00
    clat percentiles (usec):
     |  1.00th=[   58],  5.00th=[   61], 10.00th=[   63], 20.00th=[   85],
     | 30.00th=[  143], 40.00th=[ 2544], 50.00th=[ 5728], 60.00th=[ 6560],
     | 70.00th=[ 7072], 80.00th=[ 7648], 90.00th=[ 9152], 95.00th=[11712],
     | 99.00th=[14784], 99.50th=[15936], 99.90th=[22912], 99.95th=[31616],
     | 99.99th=[152576]
    bw (KB/s)  : min=   79, max=25312, per=3.13%, avg=3464.55, stdev=2690.97
    lat (usec) : 50=0.01%, 100=25.29%, 250=7.81%, 500=5.03%, 750=0.41%
    lat (usec) : 1000=0.01%
    lat (msec) : 2=0.44%, 4=3.58%, 10=49.81%, 20=7.40%, 50=0.19%
    lat (msec) : 100=0.02%, 250=0.01%, 500=0.01%
  cpu          : usr=0.25%, sys=0.74%, ctx=1660025, majf=0, minf=769
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=829652/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=12963MB, aggrb=110614KB/s, minb=110614KB/s, maxb=110614KB/s, mint=120006msec, maxt=120006msec

Disk stats (read/write):
  sdb: ios=0/829541, merge=0/94, ticks=0/155936, in_queue=155300, util=81.88%
</pre>

### 6.2.5 deadline随机读 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=128): err= 0: pid=18760
  read : io=125926MB, bw=1049.4MB/s, iops=67155 , runt=120008msec
    clat (usec): min=60 , max=20605 , avg=1903.45, stdev=2380.00
     lat (usec): min=60 , max=20605 , avg=1903.61, stdev=2380.01
    clat percentiles (usec):
     |  1.00th=[  620],  5.00th=[  748], 10.00th=[  812], 20.00th=[  892],
     | 30.00th=[  956], 40.00th=[ 1020], 50.00th=[ 1080], 60.00th=[ 1160],
     | 70.00th=[ 1240], 80.00th=[ 1400], 90.00th=[ 7456], 95.00th=[ 8640],
     | 99.00th=[ 9792], 99.50th=[10176], 99.90th=[11072], 99.95th=[11456],
     | 99.99th=[12224]
    bw (KB/s)  : min= 6016, max=15520, per=0.78%, avg=8398.79, stdev=662.66
    lat (usec) : 100=0.01%, 250=0.01%, 500=0.19%, 750=5.15%, 1000=31.47%
    lat (msec) : 2=51.88%, 4=0.30%, 10=10.36%, 20=0.65%, 50=0.01%
  cpu          : usr=0.25%, sys=1.59%, ctx=8843083, majf=0, minf=3719
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=8059256/w=0/d=0, short=r=0/w=0/d=0
</pre>

### 6.2.6 deadline随机写 ###

<pre name="code" class="shell">
innodb-data: (groupid=1, jobs=32): err= 0: pid=19175
  write: io=13133MB, bw=112058KB/s, iops=7003 , runt=120008msec
    clat (usec): min=47 , max=108446 , avg=4558.95, stdev=5001.98
     lat (usec): min=47 , max=108446 , avg=4559.15, stdev=5001.98
    clat percentiles (usec):
     |  1.00th=[   52],  5.00th=[   54], 10.00th=[   55], 20.00th=[   58],
     | 30.00th=[   87], 40.00th=[  207], 50.00th=[ 2704], 60.00th=[ 6944],
     | 70.00th=[ 7520], 80.00th=[ 8384], 90.00th=[11200], 95.00th=[13248],
     | 99.00th=[15552], 99.50th=[18304], 99.90th=[22912], 99.95th=[33536],
     | 99.99th=[88576]
    bw (KB/s)  : min= 1111, max=12181, per=3.13%, avg=3504.98, stdev=1363.81
    lat (usec) : 50=0.02%, 100=33.82%, 250=7.93%, 500=6.28%, 750=0.51%
    lat (usec) : 1000=0.02%
    lat (msec) : 2=1.21%, 4=0.55%, 10=36.57%, 20=12.76%, 50=0.29%
    lat (msec) : 100=0.03%, 250=0.01%
  cpu          : usr=0.31%, sys=0.67%, ctx=1679679, majf=0, minf=774
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=840492/d=0, short=r=0/w=0/d=0
</pre>

### 6.2.7 noop随机读 ###

<pre name="code" class="shell">
innodb-data: (groupid=2, jobs=128): err= 0: pid=19393
  read : io=126031MB, bw=1050.2MB/s, iops=67211 , runt=120009msec
    clat (usec): min=64 , max=124904 , avg=1901.92, stdev=2412.64
     lat (usec): min=64 , max=124904 , avg=1902.08, stdev=2412.64
    clat percentiles (usec):
     |  1.00th=[  628],  5.00th=[  748], 10.00th=[  812], 20.00th=[  892],
     | 30.00th=[  956], 40.00th=[ 1020], 50.00th=[ 1080], 60.00th=[ 1160],
     | 70.00th=[ 1256], 80.00th=[ 1400], 90.00th=[ 7328], 95.00th=[ 8640],
     | 99.00th=[ 9664], 99.50th=[10048], 99.90th=[11200], 99.95th=[11712],
     | 99.99th=[40704]
    bw (KB/s)  : min= 1600, max=11273, per=0.78%, avg=8405.59, stdev=790.87
    lat (usec) : 100=0.01%, 250=0.01%, 500=0.18%, 750=5.11%, 1000=31.57%
    lat (msec) : 2=51.73%, 4=0.38%, 10=10.45%, 20=0.56%, 50=0.02%
    lat (msec) : 100=0.01%, 250=0.01%
  cpu          : usr=0.25%, sys=1.51%, ctx=8736875, majf=0, minf=3739
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=8065959/w=0/d=0, short=r=0/w=0/d=0
</pre>

### 6.2.8 noop随机写 ###

<pre name="code" class="shell">
innodb-data: (groupid=3, jobs=32): err= 0: pid=19835
  write: io=13240MB, bw=112976KB/s, iops=7060 , runt=120007msec
    clat (usec): min=47 , max=46341 , avg=4522.20, stdev=3383.03
     lat (usec): min=47 , max=46341 , avg=4522.39, stdev=3383.02
    clat percentiles (usec):
     |  1.00th=[   53],  5.00th=[   56], 10.00th=[   67], 20.00th=[  149],
     | 30.00th=[ 2288], 40.00th=[ 4080], 50.00th=[ 5472], 60.00th=[ 5920],
     | 70.00th=[ 6368], 80.00th=[ 6880], 90.00th=[ 7712], 95.00th=[ 9280],
     | 99.00th=[14144], 99.50th=[15424], 99.90th=[21632], 99.95th=[23936],
     | 99.99th=[32640]
    bw (KB/s)  : min= 1380, max=15809, per=3.13%, avg=3533.82, stdev=1533.77
    lat (usec) : 50=0.03%, 100=16.84%, 250=5.47%, 500=3.68%, 750=0.34%
    lat (usec) : 1000=0.01%
    lat (msec) : 2=1.29%, 4=11.61%, 10=56.76%, 20=3.81%, 50=0.16%
  cpu          : usr=0.30%, sys=0.65%, ctx=1692756, majf=0, minf=782
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=847367/d=0, short=r=0/w=0/d=0
</pre>

### 6.2.9 noop libaio 4k随机读 ###

<pre name="code" class="shell">
innodb-data: (groupid=4, jobs=1): err= 0: pid=20116
  read : io=37719MB, bw=321869KB/s, iops=80467 , runt=120001msec
    slat (usec): min=3 , max=368 , avg= 7.66, stdev= 7.50
    clat (usec): min=132 , max=66961 , avg=785.93, stdev=282.30
     lat (usec): min=139 , max=66969 , avg=793.90, stdev=282.86
    clat percentiles (usec):
     |  1.00th=[  446],  5.00th=[  540], 10.00th=[  596], 20.00th=[  668],
     | 30.00th=[  708], 40.00th=[  756], 50.00th=[  796], 60.00th=[  836],
     | 70.00th=[  868], 80.00th=[  900], 90.00th=[  940], 95.00th=[  980],
     | 99.00th=[ 1064], 99.50th=[ 1112], 99.90th=[ 1432], 99.95th=[ 2640],
     | 99.99th=[ 7648]
    bw (KB/s)  : min=56920, max=354384, per=99.99%, avg=321829.72, stdev=29754.04
    lat (usec) : 250=0.01%, 500=2.65%, 750=36.33%, 1000=57.80%
    lat (msec) : 2=3.16%, 4=0.03%, 10=0.02%, 20=0.01%, 50=0.01%
    lat (msec) : 100=0.01%
  cpu          : usr=16.46%, sys=71.03%, ctx=172471, majf=0, minf=85
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued    : total=r=9656157/w=0/d=0, short=r=0/w=0/d=0
</pre>

### 6.2.10 noop libaio 4k随机写 ###
<pre name="code" class="shell">
innodb-data: (groupid=5, jobs=1): err= 0: pid=20316
  write: io=5512.8MB, bw=47027KB/s, iops=11756 , runt=120022msec
    slat (usec): min=6 , max=2797 , avg=10.12, stdev= 9.33
    clat (usec): min=658 , max=197801 , avg=5429.63, stdev=2532.01
     lat (usec): min=676 , max=197813 , avg=5440.04, stdev=2531.52
    clat percentiles (usec):
     |  1.00th=[ 2640],  5.00th=[ 3536], 10.00th=[ 3920], 20.00th=[ 4320],
     | 30.00th=[ 4704], 40.00th=[ 5024], 50.00th=[ 5280], 60.00th=[ 5472],
     | 70.00th=[ 5792], 80.00th=[ 6240], 90.00th=[ 6624], 95.00th=[ 7136],
     | 99.00th=[11200], 99.50th=[19584], 99.90th=[40704], 99.95th=[46848],
     | 99.99th=[57088]
    bw (KB/s)  : min=33712, max=111048, per=100.00%, avg=47066.41, stdev=4611.08
    lat (usec) : 750=0.01%, 1000=0.04%
    lat (msec) : 2=0.41%, 4=11.59%, 10=86.82%, 20=0.65%, 50=0.45%
    lat (msec) : 100=0.02%, 250=0.01%
  cpu          : usr=5.50%, sys=14.51%, ctx=99258, majf=0, minf=20
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued    : total=r=0/w=1411074/d=0, short=r=0/w=0/d=0
</pre>

## 6.3 SAS RAID5 上虚拟机的测试 ##

在146上的虚拟机，型号是2个CPU，4GB内存

### 6.3.1 CFQ随机读 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=128): err= 0: pid=17176
  read : io=8023.4MB, bw=68406KB/s, iops=4275 , runt=120105msec
    clat (usec): min=168 , max=4424.7K, avg=29913.12, stdev=48315.98
     lat (usec): min=168 , max=4424.7K, avg=29913.52, stdev=48315.98
    clat percentiles (msec):
     |  1.00th=[   15],  5.00th=[   16], 10.00th=[   17], 20.00th=[   19],
     | 30.00th=[   20], 40.00th=[   21], 50.00th=[   23], 60.00th=[   26],
     | 70.00th=[   30], 80.00th=[   37], 90.00th=[   49], 95.00th=[   64],
     | 99.00th=[  103], 99.50th=[  126], 99.90th=[  285], 99.95th=[  461],
     | 99.99th=[ 2933]
    bw (KB/s)  : min=    4, max= 1330, per=0.80%, avg=546.41, stdev=97.04
    lat (usec) : 250=0.01%, 500=0.01%, 750=0.01%
    lat (msec) : 2=0.02%, 4=0.11%, 10=0.40%, 20=35.15%, 50=54.88%
    lat (msec) : 100=8.30%, 250=1.01%, 500=0.07%, 750=0.01%, 1000=0.01%
    lat (msec) : 2000=0.02%, >=2000=0.02%
  cpu          : usr=0.03%, sys=0.15%, ctx=513730, majf=0, minf=3525
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=513495/w=0/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
   READ: io=8023.4MB, aggrb=68406KB/s, minb=68406KB/s, maxb=68406KB/s, mint=120105msec, maxt=120105msec

Disk stats (read/write):
  vdb: ios=513464/2, merge=0/1, ticks=15325456/24, in_queue=15414824, util=99.98%
</pre>

### 6.3.2 CFQ随机写 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=32): err= 0: pid=17323
  write: io=2021.2MB, bw=17244KB/s, iops=1077 , runt=120019msec
    clat (usec): min=149 , max=2262.2K, avg=29678.70, stdev=181597.64
     lat (usec): min=150 , max=2262.2K, avg=29679.15, stdev=181597.65
    clat percentiles (usec):
     |  1.00th=[  173],  5.00th=[  209], 10.00th=[  233], 20.00th=[  270],
     | 30.00th=[  294], 40.00th=[  318], 50.00th=[  338], 60.00th=[  362],
     | 70.00th=[  390], 80.00th=[  426], 90.00th=[  482], 95.00th=[  564],
     | 99.00th=[1155072], 99.50th=[1384448], 99.90th=[1679360], 99.95th=[1728512],
     | 99.99th=[1826816]
    bw (KB/s)  : min=    9, max=46880, per=3.37%, avg=581.10, stdev=1012.23
    lat (usec) : 250=14.39%, 500=77.18%, 750=4.59%, 1000=0.16%
    lat (msec) : 2=0.53%, 4=0.01%, 10=0.03%, 20=0.03%, 50=0.09%
    lat (msec) : 100=0.11%, 250=0.14%, 500=0.17%, 750=0.25%, 1000=0.74%
    lat (msec) : 2000=1.57%, >=2000=0.01%
  cpu          : usr=0.05%, sys=0.22%, ctx=262097, majf=0, minf=686
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=129353/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=2021.2MB, aggrb=17244KB/s, minb=17244KB/s, maxb=17244KB/s, mint=120019msec, maxt=120019msec

Disk stats (read/write):
  vdb: ios=0/129398, merge=0/23, ticks=0/111788, in_queue=111596, util=92.73%
</pre>

## 6.4 SSD RAID5 上虚拟机的测试 ##

在146上的虚拟机，型号是4个CPU，8GB内存

### 6.4.1 CFQ随机读 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=128): err= 0: pid=7242
  read : io=48849MB, bw=416781KB/s, iops=26048 , runt=120019msec
    clat (usec): min=140 , max=486741 , avg=4904.41, stdev=11260.51
     lat (usec): min=140 , max=486742 , avg=4905.21, stdev=11260.71
    clat percentiles (usec):
     |  1.00th=[  294],  5.00th=[  506], 10.00th=[  604], 20.00th=[  684],
     | 30.00th=[  740], 40.00th=[  788], 50.00th=[  844], 60.00th=[  916],
     | 70.00th=[ 1032], 80.00th=[ 1336], 90.00th=[27776], 95.00th=[32640],
     | 99.00th=[42240], 99.50th=[48896], 99.90th=[74240], 99.95th=[89600],
     | 99.99th=[148480]
    bw (KB/s)  : min=   52, max=10816, per=0.78%, avg=3260.73, stdev=756.35
    lat (usec) : 250=0.35%, 500=4.47%, 750=27.55%, 1000=35.00%
    lat (msec) : 2=17.17%, 4=2.15%, 10=1.00%, 20=0.24%, 50=11.63%
    lat (msec) : 100=0.42%, 250=0.03%, 500=0.01%
  cpu          : usr=0.19%, sys=2.16%, ctx=4239514, majf=1, minf=3374
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=3126350/w=0/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
   READ: io=48849MB, aggrb=416780KB/s, minb=416780KB/s, maxb=416780KB/s, mint=120019msec, maxt=120019msec

Disk stats (read/write):
  vdb: ios=3122143/3, merge=24362/1, ticks=2143164/4192, in_queue=2133152, util=99.02%
</pre>

### 6.4.2 CFQ随机写 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=32): err= 0: pid=7528
  write: io=5688.2MB, bw=48539KB/s, iops=3033 , runt=120018msec
    clat (usec): min=146 , max=2238.5K, avg=10537.49, stdev=82186.77
     lat (usec): min=147 , max=2238.5K, avg=10537.93, stdev=82186.77
    clat percentiles (usec):
     |  1.00th=[  169],  5.00th=[  191], 10.00th=[  199], 20.00th=[  219],
     | 30.00th=[  247], 40.00th=[  274], 50.00th=[  294], 60.00th=[  314],
     | 70.00th=[  338], 80.00th=[  370], 90.00th=[  446], 95.00th=[  588],
     | 99.00th=[536576], 99.50th=[602112], 99.90th=[921600], 99.95th=[1138688],
     | 99.99th=[1499136]
    bw (KB/s)  : min=    7, max=47040, per=3.24%, avg=1570.52, stdev=2041.31
    lat (usec) : 250=31.21%, 500=61.62%, 750=3.04%, 1000=0.32%
    lat (msec) : 2=2.11%, 4=0.01%, 10=0.01%, 20=0.01%, 50=0.01%
    lat (msec) : 100=0.01%, 250=0.01%, 500=0.37%, 750=1.06%, 1000=0.16%
    lat (msec) : 2000=0.07%, >=2000=0.01%
  cpu          : usr=0.14%, sys=0.57%, ctx=734000, majf=0, minf=686
  IO depths    : 1=100.0%, 2=0.0%, 4=0.0%, 8=0.0%, 16=0.0%, 32=0.0%, >=64=0.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     issued    : total=r=0/w=364095/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=5688.2MB, aggrb=48538KB/s, minb=48538KB/s, maxb=48538KB/s, mint=120018msec, maxt=120018msec

Disk stats (read/write):
  vdb: ios=0/364164, merge=0/2826, ticks=0/95912, in_queue=95376, util=79.47%
</pre>

### 6.4.3 noop libaio 4k随机读 ###

<pre name="code" class="shell">
innodb-data: (groupid=0, jobs=1): err= 0: pid=7794
  read : io=19465MB, bw=166097KB/s, iops=41524 , runt=120001msec
    slat (usec): min=3 , max=4199 , avg=18.33, stdev=13.67
    clat (usec): min=133 , max=4777.5K, avg=1518.98, stdev=2259.09
     lat (usec): min=155 , max=4777.5K, avg=1538.15, stdev=2259.63
    clat percentiles (usec):
     |  1.00th=[  692],  5.00th=[  964], 10.00th=[ 1176], 20.00th=[ 1288],
     | 30.00th=[ 1368], 40.00th=[ 1432], 50.00th=[ 1480], 60.00th=[ 1528],
     | 70.00th=[ 1624], 80.00th=[ 1784], 90.00th=[ 1896], 95.00th=[ 2008],
     | 99.00th=[ 2352], 99.50th=[ 2480], 99.90th=[ 5408], 99.95th=[ 8096],
     | 99.99th=[13248]
    bw (KB/s)  : min=78416, max=232392, per=99.98%, avg=166068.02, stdev=22821.31
    lat (usec) : 250=0.01%, 500=0.05%, 750=1.67%, 1000=3.96%
    lat (msec) : 2=89.18%, 4=5.01%, 10=0.11%, 20=0.02%, 50=0.01%
    lat (msec) : 100=0.01%, 250=0.01%, >=2000=0.01%
  cpu          : usr=18.08%, sys=80.20%, ctx=6881, majf=0, minf=84
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued    : total=r=4982956/w=0/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
   READ: io=19465MB, aggrb=166097KB/s, minb=166097KB/s, maxb=166097KB/s, mint=120001msec, maxt=120001msec

Disk stats (read/write):
  vdb: ios=4973527/3, merge=0/1, ticks=1629596/4, in_queue=1625912, util=99.93%
</pre>

### 6.4.4 noop libaio 4k随机写 ###

<pre name="code" class="shell">
innodb-data: (g=0): rw=randwrite, bs=4K-4K/4K-4K, ioengine=libaio, iodepth=64
fio 2.0.7
Starting 1 process
Jobs: 1 (f=0): [w] [30.9% done] [0K/27811K /s] [0 /6790  iops] [eta 04m:30s] 
innodb-data: (groupid=0, jobs=1): err= 0: pid=7801
  write: io=3161.1MB, bw=26980KB/s, iops=6744 , runt=120009msec
    slat (usec): min=6 , max=918 , avg=19.99, stdev=10.09
    clat (usec): min=40 , max=447856 , avg=9462.51, stdev=7419.90
     lat (usec): min=106 , max=447867 , avg=9483.31, stdev=7419.14
    clat percentiles (usec):
     |  1.00th=[  124],  5.00th=[  161], 10.00th=[  199], 20.00th=[ 5024],
     | 30.00th=[ 9536], 40.00th=[10176], 50.00th=[11072], 60.00th=[11968],
     | 70.00th=[12480], 80.00th=[12864], 90.00th=[13376], 95.00th=[13760],
     | 99.00th=[14528], 99.50th=[15040], 99.90th=[28288], 99.95th=[47872],
     | 99.99th=[432128]
    bw (KB/s)  : min=  598, max=50496, per=100.00%, avg=26991.69, stdev=3190.70
    lat (usec) : 50=0.01%, 100=0.07%, 250=14.65%, 500=4.41%, 750=0.10%
    lat (usec) : 1000=0.01%
    lat (msec) : 2=0.50%, 4=0.02%, 10=17.03%, 20=63.00%, 50=0.16%
    lat (msec) : 100=0.02%, 250=0.01%, 500=0.02%
  cpu          : usr=5.90%, sys=22.44%, ctx=433454, majf=0, minf=18
  IO depths    : 1=0.1%, 2=0.1%, 4=0.1%, 8=0.1%, 16=0.1%, 32=0.1%, >=64=100.0%
     submit    : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.0%, >=64=0.0%
     complete  : 0=0.0%, 4=100.0%, 8=0.0%, 16=0.0%, 32=0.0%, 64=0.1%, >=64=0.0%
     issued    : total=r=0/w=809447/d=0, short=r=0/w=0/d=0

Run status group 0 (all jobs):
  WRITE: io=3161.1MB, aggrb=26979KB/s, minb=26979KB/s, maxb=26979KB/s, mint=120009msec, maxt=120009msec

Disk stats (read/write):
  vdb: ios=0/807760, merge=0/20, ticks=0/7612468, in_queue=7612684, util=100.00%
</pre>

# 7 结论分析 #
## 7.1 测试结果对比 ##
 

![](http://way4ever.com/wp-content/uploads/2013/04/sas_ssd.png)

**是在SWS平台上的测试结果**

## 7.2 结果分析 ##

通过对比"SSD RAID5 IOPS"和"其他SSD RAID5 IOPS(7块SSD硬盘组成RAID5，RAID卡是LSI MegaRAID 9280-24i4e，地址是 http://marcitland.blogspot.com/2011/06/scst-ssd-arrays.html 的测试结果，IOPS都在一个数量级上，延迟时间也在同一个数量级上，这说明我们的SSD性能正常。但是"SSD RAID5 IOPS"在《libaio 4k随机写》测试中是11,756,而"其他SSD RAID5 IOPS"是19,407。原因如下：

* 我们的SSD硬盘型号是INTEL SSDSA2CW160G3，大小是160GB，接口是SATAII，接口速率是3Gb/s。而对方的SSD硬盘型号是CTFDDAC256MAG，大小是256GB，接口是SATAIII，接口速率是6Gb/s。我们的SSD硬盘性能比对方的低。
* 我们使用的是服务器上集成的RAID卡，型号是Dell PowerEdge RAID Controller H700 Integrated。对方使用的是LSI MegaRAID 9280-24i4e,对方使用的RAID卡比我们贵几百美元，我们的RAID卡性能不如对方RAID卡。RAID5的写操作对RAID卡的IO处理器要求很高。假如使用RAID10，性能应该没区别。
* 我们是在ext4文件系统上测试，而对方是在裸volume上测试。

"SSD RAID5 虚拟机 IOPS"的性能大概是"SSD RAID5 IOPS"性能的一半。而且当测试虚拟机IOPS时，宿主机的硬盘的util已经达到了100%，虚拟机硬盘的util也达到100%,但是性能却是一半，这个问题需要研究。**#TODO**

"SAS RAID5 虚拟机 IOPS"的性能和"SAS RAID5 IOPS"性能相当。


## 7.3 SSD的寿命问题 ##

计算SSD的寿命可以看这篇文章 http://ssd.zol.com.cn/283/2838309.html

假设SSD的寿命是3000次P/E(一次P/E相当与写满一次SSD硬盘)，假设我们每天写满一次SSD硬盘(几乎不可能)，则SSD硬盘的寿命是8年左右。但是估计不到2年，这块SSD硬盘就会被淘汰了(被更大容量，更高速度的SSD硬盘取代)。
