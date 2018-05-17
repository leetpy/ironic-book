==================
4.0 Inspector 介绍
==================

在我们注册完 ironic-node 之后，我们需要把裸机的硬件信息更新到 node 表里。如果数量比较少，我们可以手动添加，但是如果是大规模部署，显然不适合手动添加。另外还有 port 的创建，裸机网口和交换机的链接关系等，都不适合自动添加。

ironic inspector 主要是帮助我们收集裸机信息的。

Inspector 安装
---------------

选择一个计算节点，安装 ironic-inspector，inspector 可以和 ironic-conducor 在同一个节点，也可以在不同节点。

.. code-block:: shell

    sudo yum install openstack-ironic-inspector

创建数据库
^^^^^^^^^^

.. code-block:: shell

    CREATE DATABASE ironic_inspector CHARACTER SET utf8;

    GRANT ALL PRIVILEGES ON ironic_inspector.* TO 'inspector'@'localhost' \
    IDENTIFIED BY 'inspector';

    GRANT ALL PRIVILEGES ON ironic_inspector.* TO 'inspector'@'%' \
    IDENTIFIED BY 'inspector';

创建完数据库之后，配置数据库连接，并生成数据库表。


.. code-block:: shell

    $ cat /etc/ironic-inspector/inspector.conf

    [database]
    connection =  mysql+pymysql://inspector:inspector@10.43.210.22/ironic_inspector?charset=utf8

调用 ``ironic-inspector-dbsync`` 生成表


.. code-block:: shell
    
    ironic-inspector-dbsync --config-file /etc/ironic-inspector/inspector.conf upgrade

keystone 注册
^^^^^^^^^^^^^

.. code-block:: shell

    openstack user create --password ironic_inspector \
    --email ironic_inspector@example.com ironic_inspector

    openstack role add --project service --user ironic_inspector admin

    openstack service create --name ironic-inspector --description \
    "Ironic inspector baremetal provisioning service" baremetal-introspection

Inspector 配置
---------------

inspector 提供了两个服务 ``openstack-ironic-inspector`` 
和 ``openstack-ironic-inspector-dnsmasq``

Dnsmasq 配置
^^^^^^^^^^^^

.. code-block:: shell

    $ cat /etc/ironic-inspector/dnsmasq.conf

    port=0
    interface=eth0
    bind-interfaces
    dhcp-range=192.168.2.10,192.168.2.200
    enable-tftp
    tftp-root=/tftpboot
    dhcp-boot=pxelinux.0
    dhcp-sequential-ip


这里的 interface 是提供 DHCP 服务的网口，也是 tftp 的网口，
我们需要给 ``eth0`` 配置一个同网段的 IP。

.. code-block:: shell

    $ cat /etc/sysconfig/network-scripts/ifcfg-eth0

    TYPE=Ethernet
    BOOTPROTO=static
    NAME=eth0
    DEVICE=eth0
    ONBOOT=yes
    IPADDR=192.168.2.2
    NETMASK=255.255.255.0

Tftp 配置
^^^^^^^^^

inspector 和 provision 使用的是同一组 deploy 内核镜像
这里假设 tftp 服务器已经配置好了，我们这里只添加
default 文件，文件内容如下：

.. code-block:: shell

    default introspect

    label introspect
    kernel deploy.vmlinuz
    append initrd=deploy.initrd ipa-inspection-callback-url=http://192.168.2.2:5050/v1/continue ipa-inspection-collectors=default ipa-collect-lldp=1 systemd.journald.forward_to_console=no selinux=0

    ipappend 3


在 default 文件中，确认如下几个配置：

* ``ipa-inspection-callback-url``，这个 IP 填写 tftp 的 IP 地址，裸机需要访问这个 IP;
* ``ipa-collect-lldp=1`` 是让 IPA 收集 lldp 报文;
* ``selinux=0`` 防止某些情况下无法登陆 initramfs;

Ironic 配置
^^^^^^^^^^^
要在 ironic 里使用 inspector，需要先在 ironic 配置文件里使能 inspector，
配置如下：

.. code-block:: shell

    $ cat /etc/ironic/ironic.conf

    [inspector]
    enabled = true
    service_url = http://10.43.210.23:5050

这里的 ``service_url`` 也可以不写，ironic 会根据注册的 
endpoint 来获取。

组网说明
--------

由于 inspector 的 DHCP 服务是不区分 mac 地址的，如果在 
flat 网络中使用，跟 neutron-dhcp-agent 有冲突。因此如果
是 flat 网络，建议分开进行 inspector 和 provision。 如果是
vlan 网络，把 inspector 和 provision 放到不同的 vlan 即可。

说明
----

如果把 ironic-inspector 和 ironic-conductor 放到同一个节点，
那么 provision流程和 inspector 流程是公用一个 tftp 服务器，
然后监听不同的网口。在正常情况下是没有冲突的，但是如果部署
流程失败了，导致 tftp 数据有残留，那么后续可能进行 inspector
流程时，会下到 deploy 的镜像和配置文件，从而导致 inspector 失败。
