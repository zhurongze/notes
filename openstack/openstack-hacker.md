
**目录**

* OpenStack Hacker
* 0级
* 1级
* 2级
* 3级
* 4级
* 5级
* 6级

***

# OpenStack Hacker #

态度：开放、主动、沟通 

影响力：能说、能写、能分享 

四化：自动化、流程化、系统化、文档化 

***

# 0级 #

掌握一些基本技能：python、c、linux、git、unittest、vim/emacs


## python学习 ##

书籍：

* 《python参考手册》
* 《python基础教程》

教程： [codecademy](http://www.codecademy.com/zh/tracks/python)

挑战： [Python Challenge](http://www.pythonchallenge.com)

文档： [Python v2.7.3 documentation](http://docs.python.org/2/index.html) 

高级： [The Hitchhiker's Guide to Python!](http://docs.python-guide.org/en/latest/)

## Linux环境学习 ##

在线书籍： [鸟哥的Linux私房菜](http://vbird.dic.ksu.edu.tw)

## Linux编程学习 ##

书籍：

* 《Unix环境高级编程》
* 《UNIX系统编程》

## Git学习 ##

书籍：

* 《版本控制之道》

在线书籍：

* [Pro Git](http://git-scm.com/book/zh)
* [GotGitHub](http://www.worldhello.net/gotgithub/index.html) 

教程：

* [tryGit](http://try.github.com/levels/1/challenges/1) 
* [GitImmersion](http://gitimmersion.com/lab_01.html) 

进阶：

* [visual-git-guide](http://marklodato.github.com/visual-git-guide/index-zh-cn.html) 
* [a-successful-git-branching-model](http://nvie.com/posts/a-successful-git-branching-model/) 

最常用的git命令：

* [Everyday GIT With 20 Commands Or So](http://www.kernel.org/pub/software/scm/git/docs/everyday.html) 

## Unittest学习 ##

教程： [python unittest](http://docs.python.org/2/library/unittest.html) 

## VIM学习 ##

教程： [vim-adventures](http://vim-adventures.com/) 


***

# 1级 #

## 学习openstack的基本概念  ##

* [介绍]( https://www.openstack.org/software/)
* [Compute管理员手册](http://docs.openstack.org/trunk/openstack-compute/admin/content/ch_getting-started-with-openstack.html)
* [Network管理员手册](http://docs.openstack.org/folsom/openstack-network/admin/content/)
* [Object Storage管理员手册](http://docs.openstack.org/folsom/openstack-object-storage/admin/content/)
* [OpenStack文档](http://docs.openstack.org/)
* [OpenStack词汇表](http://docs.openstack.org/glossary/content/glossary.html)

## 使用openstack ##

* [使用stacklab.org创建虚拟机](http://stacklab.org)
* [使用界面管理openstack](http://stacklab.org) 
* [使用命令行管理openstack](http://docs.openstack.org/cli/quick-start/content/index.html)
* 使用curl管理openstack 

## 使用devstack安装openstack开发测试环境 ##

* [在stacklab.org上创建虚拟机](http://stacklab.org)
* [在虚拟机上使用devstack安装 ](http://devstack.org)
* [阅读devstack.sh脚本 ](http://devstack.org)

## 使用deb包安装openstack ##

* [使用VirtualBox安装OpenStack](http://wiki.stacklab.org/doku.php?id=stacklab:documentation:use-virtualbox-install-openstack)
* [在Ubuntu 12.04上安装OpenStack Folsom版(FlatDHCP+Multihost)](http://wiki.stacklab.org/doku.php?id=stacklab:documentation:install-openstack-folsom-with-nova-network)

## 使用源码安装openstack ##

了解openstack的配置

* [官方教程](http://docs.openstack.org/install/)
* 我们自己的教程：__TODO__

## 调试openstack ##

各种调试方法
* pdb

## 学习python的PEP8 ##

* [PEP8规范](http://www.python.org/dev/peps/pep-0008)

## 学习python的基本库 ##

* SQLAlchemy
* pyzmq
* libvirt
* pylint
* nosetest
* __TODO__

## 学习测试 ##

* unit test
* function test
* integration test

## 学习数据库 ##

* mysql

## 写blog ##

***

# 2级 #

## 学习python的库和用法 ##

理解python中optparse.OptionParser类。

* http://docs.python.org/library/optparse.html

理解collections.Mapping类。

* http://docs.python.org/library/collections.html

分析浅拷贝，深拷贝

* http://blog.csdn.net/winterttr/article/details/2590741
* http://longmans1985.blog.163.com/blog/static/70605475200991603624942/
* http://book.51cto.com/art/200806/77233.htm

LoggerAdapter类

* http://docs.python.org/howto/logging-cookbook.html#context-info中。

介绍rabbitmq

* http://blog.ftofficer.com/2010/03/translation-rabbitmq-python-rabbits-and-warrens/
* http://kombu.readthedocs.org/en/latest/introduction.html#synopsis

Python Decorators入门

* http://blog.csdn.net/beckel/article/details/3585352

Python @classmethod @staticmethod的区别。


五分钟理解元类（Metaclasses）

* http://www.cnblogs.com/coderzh/archive/2008/12/07/1349735.html

nova中用到的python知识

* http://canx.me/2011/12/%E4%B8%80%E4%BA%9Bpython/

python中类的总结

* http://ipseek.blog.51cto.com/1041109/802243

with的总结

* http://effbot.org/zone/python-with-statement.htm

Pool类

* http://nullege.com/codes/search/eventlet.pools.Pool

paste模块

* http://pythonpaste.org/

python魔术方法

* http://pycoders-weekly-chinese.readthedocs.org/en/latest/issue6/a-guide-to-pythons-magic-methods.html

Routes模块

* http://routes.readthedocs.org/en/latest/index.html

2.5版yield之学习心得

## 学习OpenStack HACKING.rst ##

在nova目录下的HACKING.rst文件

## 学习nova的架构 ##

* nova的架构分析：__TODO__(rongze)

## 学习nova的代码 ##

* [openstack的设计思想](http://wiki.openstack.org/BasicDesignTenet)
* nova的架构源码分析：__TODO__(rongze)

## 学习nova的各种机制 ##

* [quota](http://freedomhui.com/?p=144)
* [policy](http://freedomhui.com/?p=1406)
* 其他：__TODO__(rongze)

## 学习nova的单元测试 ##

* 单元测试的原理： __TODO__(rongze)
* nova单元测试架构： __TODO__(rongze)

## 学习CI ##

学习代码提交技能:git review、gerrit、launchpad。  __TODO__(rongze)

## 写blog，技术分享 ##

***

# 3级 #

## 提交patch(自己的项目) ##

## 提交patch(openstack社区) ##

## 开发nova的扩展API ##

__TODO__(rongze)

* 设计API
* 编写nova/api

## 开发cinder的driver ##

学习开发driver：__TODO__(rongze)

## 写blog，技术分享 ##

***

# 4级 #

## Review别人的代码 ##

__TODO__(rongze)

## 参加IRC meeting ##

* [会议安排](http://wiki.openstack.org/Meetings/)

## 参加邮件列表讨论 ##

__TODO__(rongze)

## 跟踪openstack项目的发展 ##

__TODO__(rongze)

## 写blog，技术分享 ##

***

# 5级 #

## OpenStack二次开发 ##

熟悉openstack某个子项目，能够熟练进行二次开发：__TODO__(rongze)

## OpenStack新项目 ##

能够使用openstack的架构，开发新的项目：__TODO__(rongze)

## 写blog，技术分享 ##

***

# 6级 #

__TODO__(rongze)

***

# 领域知识 # 
## 虚拟化 ##

## 网络 ##

## 存储 ##

### 对象存储 ###

### 块存储 ###

### 分布式存储 ###
