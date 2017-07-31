=================
3.2 Ironic driver
=================

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
