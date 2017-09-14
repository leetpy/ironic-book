============
IPA 源码分析
============

agent 主要有三个类构成：

* IronicPythonAgentStatus
* IronicPythonAgentHeartbeater
* IronicPythonAgent

``IronicPythonAgentStatus`` 比较简单，只记录了 ``started_at`` 和 ``version`` 两个属性。

在小系统运行时，会起 ``ironic-python-agent`` 服务。服务的入口是 IronicPythonAgent 的 run() 方法。
