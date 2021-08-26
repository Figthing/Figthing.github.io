title: centos7升级后无法重启或关机解决办法
author: Figthing
tags:
  - linux
  - centos
categories:
  - linux
date: 2018-03-02 10:21:00
---
由于CentOS7新发布的yum源升级包中systemd关机流程判断条件发生了变化，可能会导致centos7升级后，服务器重启时卡死。新发布的systemd进程的判断更加严格，如果某些进程不响应SIGTERM信号，可能会导致重启是挂死。该问题和业务进程对SIGTERM信号的处理有关。

执行yum update systemd（或者yum update）将systemd系列软件包更新到219-19.el7版本之后，reboot会出现如下卡机界面导致系统挂住，无法重启： 

**现象：**

![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/linux/9.jpg)

<!--more-->

**查询版本包：**

![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/linux/10.jpg)

**解决办法：**

1. 新建/etc/systemd/system/rc-local.service并写入：

 ```shell
 [Unit]
 Description=/etc/rc.d/rc.local Compatibility
 ConditionFileIsExecutable=/etc/rc.d/rc.local
 After=network.target

 [Service]
 Type=forking
 ExecStart=/etc/rc.d/rc.local start
 TimeoutSec=5
 RemainAfterExit=yes
 ```
 
2. 备份/etc/systemd/system.conf
 ```shell
 cp -a /etc/systemd/system.conf /etc/systemd/system.conf_bak
 ```
 
3. 修改文件
 ```shell
 sed -i 's/#DefaultTimeoutStopSec=90s/DefaultTimeoutStopSec=30s/g' /etc/systemd/system.conf
 ```
 
4. 重新加载
 ```shell
 systemctl daemon-reload
 ```