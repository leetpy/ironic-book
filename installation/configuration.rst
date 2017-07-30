===============
1.2 Ironic 配置
===============

使用 packstack 安装完 Ironic 默认使用 Flat 网络，
我们这里主要以 Flat 网络为例来介绍，关于 VLAN/VXLAN 将在后面章节介绍。

Ironic 有两种使用方式，一种是 standalone 模式, 另一种是结合 openstack.
如果和 openstack 其它组建集成，ironic 需要做一些配置。

默认情况下这些配置 packstack 都已经配置好了，我们这里还是介绍一下，
一是可以理解 Ironic 和其它组建怎么结合的，另一个是方便自己根据实际环境进行修改。

Keystone 配置
-------------

#. 注册 Bare Metal 服务用户::

    $ openstack user create --password IRONIC_PASSWORD \
        --email ironic@example.com ironic
    $ openstack role add --project service --user ironic admin

#. 注册服务::

    $ openstack service create --name ironic --description \
        "Ironic baremetal provisioning service" baremetal

#. 创建 endpoint::

    $ openstack endpoint create --region RegionOne \
        baremetal admin http://$IRONIC_NODE:6385

    $ openstack endpoint create --region RegionOne \
        baremetal public http://$IRONIC_NODE:6385

    $ openstack endpoint create --region RegionOne \
        baremetal internal http://$IRONIC_NODE:6385

   如果使用 keystone v2 API, 使用如下命令::

        $ openstack endpoint create --region RegionOne \
            --publicurl http://$IRONIC_NODE:6385 \
            --internalurl http://$IRONIC_NODE:6385 \
            --adminurl http://$IRONIC_NODE:6385 \
            baremetal

#. 创建角色::

    $ openstack role create baremetal_admin
    $ openstack role create baremetal_observer

#. 如果你想限制访问，可以创建一个 "baremetal" 的 Project.
   只有这个 project 下的成员才能访问 Ironic 的资源（Nodes, ports 等）::

    $ openstack project create baremetal

   给特定的用户授权::

    $ openstack user create \
        --domain default --project-domain default --project baremetal \
        --password PASSWORD USERNAME
    $ openstack role add \
        --user-domain default --project-domain default --project baremetal \
        --user USERNAME baremetal_observer

Compute 配置
------------

社区的 openstack 默认只能管理裸机或者虚机的一种，不能同时管理。
这时由于 nova 没法区分要部署的是裸机还是虚机，
当然修改代码可以达到同时管理裸机和虚机，这超出了本书范围，就不多介绍了。

Nova 的控制节点和计算节点需要做如下配置:

#. ``default`` 组配置:

   .. code-block:: ini

    [default]

    # Driver to use for controlling virtualization. Options
    # include: libvirt.LibvirtDriver, xenapi.XenAPIDriver,
    # fake.FakeDriver, baremetal.BareMetalDriver,
    # vmwareapi.VMwareESXDriver, vmwareapi.VMwareVCDriver (string
    # value)
    #compute_driver=<None>
    compute_driver=ironic.IronicDriver

    # Firewall driver (defaults to hypervisor specific iptables
    # driver) (string value)
    #firewall_driver=<None>
    firewall_driver=nova.virt.firewall.NoopFirewallDriver

    # The scheduler host manager class to use (string value)
    #scheduler_host_manager=host_manager
    scheduler_host_manager=ironic_host_manager

    # Virtual ram to physical ram allocation ratio which affects
    # all ram filters. This configuration specifies a global ratio
    # for RamFilter. For AggregateRamFilter, it will fall back to
    # this configuration value if no per-aggregate setting found.
    # (floating point value)
    #ram_allocation_ratio=1.5
    ram_allocation_ratio=1.0

    # Amount of disk in MB to reserve for the host (integer value)
    #reserved_host_disk_mb=0
    reserved_host_memory_mb=0

    # Flag to decide whether to use baremetal_scheduler_default_filters or not.
    # (boolean value)
    #scheduler_use_baremetal_filters=False
    scheduler_use_baremetal_filters=True

    # Determines if the Scheduler tracks changes to instances to help with
    # its filtering decisions (boolean value)
    #scheduler_tracks_instance_changes=True
    scheduler_tracks_instance_changes=False

    # New instances will be scheduled on a host chosen randomly from a subset
    # of the N best hosts, where N is the value set by this option.  Valid
    # values are 1 or greater. Any value less than one will be treated as 1.
    # For ironic, this should be set to a number >= the number of ironic nodes
    # to more evenly distribute instances across the nodes.
    #scheduler_host_subset_size=1
    scheduler_host_subset_size=9999999

#. ``ironic`` 组配置:

   * 把 ``IRONIC_PASSWORD`` 换成前面注册的密码；
   * 把 ``IRONIC_NODE`` 换成 ironic-api 所在节点的 IP 地址；
   * 把 ``IDENTITY_IP`` 换成 keystone 所在节点的 IP 地址；

   .. code-block:: ini

    [ironic]

    # Ironic authentication type
    auth_type=password

    # Keystone API endpoint
    auth_url=http://IDENTITY_IP:35357/v3

    # Ironic keystone project name
    project_name=service

    # Ironic keystone admin name
    username=ironic

    # Ironic keystone admin password
    password=IRONIC_PASSWORD

    # Ironic keystone project domain
    # or set project_domain_id
    project_domain_name=Default

    # Ironic keystone user domain
    # or set user_domain_id
    user_domain_name=Default

#. 重启 nova 相关服务:

   .. code-block:: console

    sudo systemctl restart openstack-nova-scheduler
    sudo systemctl restart openstack-nova-compute

Networking 配置
---------------

Ironic 在部署的时候需要使用 Neutron 的 DHCP 服务。

#. 编辑并配置 ``/etc/neutron/plugins/ml2/ml2_conf.ini``:

   .. code-block:: ini

    [ml2]
    type_drivers = flat
    tenant_network_types = flat
    mechanism_drivers = openvswitch

    [ml2_type_flat]
    flat_networks = physnet1

    [securitygroup]
    firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
    enable_security_group = True

    [ovs]
    bridge_mappings = physnet1:br-eth2
    # Replace eth2 with the interface on the neutron node which you
    # are using to connect to the bare metal server

#. 如果 neutron-openstack-agent 服务使用 ``ovs_neutron_plugin.in`` 文件，
   则编辑该文件的 [ovs] 组。

#. 添加 ovs 网桥:

   .. code-block:: console

    $ ovs-vsctl add-br br-int

#. 处理裸机和 openstack 之间的通信:

   .. code-block:: console

    $ ovs-vsctl add-br br-eth2
    $ ovs-vsctl add-port br-eth2 eth2

   这里的 br-eth2 要和前面的配置文件里的 ``bridge_mappings`` 对应，
   eth2 环境实际的物理网卡名。

#. 重启 Open vSwitch agent:

   .. code-block:: console

    # service neutron-plugin-openvswitch-agent restart

#. 重启 Open vSwitch agent 服务之后，应该能看到 br-int 和 br-eth2.

   .. code-block:: console

    $ ovs-vsctl show

    Bridge br-int
        fail_mode: secure
        Port "int-br-eth2"
            Interface "int-br-eth2"
                type: patch
                options: {peer="phy-br-eth2"}
        Port br-int
            Interface br-int
                type: internal
    Bridge "br-eth2"
        Port "phy-br-eth2"
            Interface "phy-br-eth2"
                type: patch
                options: {peer="int-br-eth2"}
        Port "eth2"
            Interface "eth2"
        Port "br-eth2"
            Interface "br-eth2"
                type: internal
    ovs_version: "2.3.0"

#. 创建租户网络：

   .. code-block:: console

    $ neutron net-create --tenant-id $TENANT_ID sharednet1 --shared \
          --provider:network_type flat --provider:physical_network physnet1

    $ neutron subnet-create sharednet1 $NETWORK_CIDR --name $SUBNET_NAME \
          --ip-version=4 --gateway=$GATEWAY_IP --allocation-pool \
          start=$START_IP,end=$END_IP --enable-dhcp

Image 配置
----------

如果使用 ``agent`` 驱动，Ironic 要使用 swift 的 temporary URLS,
因此必须要用 swift 做 glance 后端，关于 Ironic驱动，后面章节会介绍。
