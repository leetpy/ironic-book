====================
9.1 Inspect 常见问题
====================

Introspection 开始时出错
~~~~~~~~~~~~~~~~~~~~~~~~

* *Invalid provision state "available"*

  进行Introspection 时，node 的初始状态必须为 ``manageable``,
  如果是 ``available`` 状态，可通过如下命令切换::

    ironic node-set-provision-state <IRONIC NODE> manage

Introspection 超时
~~~~~~~~~~~~~~~~~~

Introspection 超时有三种可能（默认超时时间是 60min, 可以通过配置项 timeout 来更改）：

#. 处理数据错误.
   参考 `Troubleshooting data processing`_.

#. 下载镜像错误。参考 `Troubleshooting PXE boot`_ .

#. 运行错误. 参考 `Troubleshooting ramdisk run`_.

Troubleshooting data processing
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
主要检查 **ironic-inspector** 的日志::

    sudo journalctl -u openstack-ironic-inspector

(use ``openstack-ironic-discoverd`` for version < 2.0.0).


如果配置了 ``ramdisk_error`` 和 ``ramdisk_logs_dir``,
**ironic-inspector** 会接收裸机的日志，并保存到 ``ramdisk_logs_dir`` 目录。
implementation.

Troubleshooting PXE boot
^^^^^^^^^^^^^^^^^^^^^^^^

Introspection 大部分问题都是 PXE 启动失败，如果带外网络可以连接，登录 KVM 排查问题。

查看 DHCP 和 TFTP 日志::

    $ sudo journalctl -u openstack-ironic-inspector-dnsmasq

(use ``openstack-ironic-discoverd-dnsmasq`` for version < 2.0.0).

使用 ``tcpdump`` 抓 DHCP 和 TFTP 报文
::

    $ sudo tcpdump -i any port 67 or port 68 or port 69

把上面的 any 换成实际的 DHCP 口，并观察服务器是否收到 DHCP 和 TFTP 报文。

如果发现裸机没有从 PXE 启动，或者启动的网口不正确，请配置 BIOS 和交换机。

如果 PXE 启动失败，做如下检查:

#. 交换机配置正确，检查 VLAN 信息，DHCP 配置.

#. 检查是否有防火墙规则，拦截了 67 端口.

如果裸机要到了 DHCP IP， 但是下载内核镜像失败了，做如下检查:

#. TFTP 正常且可以访问，检查 xinet 服务(或者dnsmasq)，检查 SELinux,

#. 没有防火墙规则拦截 TFTP 报文,

#. DHCP options 中的 TFTP server 地址正确,

#. ``pxelinux.cfg/default`` 中的 kernel 和 ramdisk 路径正确。

.. note::
    如果使用的是 iPXE，检查 HTTP 服务器的日志以及 iPXE 的配置。

Troubleshooting ramdisk run
^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果配置了接收日志，先查看日志，检查错误，具体配置参考：
`Troubleshooting data processing`_ 章节。

如果 KVM 获取网络可以连接裸机，登录上去，检查服务状态，查看 journalctl 日志。

关于怎么动态登录裸机，可以参考文档：

.. _dynamic-login: http://docs.openstack.org/developer/diskimage-builder/elements/dynamic-login/README.html

