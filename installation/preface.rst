前言
====

本书主要介绍了 ironic 的基本原理，以及 ironic 在实际环境中的使用。本书采用的环境如下：

* Openstack: Ocata 版本
* OS: CentOS7.3

内容
----

本书主要从以下几个方面介绍 ironic

* Ironic 环境搭建；
* Ironic-api 原理介绍；
* Ironic-conductor 原理介绍；
* Ironic-inspector 原理介绍；
* Ironic-python-agent 原理介绍;
* Ironic 镜像制作；
* 测试环境搭建；
* Cloud-init 使用；
* 常见问题；

术语
----

+-----------+----------------------------------------------+
| 名称      | 含义                                         |
+-----------+----------------------------------------------+
| ironic    |  openstack 组件，主要用来管理裸机            |
+-----------+----------------------------------------------+
| baremetal |  裸金属，裸机，一般只物理服务器              |
+-----------+----------------------------------------------+
| provision |  部署                                        |
+-----------+----------------------------------------------+
| inspector |  主机发现                                    |
+-----------+----------------------------------------------+


openstack 学习
--------------

关于 openstack 的学习，笔者主要从以下几个方面学习：

* 文档: https://docs.openstack.org/developer/ironic/
* IRC 记录: http://eavesdrop.openstack.org/meetings/ironic
* code review: https://review.openstack.org/#/q/status:open
* 查看 bp: https://launchpad.net/
* 阅读源码

Ironic 常用链接
---------------

* IRC LOG: http://eavesdrop.openstack.org/meetings/ironic/2017/
* WIKI: https://wiki.openstack.org/wiki/Ironic
* API: https://developer.openstack.org/api-ref/baremetal/index.html
* 状态机: https://docs.openstack.org/ironic/latest/_images/states.svg
* 近期Bug: http://ironic-divius.rhcloud.com/
* 相关项目: https://governance.openstack.org/tc/reference/projects/ironic.html#mission
* White board: https://etherpad.openstack.org/p/IronicWhiteBoard


About
-----

* Author: liekkas
* Email: leetpy2@gmail.com
