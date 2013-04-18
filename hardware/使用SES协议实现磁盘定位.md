
## SES介绍

通常来说，扩展柜管理是通过SES 协议/IPMI协议实现的。至于SES 协议和IPMI协议本身，一哥在这里不介绍了，baidu/google一下就可以下载。这里重点讲讲SES 的实现和对RAID system的可能的影响。

SES 协议决定了SES device必须是一个SCSI device，这是因为SES协议是通过SCSI Command "Send Diagnostics" 和 "receive diagnostics"传递协议中的page。而SES Device一般来说是实现SES 协议的chip，而这个chip对外呈现一个SES Device (SCSI device type value = 0Dh), RAID Head 把它连到Drive Channel 中，通过SCSI Inband方式进行管理。如果一个扩展柜是Standalone的，那么就需要Host software来管理了。

IPMI 协议最初是用在PC/Server上的，由于它协议的可扩展性加上是大厂推动的协议，很快也应用在存储产品上。在管理途径上和SES的区别在于IPMI是通过outband来管理的。

从过去到现在，SES的实现有以下方法

* Disk 代理：已经过时，很少有人用了。
* 专门的SES Chip. 现在也渐渐少了,因为其功能单一。
* 在loop switch/SAS expander chip 上，SES 功能作为附带功能出现，同时这些chip 能提供一个SES device. 在loop switch 和 SAS Expander 普及的现在，这种方式最流行。

SES 的硬件方面: 一般来说，实现ses的chip通常在硬件上有I2C或是Serial 等连接，因为这些方式是沟通FAN/PSU 以及其他特殊硬件component的方式。ses chip 通过I2C和Serial Interface 和PSU/FAN的application code交换传递控制和状态数据。

SES 的软件方面：RAID 主要通过SES 协议上的一些status pages 和control page 来监控扩展柜，而这些pages是通过上面提到的SCSI命令实现的。我们所说的控制PSU/FAN等所谓的一些绿色存储的功能，大多就是通过动态设置 FAN/PSU的threshold等方法来实现的，决不是什么高深的学问。


## 使用SES协议实现磁盘定位

使用 SES 协议实现磁盘定位功能。给 Expander 发送SES 命令，就能控制槽位上 LED 灯的亮灭。

**首先获取 JBOD上所有槽位上的信息 ( 二进制信息) 。**

> [root@localhost md_test]# sg_ses -p 0x2 /dev/sg16 -r


        00 00 00 00 10 00 00 00  05 00 00 00 01 00 00 00
        01 00 00 00 01 00 00 00  01 00 00 00 01 00 00 00
        01 00 00 00 01 00 00 00  01 00 00 00 01 00 00 00
        01 00 00 00 01 00 00 00  01 00 00 00 01 00 00 00
        01 00 00 00 01 00 00 00   00 00 00 00 01 00 00 76
        01 00 01 f5


* “/dev/sg16” 是 Expander的 sg 标识，Expander 是 ”enclosure service device”。 ”0x2 page”是 Enclosure status diagnostic page。
* 0~7 字节代表 Enclosure的状态；
* 8~11 字节代表第 1个槽位上的信息；
* 12~15 字节代表第 2个槽位上的信息；
* 68~71 字节代表第 16个槽位上的信息；
* 用 4个字节 (32 位) 标识一个槽位上的信息。第 1 个字节代表当前槽位上设备的状态， ”05” 表示没有安装设备， ”01”表示已经安装了设备。假如第 3 个字节为”00” ，则代表这个槽位的 LED 灯亮，假如为 ”02”，则代表这槽位的 LED 灯灭。比如”01 00 02 00”代表这个槽位上有设备，并且 LED 灯亮。更多状态信息请查阅 SES-3 标准文档。


**可以给 Expander发送控制信息，控制槽位上的 LED 灯亮灭。**

> [root@localhost md_test]# cat page.out

        00 00 00 00 10 00 00 00  08 00 02 00 08 00 00 00
        01 00 00 00 01 00 00 00  01 00 00 00 01 00 00 00
        01 00 00 00 01 00 00 00  01 00 00 00 01 00 00 00
        01 00 00 00 01 00 00 00  01 00 00 00 01 00 00 00
        01 00 00 00 08 00 02 00   00 00 00 00 01 00 00 76
        01 00 01 f5
        
> [root@localhost md_test]# cat page.out | sg_ses -c -p 0x2 -d -  /dev/sg16
 

* 在第 1个槽位的 4 个控制字节中，第 1个字节 ”08” 表示这4 个字节是控制命令，第 3 个字节”02” 表示让 LED灯亮。
* 在第 2个槽位的 4 个控制字节中，第 1个字节 ”08” 表示这4 个字节是控制命令，第 3 个字节”00” 表示让 LED灯灭。
* 在第 16个槽位的 4 个控制字节中，第 1个字节 ”08” 表示这4 个字节是控制命令，第 3 个字节”02” 表示让 LED灯亮。

