=====================
8.1 Config drive 使用
=====================

Config drive 介绍
=================

Config drive 本质上是 base64 编码之后的数据。
Config drive 只能是 ISO 9600 或者是 VFAT 文件系统，这个跟 nova 的配置有关。
我们看看 Config driver 解压之后的目录结构：

.. code-block:: console

    $ tree configdrive
    /configdrive
    ├── ec2
    │   ├── 2009-04-04
    │   │   ├── meta-data.json
    │   │   └── user-data
    │   └── latest
    │       ├── meta-data.json
    │       └── user-data
    └── openstack
        ├── 2012-08-10
        │   ├── meta_data.json
        │   └── user_data
        ├── 2013-04-04
        │   ├── meta_data.json
        │   └── user_data
        ├── 2013-10-17
        │   ├── meta_data.json
        │   ├── user_data
        │   └── vendor_data.json
        ├── 2015-10-15
        │   ├── meta_data.json
        │   ├── network_data.json
        │   ├── user_data
        │   └── vendor_data.json
        └── latest
            ├── meta_data.json
            ├── network_data.json
            ├── user_data
            └── vendor_data.json

Config drive 分区了两个部分，分别是 ``ec2`` 和 ``openstack``,
其中 ``ec2`` 是亚马逊云使用的，我们这里只介绍 ``openstack``.

Openstack  组下有分了 ``2012-08-10``,  ``2013-10-17``, ``2015-10-15`` 和 ``latest``,
这些代表这 Openstack 的版本，对应如下：

* 2012-08-10: Essex
* 2013-10-17: HAVANA
* 2015-10-15: LIBERTY

每个版本下面对应如下文件：

* meta_data.json: 存储 nova instance 相关信息;
* network_data.json: 存储租户网络相关信息;
* user_data: 用户自定义脚本;
* vendor_data.json: 一般为空;

我们比较关注的是 ``network_data.json`` 和 ``user_data``,
通过 ``network_data.json`` 我们知道该怎么配置网络。
而 ``user_data`` 是用户自己编写的脚本。

Config drive 信息保存
=====================

Config drive 本身并不区分虚拟机和物理机，但是虚拟机和物理机 Config drive 存储的方式有些差异。

虚拟机
------

对于虚拟机比较简单，在虚拟机所在计算节点生成 disk.config 文件，修改虚机的 xml 文件，
添加一个 ``disk`` 标签。这样虚机启动时就能看到这个分区了。

既然已经有了分区，那么程序怎么知道 config drive 数据存放在哪呢？
Nova 在生成 config driver 时会加上一个 ``config-2`` 的 label。
也就是说不管 config drive 在哪个分区，盘符是什么，
都能通过 ``/dev/disk/by-label/config-2`` 找到它（仅 linux 系统）。

物理机
------

物理机并不存在 xml 文件，没有办法直接挂载。目前的做法是 Nova 把 config drive 数据传给 ironic,
ironic 在部署的时候把 config drive 写到物理机磁盘的最后。

.. NOTE::

    Config drive 最大是 64M，ironic 会在物理机磁盘的最后创建一个 64M 的分区，
    然后把数据 dd 到该分区。


使用 config drive
=================

使用 config drive 需要在命令行指定 ``--config-drive true``

.. code-block:: console

    nova boot --config-drive true --flavor baremetal --image test-image instance-1

或者通过 nova 配置文件，强制所有 instance 使用 config drive:

.. code-block:: ini

    [DEFAULT]
    ...

    force_config_drive=True


