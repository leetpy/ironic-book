.. _troubleshooting:

===================
9.0 Ironic 常见故障
===================

Nova 返回 "No valid host was found" 错误
========================================

有时部署的时候，在 "nova-conductor.log" 或者 http 返回中有如下错误::

    NoValidHost: No valid host was found. There are not enough hosts available.

"No valid host was found" 是指 Nova Scheduler 没有找到合适的裸机来部署新的实例。

出现这个错误时，做如下检查:

#. 确认有足够的裸机在 ``available`` 状态，且没有进入维护模式，没有和 instance 关联
   可以通过如下命令检查::

       ironic node-list --provision-state available --maintenance false --associated false

   如果上面的命令没有显示 node, 使用 ``ironic node-list`` 来检查其它节状态点. 例如,
   ``manageable`` 状态的节点。
   通过如下命令切换到 available::

       ironic node-set-provision-state <IRONIC NODE> provide

   当裸机发生异常时，例如电源同步失败，ironic 会自动把 node 设置成维护模式。
   这时候要检查电源认证信息(eg: ``ipmi_address``, ``ipmi_username`` 和 ``ipmi_password``)
   同时检查带外网络是否正常。如果恢复，通过下面的命令移除维护模式::

       ironic node-set-maintenance <IRONIC NODE> off

   ``ironic node-validate`` 用来检查各个字段是否符合要求,
   执行如下命令，正常情况下不应该有返回::

       ironic node-validate baremetal-0 | grep -E '(power|management)\W*False'

   如果自动清理失败了，ironic 也会把 node 设置为维护模式。

#. 确保每个 ``available`` 状态的 node 的 ``properties`` 属性中包含：
   ``cpus``, ``cpu_arch``, ``memory_mb`` 和 ``local_gb``.
   如果没有这些信息，你需要手动输入或者通过 Inspection 流程来收集。
   正常情况下，看到的信息如下::

        $ ironic node-show <IRONIC NODE> --fields properties
        +------------+------------------------------------------------------------------------------------+
        | Property   | Value                                                                              |
        +------------+------------------------------------------------------------------------------------+
        | properties | {u'memory_mb': u'8192', u'cpu_arch': u'x86_64', u'local_gb': u'41', u'cpus': u'4'} |
        +------------+------------------------------------------------------------------------------------+

   .. warning::
       如果使用 Nova Scheduler 的 exact match filters，要确保上面看到的属性和 flavor 精确匹配。

#. Nova flavor 和 ironic node properties 不匹配
   查看 flaover 信息::

        openstack flavor show <FLAVOR NAME>

   比较 Flavor ``capability:`` 和 Ironic node 的 ``node.properties['capabilities']``.

   .. note::
      capabilities 的格式在 Nova 和 Ironic 中略有差异.
      E.g. in Nova flavor::

        $ openstack flavor show <FLAVOR NAME> -c properties
        +------------+----------------------------------+
        | Field      | Value                            |
        +------------+----------------------------------+
        | properties | capabilities:boot_option='local' |
        +------------+----------------------------------+

      Ironic node::

        $ ironic node-show <IRONIC NODE> --fields properties
        +------------+-----------------------------------------+
        | Property   | Value                                   |
        +------------+-----------------------------------------+
        | properties | {u'capabilities': u'boot_option:local'} |
        +------------+-----------------------------------------+

#. 当 Ironic node 信息发生变化时，Nova 同步信息需要一段时间，
   大概是 1min，可以通过如下命令查看资源信息
   ::

        openstack hypervisor stats show

   查看具体节点的资源信息，使用
   ``openstack hypervisor show <IRONIC NODE>``

#. 查看是哪个 Nova Scheduler filter 没通过, 可以在 ``nova-scheduler`` 日志中搜索::

        Filter ComputeCapabilitiesFilter returned 0 hosts

   找到哪个 filter 把节点都过滤了，关于 fileter 的作用可以参考:
   `Nova filters documentation
   <http://docs.openstack.org/developer/nova/filter_scheduler.html>`_

#. 如果上面的检查都没发现什么问题，检查 Ironic conductor 日志，看看有么有
   相关的错误，导致 "No valid host was found".

Patching the Deploy Ramdisk
===========================

当调试部署或者 inspection 问题时，为了方便定位，你可能想快速修改 ramdisk 的内容。
一种是制作 ramdisk 的时候注入脚本，更通用的做法是直接解压。

创建一个空目录，解压 ramdisk 的内容到该目录::

    mkdir unpack
    cd unpack
    gzip -dc /path/to/the/ramdisk | cpio -id

修改完 ramdisk 文件之后，重新打包 ramdisk::

    find . | cpio -H newc -o > /path/to/the/new/ramdisk

.. note:: 不要修改 kernel 部分(e.g.
          ``tinyipa-master.vmlinuz``), 仅修改 ramdisk 的内容.

.. note:: CentOS 系列的 ramdisk 需要解压 ramdisk 里的 squashfs.


Retrieving logs from the deploy ramdisk
=======================================

当部署失败是，分析 ramdisk 的日志往往很有帮助。当部署失败时，
Ironic 会默认保存 ramdisk 日志到 ``/var/log/ironic/deploy`` 目录。

``/etc/ironic/ironic.conf`` 文件的 ``[agent]`` 组:

* ``deploy_logs_collect``:  Ironic 是否收集部署阶段的日志，有效配置项:

  * ``on_failure`` (**default**): 部署失败时收集。

  * ``always``: 所有情况都收集。

  * ``never``: 不收集。

* ``deploy_logs_storage_backend``: 部署日志存储后端。

  * ``local`` (**default**): 存放在本地文件系统。

  * ``swift``: 存放在 Swift.

* ``deploy_logs_local_path``: 部署日志存放路径，只有配置 ``deploy_logs_storage_backend`` 为 local,
  才有效。默认存放在 **/var/log/ironic/deploy**.


PXE 或 iPXE DHCP 不正确或地址要不到
===================================

这可能是由某些交换机上的生成树协议延迟引起的。
该延迟阻止交换机端口尝试PXE，
因此数据包不会将其发送到DHCP服务器。
解决这个问题你应该设置连接到你的裸金属节点的交换机端口作为边缘或PortFast类型端口。 
以这种方式配置交换机端口一旦建立链接，就转到转发模式。 

Cisco Nexus交换机配置如下：

.. code-block:: bash

    $ config terminal
    $ (config) interface eth1/11
    $ (config-if) spanning-tree port type edge
