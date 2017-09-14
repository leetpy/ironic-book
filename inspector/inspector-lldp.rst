================================
4.1 Inspector 兼容 SDN lldp 报文
================================

部分 SDN 交换机使用的 lldp 报文跟正常的 lldp 报文有些差异。要兼容这部分交换机，我们需要修改 inspector 的实现代码。

lldp 介绍
---------

这里先简单介绍一下 lldp 报文结构， lldp 采用 TLV(type length value)方式存储数据。报文结构如下图所示：

.. image:: http://img.blog.csdn.net/20130902202844578?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ29vZGx1Y2t3aGg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast

TLV 的类型域的定义及分配如下图所示：

.. image:: http://img.blog.csdn.net/20130902202901875?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ29vZGx1Y2t3aGg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast

我们在 ironic 中使用的 local_link_connection 和报文的对应关系如下表：

======== ========== ==========================
TLV TYPE TLV NAME   local link connection name
======== ========== ==========================
1        chassis ID switch_id
2        port_id    port_id
======== ========== ==========================

再来介绍下 SDN 的 lldp 报文和普通的 lldp 报文有啥差异，首先不管是 SDN 的 lldp 还是普通交换的 lldp， 我们要收集的都是 chassis ID 和 port id，那有什么差异呢？这是因为 chassis ID 和 port id，还存在 subtype， 通过 subtype 可以定义不同的 chassis ID 和 port id 值。

先看看 chassis ID 报文结构：

.. image:: http://img.blog.csdn.net/20130902203045343?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ29vZGx1Y2t3aGg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast

chassis子类型所可能的取值如图所示：

.. image:: http://img.blog.csdn.net/20130902203059937?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ29vZGx1Y2t3aGg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast

再看看  port id 值

.. image:: http://img.blog.csdn.net/20130902203117000?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ29vZGx1Y2t3aGg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast

其子类型的可能取值如下图所示：

.. image:: http://img.blog.csdn.net/20130902203136812?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ29vZGx1Y2t3aGg=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast

SDN 的 lldp 和普通 lldp 报文差异如下表所示：

+---------+----------+-------------+---------+-------------------+
| SWITCH  | TLV TYPE | TLV NAME    | subtype | subtype value     |
+=========+==========+=============+=========+===================+
| SDN     |   1      | chassis id  | 7       | Locally assigned  |
+---------+----------+-------------+---------+-------------------+
| normal  |   1      | chassis id  | 4       | Mac Address       |
+---------+----------+-------------+---------+-------------------+
| SDN     |   2      | port id     | 2       | Port component    |
+---------+----------+-------------+---------+-------------------+
| normal  |   2      | port id     | 3       | Mac Address       |
|         +          +             +---------+-------------------+
|         |          |             | 5       | Interface name    |
|         +          +             +---------+-------------------+
|         |          |             | 7       | Locally assigned  |
+---------+----------+-------------+---------+-------------------+
