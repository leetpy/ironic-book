=============
1.3 TFTP 配置
=============

ironic 使用 pxe 流程进行部署。pxe 主要由 DHCP 和 TFTP 两个服务来
完成。ironic 自己并未提供这两个服务，其中 DHCP 由 neutorn 来提供。
TFTP 则是用户自己配置 xinet 和 tftp-server 来完成。

安装
----
#. 安装 rpm 包

    .. code-block:: console
    
        # 创建目录并修改权限
        $ sudo mkdir -p /tftpboot/pxelinux.cfg
        $ sudo chown -R ironic /tftpboot
    
        # 安装相关 rpm 包
        $ sudo yum install tftp-server xinetd syslinux-tftpboot
    
        # 复制 pxe 引导文件到 tftp 目录
        $ sudo cp /usr/lib/PXELINUX/pxelinux.0 /tftpboot
        $ sudo cp /boot/extlinux/chain.c32 /tftpboot


#. 创建 map-file 文件，内容如下：

    .. code-block:: console
    
        $ cat /tftpboot/map-file
        re ^(/tftpboot/) /tftpboot/\2
        re ^/tftpboot/ /tftpboot/
        re ^(^/) /tftpboot/\1
        re ^([^/]) /tftpboot/\1

#. 修改 tftp 配置文件

    .. code-block:: console
    
        $ cat /etc/xinetd.d/tftp 
        {
            socket_type     = dgram
            protocol        = udp
            wait            = yes
            user            = root
            server          = /usr/sbin/in.tftpd
            server_args = -v -v -v -v -v --map-file /tftpboot/map-file /tftpboot
            disable = no
            per_source      = 11
            cps         = 100 2
            flags           = IPv4
        }


防火墙
------

centos 默认开启了 selinux， selinux 会拦截 tftp 报文，
我们需要做如下配置，让 tftp 报文通过。

.. code-block:: shell

    setsebool -P tftp_home_dir 1
