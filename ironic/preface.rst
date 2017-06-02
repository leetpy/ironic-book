前言
====

本书主要介绍了 ironic 的基本原理，以及 ironic 在实际环境中的使用。本书采用的环境如下：

* openstack: ocata 版本
* os: centos7.3

内容
----

本书主要从以下几个方面介绍 ironic

* ironic
* ironic-inspector
* disk-image-builder
* ironic-python-agent
* 测试环境
* cloud-init

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


About
-----

* author: liekkas
* email: leetpy2@gmail.com
