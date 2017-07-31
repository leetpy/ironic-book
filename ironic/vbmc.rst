==================
7.2 Virtualbmc使用
==================

通常情况下，我们要使用 IPMI必须使用有带外管理功能的物理机。但是在很多测试环境，我们使用的是虚拟机。virtualbmc是一个可以使用 IPMI命令来控制虚机的openstack 组件。

Virtualbmc 安装
---------------

.. code-block:: shell

    pip install virtualbmc

如果 pip 安装失败，可能是你环境中依赖包没有安装。

.. code-block:: shell

    sudo yum install gcc libvirt-devel python-devel


Virtualbmc 使用
---------------
#. 查看环境中的虚拟机
   
    .. code-block:: shell

     $ virsh list --all
     Id    Name                           State
     ----------------------------------------------------
     12    centos7.0-3                    running

#. 给虚机添加 vmbc

    .. code-block:: shell

        vbmc add centos7.0-3 --port 6230

#. 查看 vmbc 信息
   
    .. code-block:: shell

        $ vbmc list
        +-------------+--------+---------+------+
        | Domain name | Status | Address | Port |
        +-------------+--------+---------+------+
        | centos7.0-3 |  down  |    ::   | 6233 |
        +-------------+--------+---------+------+


#. 启动vbmc

    .. code-block:: shell

        $ vbmc start centos7.0-3

#. ipmi 控制虚机

   这里 ipmi 的默认用户名和密码分别为 admin 和 password， 用户可以通过—username 和 —password 来指定自己的用户名和密码。

    .. code-block:: shell

        $ ipmitool -I lanplus -H 127.0.0.1 -U admin -P password -p 6233 power status
        Chassis Power is on

常用命令
--------

.. code-block:: shell

    # 查看帮助
    $ vbmc --help

    # 添加vbmc
    $ vbmc add node-0

    # 启动vbmc
    $ vbmc start node-0

    # 停止vmbc
    $ vbmc stop node-0

    # 查看vmbc 列表
    $ vbmc list

    # 查看某个虚机vmbc 信息
    $ vbmc show node-0


说明
----

* vmbc 使用不同的端口号来映射到不同的虚机；
* 使用vbmc add 命令时，是在用户的$HOME/.vbmc/node_name/config 里记录 vbmc 的映射信息，vbmc list 也是查看当前用户的 vbmc信息。虽然不同用户记录文件在不同的地方，但是端口号不能重复，ipmitool 命令本身不区分
* vmbc 支持大部分的 IPMI 命令，但仍然有部分命令不支持， 例如 sol；

