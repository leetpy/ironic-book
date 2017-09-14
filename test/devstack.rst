============
7.1 devstack
============

设置的 CI 都是用 devstack 来搭建 openstack 环境的，我们这里也使用 devstack
来搭建 ironic 环境。具体步骤如下：

#. 下载 devstack 代码:

   .. code-block:: console

    git clone https://git.openstack.org/openstack-dev/devstack -b stable/ocata

#. 配置 stack 用户

   .. code-block:: shell 

    # 创建用户
    devstack/tools/create-stack-user.sh

    # 设置权限
    mv devstack /opt/stack
    chown -R stack:stack /opt/stack/devstack

    # 切换用户
    su -- stack 
    cd devstack

#. 编写运行配置文件

   .. code-block:: console

    $ cat /opt/stack/devstack/local.conf
    [[local|localrc]]

    # Configure ironic from ironic devstack plugin.
    enable_plugin ironic https://git.openstack.org/openstack/ironic stable/ocata

    # Install networking-generic-switch Neutron ML2 driver that interacts with OVS
    enable_plugin networking-generic-switch https://git.openstack.org/openstack/networking-generic-switch stable/ocata
    #Q_PLUGIN_EXTRA_CONF_PATH=/etc/neutron/plugins/ml2
    #Q_PLUGIN_EXTRA_CONF_FILES['networking-generic-switch']=ml2_conf_genericswitch.ini

    # Add link local info when registering Ironic node
    IRONIC_USE_LINK_LOCAL=True

    IRONIC_ENABLED_NETWORK_INTERFACES=flat,neutron
    IRONIC_NETWORK_INTERFACE=neutron

    #Networking configuration
    OVS_PHYSICAL_BRIDGE=brbm
    PHYSICAL_NETWORK=mynetwork
    IRONIC_PROVISION_NETWORK_NAME=ironic-provision
    IRONIC_PROVISION_SUBNET_PREFIX=10.0.5.0/24
    IRONIC_PROVISION_SUBNET_GATEWAY=10.0.5.1

    Q_PLUGIN=ml2
    ENABLE_TENANT_VLANS=True
    Q_ML2_TENANT_NETWORK_TYPE=vlan
    TENANT_VLAN_RANGE=100:150
    # Neutron public network type was changed to flat by default
    # in neutron commit 1554adef26bd3bd184ddab668660428bdf392232
    Q_USE_PROVIDERNET_FOR_PUBLIC=False

    # Credentials
    ADMIN_PASSWORD=password
    RABBIT_PASSWORD=password
    DATABASE_PASSWORD=password
    SERVICE_PASSWORD=password
    SERVICE_TOKEN=password
    SWIFT_HASH=password
    SWIFT_TEMPURL_KEY=password

    # Enable Ironic API and Ironic Conductor
    enable_service ironic
    enable_service ir-api
    enable_service ir-cond

    # Enable Neutron which is required by Ironic and disable nova-network.
    disable_service n-net
    disable_service n-novnc
    enable_service q-svc
    enable_service q-agt
    enable_service q-dhcp
    enable_service q-l3
    enable_service q-meta
    enable_service neutron

    # Enable Swift for agent_* drivers
    enable_service s-proxy
    enable_service s-object
    enable_service s-container
    enable_service s-account

    # Disable Horizon
    disable_service horizon

    # Disable Heat
    disable_service heat h-api h-api-cfn h-api-cw h-eng

    # Disable Cinder
    disable_service cinder c-sch c-api c-vol

    # Disable Tempest
    disable_service tempest

    # Swift temp URL's are required for agent_* drivers.
    SWIFT_ENABLE_TEMPURLS=True

    # Create 3 virtual machines to pose as Ironic's baremetal nodes.
    IRONIC_VM_COUNT=3
    IRONIC_BAREMETAL_BASIC_OPS=True

    # Enable Ironic drivers.
    IRONIC_ENABLED_DRIVERS=fake,agent_ipmitool,pxe_ipmitool

    # Change this to alter the default driver for nodes created by devstack.
    # This driver should be in the enabled list above.
    IRONIC_DEPLOY_DRIVER=agent_ipmitool

    # The parameters below represent the minimum possible values to create
    # functional nodes.
    IRONIC_VM_SPECS_RAM=1024
    IRONIC_VM_SPECS_DISK=10

    # Size of the ephemeral partition in GB. Use 0 for no ephemeral partition.
    IRONIC_VM_EPHEMERAL_DISK=0

    # To build your own IPA ramdisk from source, set this to True
    IRONIC_BUILD_DEPLOY_RAMDISK=False

    VIRT_DRIVER=ironic

    # By default, DevStack creates a 10.0.0.0/24 network for instances.
    # If this overlaps with the hosts network, you may adjust with the
    # following.
    NETWORK_GATEWAY=10.1.0.1
    FIXED_RANGE=10.1.0.0/24
    FIXED_NETWORK_SIZE=256

    # Log all output to files
    LOGFILE=$HOME/devstack.log
    LOGDIR=$HOME/logs
    IRONIC_VM_LOG_DIR=$HOME/ironic-bm-logs

    GIT_BASE=http://git.trystack.cn 

#. 开始部署

   .. code-block:: console

    $ ./stack.sh

配置
====

加速
----

#. 由于 openstack 的源在国内下载比较慢，所有把 git 源设置成了 ``trystack`` 的地址；
#. pip 可以提前配置成国内源；
#. yum 或 deb 源提前设置成国内源；

离线模式
--------

local.conf 文件中添加:

.. code-block:: ini

    OFFLINE=True
    RECLONE=no

说明
====

ocata 版本的虚机是挂在 linux 桥上的，实测这种方式跟 ``brbm`` 桥是不通，不知道哪里配置不对，
后来把虚机直接挂到 ``brbm`` 上可以了。
