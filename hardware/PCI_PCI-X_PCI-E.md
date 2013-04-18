
# PCI  PCI-X  PCI-E之间的比较

## PCI
PCI总线的主要性能

* 支持10台外设
* 总线时钟频率33.3MHz/66MHz
* 与CPU及时钟频率无关
* 总线位宽32bit（5V）或64bit（5V）
 
 
<table>
    <tr>
        <th>类型</th>
        <th>位宽</th>
        <th>总线时钟频率</th>
        <th>带宽速度</th>
    </tr>
    <tr>
        <td>PCI</td>
        <td>32bit</td>
        <td>33MHz</td>
        <td>133MB/s</td>
    </tr>
    <tr>
        <td>PCI</td>
        <td>64bit</td>
        <td>33MHz</td>
        <td>266MB/s</td>
    </tr>
</table>

64bit PCI可分为33MHz、66MHz、133MHz、266MHz、533MHz等不同的频率规范。其中运行与33MHz 64bit PCI接口速率为266MB/s，并可下兼容于32bit PCI接口。

***

## PCI-X

66MHz以上的64bit PCI全面改称为PCI-X，其接口方式与32bit PCI完全不同，但无论在运行频率还是在传输率上都较PCI有很大提升。最新的PCI-X已经发展到了2.0规范，最高的PCI-X 2.0接口可提供1066 MHz的工作频率，传输速度也达到8.6 GB/s(1066MHzX64bit/8)，是32bit PCI的66倍。

<table>
    <tr>
        <th>类型</th>
        <th>位宽</th>
        <th>总线时钟频率</th>
        <th>带宽速度</th>
    </tr>
    <tr>
        <td>PCI-X 1.0</td>
        <td>64bit</td>
        <td>66 MHz</td>
        <td>528 MB/s</td>
    </tr>
    <tr>
        <td>PCI-X 1.0</td>
        <td>64bit</td>
        <td>133 MHz</td>
        <td>1064 MB/s</td>
    </tr>
    <tr>
        <td>PCI-X 2.0</td>
        <td>64bit</td>
        <td>266 MHz</td>
        <td>2128 MB/s</td>
    </tr>
    <tr>
        <td>PCI-X 2.0</td>
        <td>64bit</td>
        <td>533 MHz</td>
        <td>4264 MB/s</td>
    </tr>
    <tr>
        <td>PCI-X 3.0</td>
        <td>64bit</td>
        <td>1066 MHz</td>
        <td>8528 MB/s</td>
    </tr>
</table>

### PCI/PCI-X 的性能与需求

![](http://way4ever.com/wp-content/uploads/2013/04/p1.png)


### PCI-X的限制

![](http://way4ever.com/wp-content/uploads/2013/04/p2.png)
![](http://way4ever.com/wp-content/uploads/2013/04/p3.png)
![](http://way4ever.com/wp-content/uploads/2013/04/p4.jpg)





 
### PCI-X的未来

![](http://way4ever.com/wp-content/uploads/2013/04/p5.png)
![](http://way4ever.com/wp-content/uploads/2013/04/p6.png)
 
***
 
## PCI-E 的介绍
 
PCI Express，简称PCI-E，基于更快的串行通信系统。PCIe仅应用于内部互连。由于PCIe是基于现有的PCI系统，只需修改物理层而无须修改软件就可将现有PCI系统转换为PCIe。PCIe拥有更快的速率，以取代几乎全部现有的内部总线（包括AGP和PCI）。英特尔希望将来能用一个PCIe控制器和所有外部设备交流，取代现有的南桥／北桥方案。

![](http://way4ever.com/wp-content/uploads/2013/04/p7.png)

物理层：PCI Express中一个通道基本连接构成是两对低压差动信号：一对用于接收，另一对用于发送（可以实现双向）。时钟信号用8bit/10bit编码方案嵌入数据以达到高速数据率。PCI Express物理层支持1x(单通道)、2x(双通道)、4x、8x、12x、16x、32x（32通道）。

![](http://way4ever.com/wp-content/uploads/2013/04/p8.jpg)

数据链路层：数据链路层的主要作用是确保数据包在物理层的可靠传送。它确保数据的完好无损和对来自交易层的交易包添加序列号和CRC。

交易层：交易层接收来自软件或应用层的读写请求，然后形成交易请求包并传给数据链路层。所有交易请求将被分割，有些请求需要应答。交易层也接收来自数据链路层的应答包并与来自软件层的原始请求进行对比。每个包都有一个独一无二的标号，这样应答包可被传给正确的请求者。
 
**PCI Express的拓扑结构** 
 
![](http://way4ever.com/wp-content/uploads/2013/04/p9.png)
 


 
### PCI Express 数据带宽

由于PCI Express通道是双单工通道，因此它可以在接收一个方向信号的同时，接收来自另一方向的信号。这种双向性可以实现双倍的整体有效带宽或吞吐量。x1通道的最高带宽大约是2.5 Gbps （2.5 GHz时钟频率） ，但由于带宽是双向的，因此双向有效数据速率就是5Gbps。这
些速率是指以8b/10b编码发送剥离数据字节的速率，2.5Gbps即编码速率。带宽也可能是指一个未编码或有效数据速率，此类速率是编码速率的80% （例如，x1通道的2 Gbps单向未编码速率和4 Gbps双向未编码速率） 。在所有规格的PCI Express通道中，都可通过插卡插槽实现
x1、x4、x8或x16通道。因此，设计师们可以通过增加PCIExpress插卡和计算机插槽的通道，来扩充PCI Express的串行总线I/O频率。例如，x1通道可为英特尔®PRO/1000 PT服务器网卡的PCI ExpressI/O提供充足的总线带宽。但是，要为双端口千兆位以太网网卡提供额外的I/O带宽，就需要在英特尔®PRO/1000 PT双端口服务器网卡中使用x4通道。对网络设计师来说，在选择服务器和服务器网卡时，PCIExpress通道规格尤为重要。如上所述，多端口网卡具备额外通道，能够支持多千兆位以太网端口额外的流量和带宽需求。但是，选择服务器时，需注意PCI Express插槽的通道数量：PCI Express插槽的通道数量必须不少于网卡的最大通道数。例如，x1网卡可在任何PCIe插槽中使用，但x4网卡则只能在x4或以上的插槽中使用。

![](http://way4ever.com/wp-content/uploads/2013/04/p10.jpg)

![](http://way4ever.com/wp-content/uploads/2013/04/p11.jpg)
 

x4 PCI Express服务器插槽的意义在于能够提供更高的I/O带宽，从而支持多端口网卡性能。当不需要多端口连接时，x4插槽可兼容单端口x1网卡。x4性能可在未来使用双端口网卡升级网络时应用。

![](http://way4ever.com/wp-content/uploads/2013/04/p14.png)

### PCI与PCI-E的对比
 
PCI是共享带宽

![](http://way4ever.com/wp-content/uploads/2013/04/p12.jpg)
 

PCI-E是采用端到端总线方式，使用包交换，而不是共享总线。PCIe的连接是建立在一个双向的序列的（1-bit）点对点连接基础之上，这称之为“传输通道”。与PCI 连接形成鲜明对比的是PCI是基于总线控制，所有设备共同分享的单向32位并行总线。PCIe是一个多层协议，由一个对话层，一个数据交换层和一个物理层构成。物理层又可进一步分为逻辑子层和电气子层。逻辑子层又可分为物理代码子层（PCS）和介质接入控制子层（MAC）。

![](http://way4ever.com/wp-content/uploads/2013/04/p13.jpg)
