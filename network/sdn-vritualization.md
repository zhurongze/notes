
# Software Defined Networking and Network Virtualization

资料来源：[Software_Defined_Networking_and_Network_Virtualization](http://wikibon.org/wiki/v/Networking_Revolution:_Software_Defined_Networking_and_Network_Virtualization)
    
# 1 网络产业的变化

* IT人员需要自动化的基础架构，以便简化运维。
* 新技术已经出现，客户是否有强力的愿望去替换旧技术。
    

# 2 传统的网络架构


![](http://way4ever.com/wp-content/uploads/2013/04/sdn1.png)

有data plane和 control plane。

* control plane负责配置route table和path，保证data flows。
* control plance把path下发到data plane并存在硬件中。
    
但是会遇到两种问题：

* QoS：你需要配置多个交换机，而且配置是一样的。
* 不同的traffic types需要不同的path，以便满足要求。比如iSCSI需要高带宽，voice需要低延迟。



# 3 SDN的三个要素

1. Ability to manage the forwarding of frames/packets and apply policy;
2. Ability to perform this at scale in a dynamic fashion;
3. Ability to be programmed. 

SDN架构可以在大规模网络下通过可编程的方式操作数据包流。SDN架构提供network-wide的视图和管理network and network flows的能力。

架构的原理是分离control plane 和 data  plane，并给 control plane 提供编程接口。data plane 设备从 control plane 接收 forwarding rules ，并应用这些rules。

![](http://way4ever.com/wp-content/uploads/2013/04/sdn2.png)

control plane具有整个网络的视图，这是一个可编程的平台，它可以基于收集到的信息(实时流量等)，做出关于routing、securtiy、quality的决定。
    
这还允许同时管理虚拟资源和物理资源。通过Virtual Ethernet Bridge (VEB) in the hypervisor 进行虚拟网络的整合。

![](http://way4ever.com/wp-content/uploads/2013/04/sdn3.png)

通过SDN的集中管理和编程接口，可以实现许多高级功能，比如 traffic optimization, security, outage, or maintenance、Separate traffic types、network changes等。

# 4 网络虚拟化

Network Virtualization和SDN不一样。

Nicira使用软件对网络进行虚拟化，跟VMware的ESX一样。不需要购买新的硬件。这就是为什么VMware收购的原因之一。
因为VMware VMotion和 Microsoft Live Migration的需要，Nicira很适合IaaS的环境。


![](http://way4ever.com/wp-content/uploads/2013/04/nicira.png)

# 5 个人总结

以前的虚拟化平台只是对计算资源进行池化，IaaS平台需要对存储、网络资源进行池化。

VMware的一盘大旗 = ESX + Nicira + Virsto，分别对应着计算虚拟化、网络虚拟化、存储虚拟化。

根据SDN的定义，可以用于定义SDS(Software define Storage)上:

1. 把不同存储阵列上的基础存储资源进行虚拟化，融入资源池中。
2. 提供API，便于二次开发或与其他系统整合。

OpenStack的Cinder项目离SDS很近。
