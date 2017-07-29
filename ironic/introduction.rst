===============
1.0 ironic 简介
===============

ironic 是一个 openstack 项目，主要用来管理裸机。包括裸机的电源控制，系统部署，网络配置等。

ironic 作用
-----------

* 裸机部署: 有时候虚机的性能不能满足我们的要求，这时候可以使用裸机代替；
* 异地重生: 当 nova computer 节点挂了，能及时检查，并迁移虚机；

.. image:: https://docs.openstack.org/developer/ironic/_images/conceptual_architecture.png

ironic 部署架构图如图所示：

.. image:: https://docs.openstack.org/developer/ironic/_images/deployment_architecture_2.png
