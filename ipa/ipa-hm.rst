========================
5.2 IPA Hardware Manager
========================

获取当前主机 bmc 信息
---------------------

.. code-block:: bash

    # 加载相关驱动
    $ modprobe ipmi_msghandler
    $ modprobe ipmi_devintf
    $ modprobe ipmi_si

    # 查看 ipmi 信息
    $ ipmitool lan print
    IP Address Source       : Static Address
    IP Address              : 129.0.0.10
    Subnet Mask             : 255.0.0.0
    MAC Address             : 0c:12:62:e4:c1:19
    SNMP Community String   : Public
    Default Gateway IP      : 129.0.1.1
    802.1q VLAN ID          : Disabled
    802.1q VLAN Priority    : 0
    Cipher Suite Priv Max   : aaaaaaaaaaaaaaa
                            :     X=Cipher Suite Unused
                            :     c=CALLBACK
                            :     u=USER
                            :     o=OPERATOR
                            :     a=ADMIN
                            :     O=OEM
    
获取硬盘信息
-------------

.. code-block:: bash

    $ lsblk -Pbdi -o KNAME,MODEL,SIZE,ROTA,TYPE
    KNAME="sda" MODEL="Logical Volume  " SIZE="298999349248" ROTA="1" TYPE="disk"
    KNAME="sdb" MODEL="Logical Volume  " SIZE="19999998607360" ROTA="1" TYPE="disk"
    KNAME="sdc" MODEL="TOSHIBA MG04ACA4" SIZE="4000787030016" ROTA="1" TYPE="disk"
    KNAME="loop0" MODEL="" SIZE="2158016512" ROTA="1" TYPE="loop"
    KNAME="loop1" MODEL="" SIZE="1163403264" ROTA="1" TYPE="loop"
    KNAME="loop2" MODEL="" SIZE="1163403264" ROTA="1" TYPE="loop"
    KNAME="loop3" MODEL="" SIZE="1163403264" ROTA="1" TYPE="loop"
    KNAME="loop4" MODEL="" SIZE="1163403264" ROTA="1" TYPE="loop"


获取 CPU 信息
-------------

.. code-block:: bash

    $ lscpu
    Architecture:          x86_64
    CPU op-mode(s):        32-bit, 64-bit
    Byte Order:            Little Endian
    CPU(s):                32
    On-line CPU(s) list:   0-31
    Thread(s) per core:    2
    Core(s) per socket:    8
    Socket(s):             2
    NUMA node(s):          2
    Vendor ID:             GenuineIntel
    CPU family:            6
    Model:                 63
    Model name:            Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz
    Stepping:              2
    CPU MHz:               1396.593
    BogoMIPS:              4809.94
    Virtualization:        VT-x
    L1d cache:             32K
    L1i cache:             32K
    L2 cache:              256K
    L3 cache:              20480K
    NUMA node0 CPU(s):     0-7,16-23
    NUMA node1 CPU(s):     8-15,24-31


获取内存信息
------------

内存计算稍微复杂一点，这里分为两部分 ``total`` 和 ``physical_mb``

#. ``total`` 计算


   total, free, buffers, shared 几个字段是调用 python 的 ``_psutil_linux``
   库来获取的。

   .. code-block:: python

    import _psutil_linux
    from _psutil_linux import *

    total, free, buffers, shared, _, _ = _psutil_linux.get_sysinfo()

   .. code-block:: console

    $ cat /proc/meminfo | grep "^Cached:"
    Cached:          1089344 kB

    $ cat /proc/meminfo | grep "^Active:"
    Active:          2491184 kB

    $ cat /proc/meminfo | grep "^Active:"
    Inactive:         977736 kB

   .. code-block:: python

    avail = free + buffers + cached
    used = total - free
    percent = usage_percent((total - avail), total, _round=1)

   最终返回的 ``total`` 就是 ``get_sysinfo()`` 的第一个值。

#. ``physical_mb`` 计算

   ``physical_mb`` 的计算是先通过 dmidecode 获取所有内存条的大小，
   然后把结果求和。

   .. code-block:: bash
    
        $ dmidecode --type 17 | grep Size
          Size: 4096 MB
          Size: No Module Installed
    

网口信息
--------

.. code-block:: bash

    # 获取所有网卡名
    ls /sys/class/net

    # 获取所有网卡设备（去掉虚拟网卡）
    # 所有实际网卡都会链接到一个 PCI 设备
    ls /sys/class/net/eth0/device

    # mac 地址
    cat /sys/class/net/eth0/address

    # carrier
    cat /sys/class/net/eth0/carrier

    # vendor
    cat /sys/class/net/eth0/device/vendor

    # device
    cat /sys/class/net/eth0/device/device


启动方式
--------

如果存在 ``/sys/firmware/efi`` 认为是 uefi 启动方式，
否则认为是 BIOS 启动。

制造商信息
----------

.. code-block:: console

    $ dmidecode --type system


