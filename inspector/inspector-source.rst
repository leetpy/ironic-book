======================
4.3 Inspector 源码分析
======================

ironic-inspector 使用 flask 框架来编写的。flask 是一个轻量级
的 python web 框架， 详细资料可以查看: http://flask.pocoo.org/

Ironic 处理阶段
---------------

我们首先通过 cli 来触发 ironic inspector 流程。

.. code-block:: shell

    ironic node-set-provision-state <node_uuid> manage
    ironic node-set-provision-state <node_uuid> inspect

我们知道 openstack 组件之前的调用都是通过 restful 接口来完成的，
而组件内部是通过 rpc 调用来完成的。

上面的 cli 会发送 PUT 请求到 ``/v1/nodes/{node_ident}/provision`` ,
ironic-api 收到这个请求后会根据 body 的 ``target`` 字段做处理：

.. code-block:: python

    # ironic/api/controllers/v1/node.py
    def provision():
    ...
        elif target == ir_state.VERBS['inspect']:
            pecan.request.rpcapi.inspect_hardware()

然后 ironic 通过 rpc 调用 inspect_hardware 方法。然后通过发送 http 请求
到 ironic-inspector。inspect 的具体实现是跟 driver 有关，在
driver.inspect.inspect_hardware 中。

.. code-block:: python

    # ironic/drivers/modules/inspector.py
    def _start_inspection(node_uuid, context):
        try:
            _call_inspector(client.introspect, node_uuid, context)
            ...
            
    # ironic_inspector_client/client.py
    def introspect(uuid, base_url=None, auth_token=None,
                   new_ipmi_password=None, new_ipmi_username=None,
                   api_version=DEFAULT_API_VERSION, session=None, **kwargs):
        c = v1.ClientV1(api_version=api_version, auth_token=auth_token,
                        inspector_url=base_url, session=session, **kwargs)
        return c.introspect(uuid, new_ipmi_username=new_ipmi_username,
                            new_ipmi_password=new_ipmi_password)

    # ironic_inspector_client/v1.py
    def introspect(self, uuid, new_ipmi_password=None, new_ipmi_username=None):
        ...
        params = {'new_ipmi_username': new_ipmi_username,
                  'new_ipmi_password': new_ipmi_password}
        self.request('post', '/introspection/%s' % uuid, params=params) 


通过上面的代码，我们可以看到 ironic 发送了 post 请求到 ``/introspection/<uuid>`` 。
下面流程就到了 inspector 了。

Inspector处理阶段
-----------------

ironic-inspector 的 restful api 实现在 ``main.py`` 中，我们首先根据
url 找到对应的函数：

.. code-block:: python

    # main.py
    @app.route('/v1/introspection/<uuid>')
    @convert_exceptions
    def api_introspection(uuid):
        ...
        introspect.introspect(uuid,
                              new_ipmi_credentials=new_ipmi_credentails,
                              token=flask.request.headers.get('X-Auth-Token'))
        return '', 202

再看看introspect的具体实现：

.. code-block:: python

    # introspect.py
    def introspect():
    node_info = node_cache.add_node(node.uuid,
                                    bmc_address=bmc_address,
                                    ironic=ironic)
    future = utils.executor().submit(_background_introspect, ironic, node_info)
    ... 

introspect函数先是更新了ipmi信息，然后在inspector的node表里添加一条记录，
另外在attributes表里添加bmc_address信息。最终后台调用
_background_introspect做主机发现。

.. code-block:: python

    # introspect.py
    def _background_introspect_locked(ironic, node_info):
        try:
            ironic.node.set_boot_device(node_info.uuid, 'pxe',
                                        persistent=False)
        try:
            ironic.node.set_power_state(node_info.uuid, 'reboot')

我们在已经配置好了裸机，并在/tftpboot/pxelinux.cfg/default设置了如下信息:

.. code-block:: shell

    default introspect                                                                                                                                           
    label introspect
    kernel ironic-agent.vmlinuz
    append initrd=ironic-agent.initramfs ipa-inspection-callback-url=http://192.168.2.2:5050/v1/continue ipa-inspection-collectors=default,logs systemd.journald.forward_to_console=no
    ipappend 3

IPA 阶段
--------

裸机从小系统启动之后，会启动ironic-python-agent服务。
该服务会收集裸机的硬件信息，并发送到
ipa-inspection-callback-url指定的url。


Inspector主机上报阶段
----------------------

先看看inspector怎么处理ipa上报的数据：

.. code-block:: python

    # main.py
    @app.route('/v1/continue', methods=[post])
    @convert_exceptions
    def api_continue():
        data = flask.request.get_json(force=True)
        return flask.jsonify(process.process(data))
    # process.py
    def process(introspection_data):
        """Process data from the ramdisk.
        
        This function heavily relies on the hooks to do the actual data processing.
        """
        hooks = plugins_base.processing_hooks_manager()
        failures = []
        for hook_ext in hooks:
            try:
                hook_ext.obj.before_processing(introspection_data)
            except utils.Error as exc:
                ...
        # 根据ipmi_address和macs获取inpsector node
        node_info = _finde_node_info(introspection_data, failures)
        try:
            node = node_info.node()
            ...
            
        try:
            return _process_node(node, introspection_data, node_info)

我们可以看到这里数据是交由process出来，而process函数又调用各种钩子来出来ipa数据。
接着根据ipmi_address查找对应的inspector node，再根据获取到的uuid来得到
ironic node，交由_process_node()函数处理。

.. code-block:: python

    # process.py
    def _process_node(node, introspection_data, node_info):
        ir_utils.check_provision_state(node)
        node_info.create_ports(introspection_data.get('macs') or ())
        _run_post_hooks(node_inof, introspection_data)
        
        if CONF.processing.store_data == 'switf':
            stored_data = {k: v for k, v in introspection_data.items()
                           if k not in _STORAGE_EXCLUDE_KEYS}
            swift_object_name = switf.store.store_introspection_data(stored_data,
                                                                     node_info.uuid)
        ironic = ir_utils.get_client()
        firewall.update_filters(ironic)
        
        node_info.invalidate_cache()
        rules.apply(node_info, introspection_data)
        ...
        utils.executor().submit(_finish, ironic, node_info, introspection_data)
        
    def _finish(ironic, node_info, introspection_data):
        try:
            ironic.node.set_power_state(node_info.uuid, 'off')
        node_info.finished()

我们可以看到，如果配置了store_data=swift，inspector会把ipa上报的数据
存储到swift中。最后的 node_info.finished()是删除inspector数据库中已
完成的数据。
