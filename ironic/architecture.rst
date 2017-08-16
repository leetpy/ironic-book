===================
2.0 Ironic 部署架构
===================

Openstack 早期的项目采用 ``Paste + PasteDeploy + Routes + WebOb`` 架构。
后来，OpenStack社区的人受不了这么啰嗦的代码了，决定换一个框架，他们最终选中了Pecan。
Pecan 框架有如下好处：[1]_

* 不用自己写WSGI application了
* 请求路由很容易就可以实现了

总的来说，用上Pecan框架以后，很多重复的代码不用写了，开发人员可以专注于业务，也就是实现每个API的功能。

Ironic 对外提供 restful api 接口，来响应外部请求，
ironic-api 和 ironic-conductor 之间通信则采用 RPC。
我们先看一下 ironic 的部署架构图:

.. image:: ../images/deployment_architecture_2.png


参考文献
--------

.. [1] http://www.infoq.com/cn/articles/OpenStack-UnitedStack-API2
