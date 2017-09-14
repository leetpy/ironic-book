===============
6.0 Ironic 映像
===============

ironic 整个部署流程中有两组映像，分别是 deploy 映像和 user 映像，
其中 deploy 映像用在 inspector 和 部署阶段， user 映像是用户需要安装的操作系统映像。


Deploy 映像
===========

制作ironic deploy镜像其实就是在普通镜像中添加一个ipa服务，用来裸机和ironic通信。
官方推荐制作镜像的工具有两个，分别是CoreOS tools和disk-image-builder
具体链接如下: https://docs.openstack.org/project-install-guide/baremetal/ocata/deploy-ramdisk.html

coreos 映像
-----------

coreos 是一个 docker 镜像， 你可以自己构建，也可以直接下载社区
构建好的: http://tarballs.openstack.org/ironic-python-agent/coreos/files/


映像密码
^^^^^^^^

添加内核启动参数:

.. code-block:: ini

    coreos.autologin

dib 映像
--------

映像密码
^^^^^^^^

有时候，部署会卡很长时间，我们希望能登录到裸机，查看原因。
这个时候需要有密码可以或者是 ssh 能免密码登录。

对应 dib 添加密码，是通过 dynamic-login element 来完成的。
首先制作带 dynamic-login 的映像：

.. code-block:: shell

    disk-image-create ironic-agent centos7 dynamic-login -o ironic-deploy

dynamic-login 的原理是在系统里起一个 ``dynamic-login`` 服务，在系统
上电时，解析 ``/proc/cmdline`` 里的参数，如果用户传了 rootpwd 或者 sshkey，
则写到对应的文件中，这样用户就可以登录系统了。

dynamic-login 使用的是密文，我们可以使用 ``openssl`` 生产密码：

.. code-block:: shell

    $ openssl passwd

    Password: 
    Verifying - Password: 
    mNw2hVHmny2Ho

然后我们把在 ``/etc/ironic/ironic.conf`` 添加我们的密码。

.. code-block:: shell

    $ cat /etc/ironic/ironic.conf

    [pxe]
    pxe_append_params = rootpwd="mNw2hVHmny2Ho"

如果使用 ssh 方式登录，则添加 sshkey

.. code-block:: shell

    $ cat ~/.ssh/id_rsa.pub

    # 添加 sshkey="<your_sshkey>"
    $ cat /etc/ironic/ironic.conf

    [pxe]
    pxe_append_params = sshkey=""

User 映像
=========

user 映像又分为 partition 映像和 whole disk 映像，两者的区别是
whole disk 映像包含分区表和 boot。目前 partition 映像已经很少
使用了，现在基本都使用 whole disk 映像。


镜像驱动问题
------------

我们使用虚机制作的镜像安装在物理机上，很可能缺少驱动，而导致用户
系统起不来。这里我们以 CentOS 为例，说明如何重新制作驱动。

.. code-block:: bash

    mount -o loop CentOS.iso /mnt
    cd /mnt/isolinux
    lsinitrd initrd.img | grep "\.ko" | awk -F / '{print $NF}' | tr "\n" " "

    # 将如上命令获得的ko列表拷贝到 /etc/dracut.conf 中 
    add_drivers+=""

    rm -rf /boot/*kdump.img
    dracut --force

