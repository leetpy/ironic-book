====================
1.1 安装 ironic 环境
====================

本文以 packstack 来安装 openstack 测试环境。

环境准备
--------

这里我们以CentOS7为例，安装openstack ocata版本，
其它版本安装方法类似。packstack目前对NetworkManager
还不支持，我们修改下配置：

.. code-block:: shell

    systemctl disable firewalld
    systemctl stop firewalld
    systemctl disable NetworkManager
    systemctl stop NetworkManager
    systemctl enable network
    systemctl start network

安装 packstack
--------------

.. code-block:: shell

    # 添加packstack yum源
    yum -y install centos-release-openstack-ocata

    # 安装openstack-packstack
    yum -y install openstack-packstack

    # 生成answer-file
    packstack --gen-answer-file=filename

安装 openstack
---------------

如果你安装的是ocata版本，这里packstack有点小bug，
有几个文件需要修改一下，
参考： https://www.redhat.com/archives/rdo-list/2017-March/msg00011.html


两个bug的review链接：

* https://review.openstack.org/#/c/440258/
* https://review.openstack.org/442551

照着review提交的内容修改一下就可以了。接着使用下面的命令安装openstack。

修改 answer-file
----------------

packstack 默认是不安装 ironic 的，需要做如下修改：

.. code-block:: shell

    CONFIG_IRONIC_INSTALL=y

安装 openstack
--------------

.. code-block:: shell

    packstack --answer-file=filename
