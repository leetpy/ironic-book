Enabling drivers and hardware types
===================================

介绍
----

裸机服务将实际的硬件管理委托给驱动来处理。从 Ocata 版本开始支持两种类型的驱动:

* *classic drivers* (例如: ``pxe_ipmitool``, ``agent_ilo`` 等) 
* *hardware types* (例如通用的 ``redfish`` 和 ``ipmi`` 或者厂商相关的 ``ilo`` 和 ``irmc``)。

Driver 反过来说是由许多 *hardware interfaces* 构成的，每个 *hardware interface*
用来处理特定厂商裸机的某些配置。*Classic driver* 把所有的 *hardware interface*
都硬编码到了一起，而 *hardware types* 只申明了兼容哪些 *hardware interface*.

举个例子，``pxe_ipmitool`` 是一个 *classis driver*,在你创建驱动的时候，
使用的 power, boot, console, management, deploy, raid 等模块都是固定好了， 例如:

.. code-block:: python

    class PXEAndIPMIToolDriver(base.BaseDriver):

        def __init__(self):
            self.power = ipmitool.IPMIPower()
            self.console = ipmitool.IPMIShellinaboxConsole()
            self.boot = pxe.PXEBoot()
            self.deploy = iscsi_deploy.ISCSIDeploy()
            self.management = ipmitool.IPMIManagement()
            self.inspect = inspector.Inspector.create_if_enabled(
                'PXEAndIPMIToolDriver')
            self.vendor = ipmitool.VendorPassthru()
            self.raid = agent.AgentRAID()


``ipmi`` 驱动是一个 *hardware types*, 该驱动每个模块会定义支持哪些 *hardware interface*

.. code-block:: python

    class IPMIHardWare(generic.GenericHardware):
        @property
        def supported_console_interfaces(self):
            return [ipmitool.IPMISocatSonsole, ipmitool.IPMIShellinaboxConsole,
                    noop.NoConsole]

简单的说就是 *hardware types* 可以让你自由组合使用 *hardware interface*, 
更加灵活。

从用户角度看，*hardware types* 和 *Classic driver* 都是对应 node 表的 ``driver`` 字段。
但是这两种方式的配置是不同的。

使能 hardware types
-------------------

.. code-block:: ini

    [DEFAULT]
    enabled_hardware_type = ipmi,redfish

由于 driver 是动态组合的，用户需要配置 hardware interfaces.

使能 hardware interfaces
------------------------

有多种类型的 hardware interfaces:

boot

console

deploy

inspect

management

network

power

raid

verndor

配置 interface 默认值
---------------------

当没有明确指定 interface 是，会使用默认值，配置如下:

.. code-block:: ini

    [DEFAULT]
    default_deploy_interface = direct
    default_network_interface = neutron


