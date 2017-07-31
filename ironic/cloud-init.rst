===================
8.3 cloud-init 介绍
===================

cloud-init 是一个系统配置工具，当系统起来时，
cloud-init 会从固定分区读取数据，然后执行定制化
操作。

cloud-init 有四个执行阶段：

* local:  cloud-init-local.service
* init:   cloud-init.service
* config: cloud-config.service
* final:  cloud-final.service

配置
----

cloud-init 各阶段完成哪些工作可以在 ``/etc/cloud/cloud.cfg`` 中查看

#. local 阶段

   作为 cloud-init 的初始阶段，此时主要完成网口的配置

   ``/usr/bin/cloud-init init --local``

#. init 阶段

#. config 阶段


#. final 阶段

cloud-init 脚本生成
--------------------

社区提供了 `write-mime-multipart` 工具来生成 cloud-init 脚本。
使用方法如下：

.. code-block:: bash

    $ cat my-boothook.txt
    #!/bin/sh
    echo "Hello World!"
    echo "This will run as soon as possible in the boot sequence"
    
    $ cat my-user-script.txt
    #!/usr/bin/perl
    print "This is a user script (rc.local)\n"
    
    $ cat my-include.txt
    # these urls will be read pulled in if they were part of user-data
    # comments are allowed.  The format is one url per line
    http://www.ubuntu.com/robots.txt
    http://www.w3schools.com/html/lastpage.htm
    
    $ cat my-upstart-job.txt
    description "a test upstart job"
    start on stopped rc RUNLEVEL=[2345]
    console output
    task
    script
    echo "====BEGIN======="
    echo "HELLO From an Upstart Job"
    echo "=====END========"
    end script
    
    $ cat my-cloudconfig.txt
    #cloud-config
    ssh_import_id: [smoser]
    apt_sources:
     - source: "ppa:smoser/ppa"

    $ write-mime-multipart --output=combined-userdata.txt \
        my-boothook.txt:text/cloud-boothook \
        my-include.txt:text/x-include-url \
        my-upstart-job.txt:text/upstart-job \
        my-user-script.txt:text/x-shellscript \
        my-cloudconfig.txt

User Data 输入格式
------------------

#. Gzip Compressed Content

#. Mime Multi Part archive

#. User-Data Script

   以 ``#!`` 或 ``Content-Type: text/x-shellscript`` 开头。
   
   存放用户脚本，可以是 python, shell, perl 等。``user data``
   里的脚本执行比较晚，类似 rc.local， 而且只在第一次启动时
   执行。

#. Include File

   以 ``#include`` 或 ``Content-Type: text/x-include-url`` 开头。

   ``includ file`` 的内容是一串 url, 每行一个，cloud-init 启动时会
   读取 url 链接的内容。

#. Cloud Config Data

   以 ``#cloud-config`` 或 ``Content-Type: text/cloud-config`` 开头，
   内容按规定格式编写。

#. Upstart Job

   以 ``#upstart-job`` 或 ``Content-Type: text/upstart-job`` 开头，

   文件内容会存放在 `/etc/init` ，以供其它 job 使用。

#. Cloud Boothook

   以 ``#cloud-boothook`` 或 ``Content-Type: text/cloud-boothook`` 开头。

   ``boothook`` 数据会保存到 `/var/lib/cloud` ，然后立刻执行。``boothook``
   每次上电都会执行，没有机制指定只运行一次。

#. Part Handler
