
调查于 2013-04-30 不涉及具体技术

# 1. ceph

# 1.1 简介

Ceph is a unified,distributed storage system designed for excellent performance, reliability and scalability.

ceph的底层是RADOS(Reliable Autonomic Distributed Object Store)，可以通过客户端（Native API）直接访问到RADOS的object-baseed storage system。RBD、RADOS Gateway、File System都是基于RADOS的。

**LIBRADOS（Native API）:**
        
* 读写操作
* 快照
* append、tuncate、clone range
* object level key-value mappings
        
**RADOS Gateway(REST API)：**

* 兼容S3和Swift接口
            
**RBD(Block Storage ):**
        
* Thinly provisioned
* Resizable images
* Image import/export
* Image copy or rename
* Read-only snapshots
* Revert to snapshots
* Ability to mount with Linux or QEMU KVM clients!
    
**Ceph File System：**
        
* It provides stronger data safety for mission-critical applications.
* It provides virtually unlimited storage to file systems.
* Applications that use file systems can use Ceph FS with POSIX semantics. No integration or customization required!
* Ceph automatically balances the file system to deliver maximum performance.

# 1.2 社区

最新稳定版本是 0.56 bobtail 下一个稳定版本是 0.61 Cuttlefish(May) 下下个稳定版本是  0.67 Dumpling (August)

**主要贡献者(有很多是inktank公司的):**
        
* Sage Weil：不解释 
* Yehuda Sadeh: https://github.com/yehudasa  
* TOmmi Virtanen: https://github.com/tv42  
* Greg Farnum:https://github.com/gregsfortytwo
* Josh Durgin:https://github.com/jdurgin
* Samuel Just: https://github.com/athanatos
* Wido den Hollander: https://github.com/wido

**项目统计:** http://www.ohloh.net/p/cephfs

**社区设施：**

* 有邮件列表和IRC
* 有 Submit Issues：http://tracker.ceph.com/projects/ceph (redmine工具)
* 有 FIX ISSUES： GitHub pull request.
* 安装部署：http://ceph.com/docs/master/install/
* 帮助文档：非常详细 http://ceph.com/docs/
* 路线图： http://tracker.ceph.com/projects/ceph/roadmap
* build状态：http://ceph.com/gitbuilder.cgi
* wiki: http://wiki.ceph.com/
* 有自己的design summit

**主要子项目：**

* devops(自动化部署和运维)
* fs(POSIX接口)
* linux kernel client
* performance(主要是用于测试ceph的性能)
* rbd(Block Storage)
* rgw（RADOS Gateway）
* teuthology（ceph test suite）

**版本发布策略:**

* Point Releases — ad hoc
* Interim Releases — every 2 weeks (每两周发布一个开发版)
* Stable Releases — every 3 months (每三个月发布一个稳定版)
* LTS Releases — coming soon!



# 1.3 影响力

**对其他系统的支持**

* OpenStack
* CloudStack
* OpenNebula

**Ceph的使用调查:**

在ceph的邮件列表中做的调查 http://ceph.com/community/results-from-the-ceph-census/


在81份收到的调查表中(不包括Dreamhost)

* 有36个公司在Assesment/Inverstigation
* 有24个公司在Pre-Producation
* 有21个公司在Production

在production中，已经使用的raw storage有1154 TB。Dreamhost已经超过 3PB了。
在Pre-production中，一共有2466TB。(不包括一个特别的样本，它有20PB)

平均每个集群的大小是72TB。不包括一个特别的样本，它有20PB，用于部署OpenStack。另外一个特别的样本是1PB，在pre-production  CRNET SA。

**在ODS 2013中的一个 Session：roadmap-for-ceph-integration-with-openstack**

他们打算全面集成ceph到openstack上，让nova、cinder、glance、ceilometer、keystone都用上ceph。

提高ceph和cinder、swift、ceilometer、glance的集成度。

邀请开发者：
    
1. 给openstack core projects集成ceph
2. 开发ceph的核心组件(RADOS、RGW、RBD)

**在ODS 2013中的一个Topic：Wicked Easy Ceph Block Storage & OpenStack Deployment with Crowbar**

**在ODS 2013中的一个Session:new-features-for-ceph-with-cinder-and-beyond**

what's new in Bobtail:

    Improved OSD　threading
    Filesystem and journal related-locks are now more fine-grained
    Boosted single disk IOPS from 6K to 22K
    Restructure how map updates are handled, letting each placement group process them independently
    
    Recovery QoS
    Message priority system reworked to prevent starvation
    Recovery operations can be lower priority than client I/O without starving
    Requests to access an object can increase recovery priority for that object
    
    Keystone Integration
    RADOS gateway can talk to keystone to authenticate swift api requests
    Let keystone manage your users
    Supported by the Ceph juju charm


What's next: Cuttlefish

    Incremental backup for block devices
    On-disk encryption
    REST management API for RADOS gateway
    More performance improvements(especially for small I/O)
    More!! http://www.inktank.com/about-inktank/roadmap/    
    
What's next: Dumpling

    Geo-replication for RADOS gateway
    REST management API for Ceph cluster
    ...


**Ceph的第一次Design Summit：** http://ceph.com/events/ceph-developer-summit/



# 1.4 其他

**Inktank介绍**

公司的主要工作是开发ceph，并和其他系统进行整合。公司拿到Canonical的1M投资。

公司提供各种服务：

* 咨询服务：education, evaluation, implementation and system optimization of Ceph infrastructure.
* Support subscriptions: Pre-production subscription、production subscription（提供四种级别的服务）
* 培训服务

合作伙伴有：
        
    hastexo、Dell、Mirantis、Alcatel-Lucent、SUSE、eNovance、Critix、Canonical

客户有：
    
    DreamHost Bloomberg  AT&T ebay intel filoo netelligent HUAWEI Unity zatta

管理层：

* Seagewei(Founder & CTO):Ceph的作者。也是DreamHost的co-founder，年轻的时候创建WebRing,并卖掉。
* Bryan Bogensberger(CEO)：他创建过SaaS公司Marketingisland, 创建过PaaS公司Reasonably Smart。他还在IaaS公司Joyent、DreamHost做过高级行政职务。
* Simon Anderson(Chairman):他是DreamHost的CEO，Ceph的主要赞助者，给Inktank提供了初始资金和战略意见。他已经参与过15个Startup。
* Mark Kampe（Vice President, Engineering）：40年的操作系统软件的开发经验，在小公司待过，也在大公司(IBM，SUN，HDS)待过。在Inktank，他负责建立团队、方法论、流程，以便把
Ceph变成企业级存储平台。
* Wolfgang Schulze（Vice President, Professional Services）：他负责Inktank的全球的专业服务团队，包括support,consulting,education.他有20年的IT经验。
* Debbie Moynihan（Vice President, Marketing）：她负责Inktank的全球的市场战略和执行。她有15年的B2B市场经验。
* Ross Turk（Vice President, Community）：他负责和用户、贡献者、开源社区建立关系。他也有15年的IT经验。他以前在SourceForge工作。
* Neil Levine（Vice President, Product Managemen）：他以前是Canonical的GM/VP。
* Nigel Thomas(Vice President, Business Development):负责销售。
* Artur Elizarov（Vice President, People Strategy and Administration）：负责人力资源。

# 2. sheepdog

# 2.1 简介

Sheepdog is a distributed storage system for QEMU. It provides highly available block level storage volumes that can be attached to QEMU-based virtual machines.   http://www.osrg.net/sheepdog/

# 2.2 社区

**文档：** https://github.com/collie/sheepdog/wiki

社区邮件列表、commit数、开发者数量和ceph相比是一个数量级的差距。

# 2.3 影响力

和OpenStack整合。


在Google的搜索结果：

* openstack sheepdog - 26,600 条结果
* openstack ceph - 93,900 条结果



# 3. 结论

* ceph背后的力量比sheepdog大。
* ceph社区比sheepdog活跃。
* ceph的影响力比sheepdog大
* ceph的功能比sheepdog强大
* ceph比sheepdog复杂




