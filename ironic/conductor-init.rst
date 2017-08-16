.. _ironic-conductor-init:

====================
3.1 Conductor 初始化
====================

Ironic 架构
===========

Ironic 由两部分组成: API 和 Conductor, 其中 API 采用 pecan 框架，
使用 restful 方式访问，而 API 和 Conductor 之间采用 RPC 通信，
通常是 rabbitmq.

.. code-block:: console

                        +--------+
                        |        |
                        |  API   |
                        |        |
                        +--------+
                            |              RPC
    ---------------------------------------------------
                |                           |
        +--------------+            +--------------+
        |              |            |              |
        |  Conductor   |            |  Conductor   |
        |              |            |              |
        +--------------+            +--------------+

RPC 初始化
==========

这里 Conductor 是 RPC 的 server 端(消费者)，API 是 RPC 的 client 端(生产者)。
我们先看看 ironic-conductor 的入口函数:

.. code-block:: python

    def main():
        assert 'ironic.conductor.manager' not in sys.modules

        # Parse config file and command line options, then start logging
        # 配置文件的初始化
        ironic_service.prepare_service(sys.argv)

        # 告警初始化
        gmr.TextGuruMeditation.setup_autorun(version)

        # 创建 RPC Server
        mgr = rpc_service.RPCService(CONF.host,
                                     'ironic.conductor.manager',
                                     'ConductorManager')

        issue_startup_warnings(CONF)

        profiler.setup('ironic_conductor', CONF.host)

        launcher = service.launch(CONF, mgr)
        launcher.wait()


RPCService
==========

上面的代码中创建了一个 RPCService 对象。然后设置了 RPC 的 topic, endpoints, serializer。
这里通过反射的方式设置了 manager 属性。

即: ``ironic.conductor.manager.ConductorManager(host, 'ironic.conductor_manager')``

Ironic conductor 通过 oslo.messaging 来创建了 RPC Server，
并把 ConductorManager 注册为 RPC 的 endpoint.
Ironic api 通过 RPC 调用的时候，就调用到了 ConductorManager 里对应的方法。

.. code-block:: python

    class RPCService(service.Service):
    
        def __init__(self, host, manager_module, manager_class):
            super(RPCService, self).__init__()
            self.host = host
            manager_module = importutils.try_import(manager_module)
            manager_class = getattr(manager_module, manager_class)
            self.manager = manager_class(host, manager_module.MANAGER_TOPIC)
            self.topic = self.manager.topic
            self.rpcserver = None
            self.deregister = True

        def start(self):
            super(RPCService, self).start()
            admin_context = context.get_admin_context()

            target = messaging.Target(topic=self.topic, server=self.host)
            endpoints = [self.manager]
            serializer = objects_base.IronicObjectSerializer(is_server=True)
            self.rpcserver = rpc.get_server(target, endpoints, serializer)
            self.rpcserver.start()

            self.handle_signal()
            self.manager.init_host(admin_context)

            LOG.info('Created RPC server for service %(service)s on host '
                     '%(host)s.',
                     {'service': self.topic, 'host': self.host})

Manager 初始化
==============

在 start RPCServer 的时候会调用 manager 的 init_host 函数。
init_host 中主要完成如下操作:

* dbapi 初始化；
* Conductor 保活线程；
* 协程池初始化；
* 哈希环初始化；
* drivers；
* hardware_types；
* NetworkInterfaceFactory；
* StorageInterfaceFactory；


ConductorManager 类图如下图所示:

.. code-block:: console

        +------------------------+   
        |  BaseConductorManager  |
        +------------------------+   
        | + host                 |
        | + topic                |
        | + sensors_notifier     |
        +------------------------+   
        | + init_host()          |
        | + del_host()           |
        | + iter_nodes()         |
        |                        |
        +------------------------+   
                    ^
                    |
                    |
        +----------------------------+   
        |     ConductorManager       |
        +----------------------------+   
        | + power_state_sync_count   |
        +----------------------------+   
        | + create_node()            |
        | + update_node()            |
        | ...                        |
        +----------------------------+   

