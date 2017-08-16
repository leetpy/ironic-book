==============
2.1 Ironic-api
==============

目前社区新项目都使用 pecan 框架，通常 pecan 的目录结构如下:

.. code-block:: console

    ironic/api/
    ├── app.py
    ├── app.wsgi
    ├── config.py
    ├── controllers
    │   ├── base.py
    │   ├── __init__.py
    │   ├── link.py
    │   ├── root.py
    │   └── v1
    │       ├── driver.py
    │       ├── __init__.py
    │       ├── node.py
    │       ├── port.py
    │       ├── state.py
    │       ├── types.py
    │       ├── versions.py
    │       ├── volume.py
    │       └── volume_target.py
    ├── expose.py
    ├── hooks.py
    ├── __init__.py
    └── middleware
        ├── auth_token.py
        ├── __init__.py
        └── parsable_error.py

* app.py 一般包含了Pecan应用的入口，包含应用初始化代码;
* config.py 包含Pecan的应用配置，会被app.py使用;
* controllers/ 这个目录会包含所有的控制器，也就是API具体逻辑的地方;
* controllers/root.py 这个包含根路径对应的控制器;
* controllers/v1/ 这个目录对应v1版本的API的控制器。如果有多个版本的API，你一般能看到v2等目录; [1]_

application 配置
----------------

Pecan的配置很容易，通过一个Python源码式的配置文件就可以完成基本的配置。
这个配置的主要目的是指定应用程序的root，然后用于生成WSGI application。

.. code-block:: python

    # file: ironic/api/config.py
    server = {
        'port': '6385',
        'host': '0.0.0.0'
    }

    app = {
        'root': 'ironic.api.controllers.root.RootController',
        'modules': ['ironic.api'],
        'static_root': '%(confdir)s/public',
        'debug': False,
        'acl_public_routes': [
            '/',
            '/v1',
            # IPA ramdisk methods
            '/v1/lookup',
            '/v1/heartbeat/[a-z0-9\-]+',
        ],
    }

上面这个app对象就是Pecan的配置，每个Pecan应用都需要有这么一个名为app的配置。
app配置中最主要的就是root的值，这个值表示了应用程序的入口，也就是从哪个地方开始解析HTTP的根path：/。
hooks对应的配置是一些Pecan的hook，作用类似于WSGI Middleware。

有了app配置后，就可以让Pecan生成一个WSGI application。在 Ironci 项目中，
ironic/api/app.py文件就是生成WSGI application的地方，我们来看一下这个的主要内容：



参考文献
========

.. [1] http://www.infoq.com/cn/articles/OpenStack-demo-API3
