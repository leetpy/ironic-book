=================
3.2 Ironic driver
=================

驱动类型
========

从 Ocata 开始 ironic 支持两种类型的驱动: *classic drivers* 和 *hardware types*,
以后 ironic 会停止 *classic drivers* 的支持，并只支持 *hardware types*,
详情参考 :doc:`/install/enabling-drivers`


驱动初始化
==========

openstack 的请求流程是 ``restful-api -> rpc`` ,在 ironic 每个
rpc 请求处理中，一般会创建一个 `TaskManager` 的对象，后续的大部分
操作都是通过这个对象来完成的。其中最主要的是 `ironic drive` 的使用。
先看看初始化代码：

.. code-block:: python

    # file: task_manager.py
    class TaskManager(object):

        def __init__(self, context, node_id, shared=False, driver_name=None,
                     purpose='unspecified action'):

            self.driver = driver_factory.build_driver_for_task(
                self, driver_name=driver_name)

可以看到这里的 ``task.driver`` 是采用工厂模式初始化的。看看工厂的具体实现：

.. code-block:: python

    # file: driver_factory.py
    def build_driver_for_task(task, driver_name=None):
        node = task.node
        driver_name = driver_name or node.driver

        driver_or_hw_type = get_driver_or_hardware_type(driver_name)
        try:
            check_and_update_node_interfaces(
                node, driver_or_hw_type=driver_or_hw_type)
        except exception.MustBeNone as e:
            LOG.warning('%s They will be ignored. To avoid this warning, '
                        'please set them to None.', e)

        bare_driver = driver_base.BareDriver()
        _attach_interfaces_to_driver(bare_driver, node, driver_or_hw_type)

        return bare_driver

Ironic driver 类图如下图所示:

.. image:: /images/driver_class.png

假设我们 node 里使用的是 ``pxe_ipmitool`` 驱动，ironic 会根据这个名字到 setup.cfg
中找到对应的类名，这里是: ``ironic.drivers.ipmi:PXEAndIPMIToolDriver``.

初始化 task 的驱动步骤如下:

#. 首先根据 node 里的 driver 名称加载对应的 driver;
#. 检查并更新 node interfaces，这里主要是检查驱动的 hardware interface 有没有设置;
#. 创建 BareDriver 对象 bare_driver;
#. 把 driver 里所有的 interface 复制到 bare_driver 中;

   这里的 interface由如下三部分构成:

   * core_interfaces
   * standard_interfaces
   * vendor

#. 获取 node 里的 dynamic_interfaces 并复制到 bare_drivre 中;

   .. NOTE::
    默认 dynamic_interfaces 是 ['network', 'storage']

这里的 ``driver_or_hw_type`` 就是我们注册 node 使用的 driver。
*classic drivers* 的 driver 都是直接继承 ``BaseDriver`` 的，
而 *hardware types* 的 driver 继承的 ``GenericHardware``.
最终我们在 task 使用的 driver 是 ``BareDriver`` 类型。
