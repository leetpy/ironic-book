.. _troubleshooting:

================
Ironic 常见故障
================

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

   correctly shows total amount of resources in your system. You can also
   check ``openstack hypervisor show <IRONIC NODE>`` to see the status of
   individual Ironic nodes as reported to Nova.

#. Figure out which Nova Scheduler filter ruled out your nodes. Check the
   ``nova-scheduler`` logs for lines containing something like::

        Filter ComputeCapabilitiesFilter returned 0 hosts

   The name of the filter that removed the last hosts may give some hints on
   what exactly was not matched. See `Nova filters documentation
   <http://docs.openstack.org/developer/nova/filter_scheduler.html>`_ for more
   details.

#. If none of the above helped, check Ironic conductor log carefully to see
   if there are any conductor-related errors which are the root cause for
   "No valid host was found". If there are any "Error in deploy of node
   <IRONIC-NODE-UUID>: [Errno 28] ..." error messages in Ironic conductor
   log, it means the conductor run into a special error during deployment.
   So you can check the log carefully to fix or work around and then try
   again.

Patching the Deploy Ramdisk
===========================

When debugging a problem with deployment and/or inspection you may want to
quickly apply a change to the ramdisk to see if it helps. Of course you can
inject your code and/or SSH keys during the ramdisk build (depends on how
exactly you've built your ramdisk). But it's also possible to quickly modify
an already built ramdisk.

Create an empty directory and unpack the ramdisk content there::

    mkdir unpack
    cd unpack
    gzip -dc /path/to/the/ramdisk | cpio -id

The last command will result in the whole Linux file system tree unpacked in
the current directory. Now you can modify any files you want. The actual
location of the files will depend on the way you've built the ramdisk.

After you've done the modifications, pack the whole content of the current
directory back::

    find . | cpio -H newc -o > /path/to/the/new/ramdisk

.. note:: You don't need to modify the kernel (e.g.
          ``tinyipa-master.vmlinuz``), only the ramdisk part.

.. note:: For CoreOS-based ramdisk you also need to unpack and pack back the
          squashfs archive inside the unpacked ramdisk.

API Errors
==========

The `debug_tracebacks_in_api` config option may be set to return tracebacks
in the API response for all 4xx and 5xx errors.

Retrieving logs from the deploy ramdisk
=======================================

When troubleshooting deployments (specially in case of a deploy failure)
it's important to have access to the deploy ramdisk logs to be able to
identify the source of the problem. By default, Ironic will retrieve the
logs from the deploy ramdisk when the deployment fails and save it on the
local filesystem at ``/var/log/ironic/deploy``.

To change this behavior, operators can make the following changes to
``/etc/ironic/ironic.conf`` under the ``[agent]`` group:

* ``deploy_logs_collect``:  Whether Ironic should collect the deployment
  logs on deployment. Valid values for this option are:

  * ``on_failure`` (**default**): Retrieve the deployment logs upon a
    deployment failure.

  * ``always``: Always retrieve the deployment logs, even if the
    deployment succeed.

  * ``never``: Disable retrieving the deployment logs.

* ``deploy_logs_storage_backend``: The name of the storage backend where
  the logs will be stored. Valid values for this option are:

  * ``local`` (**default**): Store the logs in the local filesystem.

  * ``swift``: Store the logs in Swift.

* ``deploy_logs_local_path``: The path to the directory where the
  logs should be stored, used when the ``deploy_logs_storage_backend``
  is configured to ``local``. By default logs will be stored at
  **/var/log/ironic/deploy**.

* ``deploy_logs_swift_container``: The name of the Swift container to
  store the logs, used when the deploy_logs_storage_backend is configured to
  "swift". By default **ironic_deploy_logs_container**.

* ``deploy_logs_swift_days_to_expire``: Number of days before a log object
  is marked as expired in Swift. If None, the logs will be kept forever
  or until manually deleted. Used when the deploy_logs_storage_backend is
  configured to "swift". By default **30** days.

When the logs are collected, Ironic will store a *tar.gz* file containing
all the logs according to the ``deploy_logs_storage_backend``
configuration option. All log objects will be named with the following
pattern::

  <node-uuid>[_<instance-uuid>]_<timestamp yyyy-mm-dd-hh:mm:ss>.tar.gz

.. note::
   The *instance_uuid* field is not required for deploying a node when
   Ironic is configured to be used in standalone mode. If present it
   will be appended to the name.


Accessing the log data
----------------------

When storing in the local filesystem
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

When storing the logs in the local filesystem, the log files can
be found at the path configured in the ``deploy_logs_local_path``
configuration option. For example, to find the logs from the node
``5e9258c4-cfda-40b6-86e2-e192f523d668``:

.. code-block:: bash

   $ ls /var/log/ironic/deploy | grep 5e9258c4-cfda-40b6-86e2-e192f523d668
   5e9258c4-cfda-40b6-86e2-e192f523d668_88595d8a-6725-4471-8cd5-c0f3106b6898_2016-08-08-13:52:12.tar.gz
   5e9258c4-cfda-40b6-86e2-e192f523d668_db87f2c5-7a9a-48c2-9a76-604287257c1b_2016-08-08-14:07:25.tar.gz

.. note::
   When saving the logs to the filesystem, operators may want to enable
   some form of rotation for the logs to avoid disk space problems.


When storing in Swift
~~~~~~~~~~~~~~~~~~~~~

When using Swift, operators can associate the objects in the
container with the nodes in Ironic and search for the logs for the node
``5e9258c4-cfda-40b6-86e2-e192f523d668`` using the **prefix** parameter.
For example:

.. code-block:: bash

  $ swift list ironic_deploy_logs_container -p 5e9258c4-cfda-40b6-86e2-e192f523d668
  5e9258c4-cfda-40b6-86e2-e192f523d668_88595d8a-6725-4471-8cd5-c0f3106b6898_2016-08-08-13:52:12.tar.gz
  5e9258c4-cfda-40b6-86e2-e192f523d668_db87f2c5-7a9a-48c2-9a76-604287257c1b_2016-08-08-14:07:25.tar.gz

To download a specific log from Swift, do:

.. code-block:: bash

   $ swift download ironic_deploy_logs_container "5e9258c4-cfda-40b6-86e2-e192f523d668_db87f2c5-7a9a-48c2-9a76-604287257c1b_2016-08-08-14:07:25.tar.gz"
   5e9258c4-cfda-40b6-86e2-e192f523d668_db87f2c5-7a9a-48c2-9a76-604287257c1b_2016-08-08-14:07:25.tar.gz [auth 0.341s, headers 0.391s, total 0.391s, 0.531 MB/s]

The contents of the log file
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The log is just a ``.tar.gz`` file that can be extracted as:

.. code-block:: bash

   $ tar xvf <file path>


The contents of the file may differ slightly depending on the distribution
that the deploy ramdisk is using:

* For distributions using ``systemd`` there will be a file called
  **journal** which contains all the system logs collected via the
  ``journalctl`` command.

* For other distributions, the ramdisk will collect all the contents of
  the ``/var/log`` directory.

For all distributions, the log file will also contain the output of
the following commands (if present): ``ps``, ``df``, ``ip addr`` and
``iptables``.

Here's one example when extracting the content of a log file for a
distribution that uses ``systemd``:

.. code-block:: bash

   $ tar xvf 5e9258c4-cfda-40b6-86e2-e192f523d668_88595d8a-6725-4471-8cd5-c0f3106b6898_2016-08-08-13:52:12.tar.gz
   df
   ps
   journal
   ip_addr
   iptables

DHCP during PXE or iPXE is inconsistent or unreliable
=====================================================

This can be caused by the spanning tree protocol delay on some switches. The
delay prevents the switch port moving to forwarding mode during the nodes
attempts to PXE, so the packets never make it to the DHCP server. To resolve
this issue you should set the switch port that connects to your baremetal nodes
as an edge or PortFast type port. Configured in this way the switch port will
move to forwarding mode as soon as the link is established. An example on how to
do that for a Cisco Nexus switch is:

.. code-block:: bash

    $ config terminal
    $ (config) interface eth1/11
    $ (config-if) spanning-tree port type edge
