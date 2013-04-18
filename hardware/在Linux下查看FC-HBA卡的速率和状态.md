
 
## 查看FC HBA卡的WWN号

一般是在/sys/class/fc_host/host*/目录下

<pre name="code" class="shell"> 
[root@localhost ~]#cat  /sys/class/fc_host/host2/port_name
0x2100001b329240d7
[root@localhost ~]#cat  /sys/class/fc_host/host2/node_name
0x2100001b329240d7
[root@localhost ~]#cat  /sys/class/fc_host/host2/fabric_name
0x2100001b329240d7
</pre>
 
## 查看当前port的状态

<pre name="code" class="shell"> 
[root@localhost ~]#cat  /sys/class/fc_host/host2/port_state
Online
</pre>
 
## 查看PORT的端口ID

<pre name="code" class="shell"> 
[root@localhost ~]#cat  /sys/class/fc_host/host2/port_id
0x000001
</pre>
 
## 查看port支持的速率

<pre name="code" class="shell"> 
[root@localhost ~]#cat  /sys/class/fc_host/host2/supported_speeds
1 Gbit, 2 Gbit, 4 Gbit 
[root@localhost ~]#cat  /sys/class/fc_host/host2/supported_classes
Class 3
</pre>

 
## 在FC HBA没有插上光纤时

<pre name="code" class="shell"> 
[root@localhost ~]#cat  /sys/class/fc_host/host2/speed
unknow
[root@localhost ~]#cat  /sys/class/fc_host/host2/port_type
unknow
</pre>
 
## 给FC HBA卡插上光纤线，和其他HBA卡相连时。

<pre name="code" class="shell"> 
[root@localhost ~]#cat  /sys/class/fc_host/host2/speed
4 Gbit 
[root@localhost ~]#cat  /sys/class/fc_host/host2/port_type
LPort (private loop) 
</pre>
 
## 和光纤交换机相连时

<pre name="code" class="shell"> 
[root@localhost ~]#cat  /sys/class/fc_host/host2/port_type
NPort (fabric via point-to-point)
</pre>

