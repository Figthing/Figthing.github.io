title: Rsync + Sersync 实现文件实时同步
author: Figthing
tags:
  - software
  - rsync
categories:
  - software
  - rsync
date: 2020-07-21 15:48:00
---
## Rsync + Sersync 实现文件实时同步

### 概要

sersync主要用于服务器同步，web镜像等功能。基于boost1.43.0,inotify api,rsync command.开发。目前使用的比较多的同步解决方案是inotify-tools+rsync ，另外一个是google开源项目Openduckbill（依赖于inotify- tools），这两个都是基于脚本语言编写的。相比较上面两个项目，本项目优点是：



- sersync是使用c++编写，而且对linux系统文件系统产生的临时文件和重复的文件操作进行过滤，所以在结合rsync同步的时候，节省了运行时耗和网络资源。因此更快。
- sersync配置起来很简单，其中bin目录下已经有基本上静态编译的2进制文件，配合bin目录下的xml配置文件直接使用即可。
- 使用多线程进行同步，尤其在同步较大文件时，能够保证多个服务器实时保持同步状态。
- 有出错处理机制，通过失败队列对出错的文件重新同步，如果仍旧失败，则按设定时长对同步失败的文件重新同步。
- 自带crontab功能，只需在xml配置文件中开启，即可按要求隔一段时间整体同步一次。无需再额外配置crontab功能。
- 能够实现socket与http插件扩展。

<!--more-->


### 原理

步骤原理：

- 在同步服务器（Master）上开启sersync服务，sersync负载监控配置路径中的文件系统事件变化；
- 调用rsync命令把更新的文件同步到目标服务器（S1 和 S2）；
- 需要在主服务器配置sersync，在同步目标服务器配置rsync server（注意：是rsync服务）

同步原理：

- 用户实时的往sersync服务器（M）上写入更新文件数据；
- 此时需要在同步主服务器（M）上配置sersync服务；
- 在S1 和S2上开启rsync守护进程服务，以同步拉取来自sersync服务器（M）上的数据；



通过rsync的守护进程服务后可以发现，实际上sersync就是监控本地的数据写入或更新事件；然后，在调用rsync客户端的命令，将写入或更新事件对应的文件通过rsync推送到目标服务器（S1 和S2）



### 实战

服务器拓扑图：

| IP          | 说明           | 主机名 |
| ----------- | -------------- | ------ |
| 172.24.2.71 | 文件存储服务器 | node1  |
| 172.24.2.70 | 文件存储服务器 | node2  |
| 172.24.2.62 | 备份服务器     | backup |

node1与node2配置相同，下面仅配置node1，有配置不同的地址会指出



#### 安装部署rsync

进入`backup`服务器

```shell
## 创建备份文件夹
[root@backup ~]# mkdir -p /backup/nfs/71

## 安装rsync
[root@backup ~]# yum install rsync
```

##### 配置rsyncd.conf

```shell
[root@backup ~]# vi /etc/rsyncd/rsyncd.conf

port = 873
pid file = /etc/rsyncd/rsyncd.pid
motd file= /etc/rsyncd/welcome.msg
lock file = /etc/rsyncd/rsyncd.lock
log file = /etc/rsyncd/rsyncd.log
timeout = 900

#配置某些格式不压缩
dont compress   = *.gz *.tgz *.zip *.z *.Z *.rpm *.deb *.bz2

[71]
#服务器读写目录
path = /backup/nfs/71
use chroot = no
#只写选择，只让客户端到服务器上写入
write only = yes
#只读选择，只让客户端从服务器上读取文件
read only = no
max connections = 5
list = yes
uid = root
gid = root
# 账号密码存放位置
secrets file = /etc/rsyncd/rsyncd.secrets
# 允许客户端连接IP地址
hosts allow = 172.24.2.71
hosts deny = *
ignore errors = yes
transfer logging = yes
log format = %t %a %m %f %b
auth users = root
```

##### 配置rsyncd.secrets

```shell
[root@backup ~]# vi /etc/rsyncd/rsyncd.secrets

root:71

[root@backup ~]# chmod 600 /etc/rsyncd/rsyncd.secrets
```

##### 启动rsync

```shell
[root@backup ~]# rsync --daemon --config=/etc/rsyncd/rsyncd.conf
```

进入`node1`服务器

```shell
## 安装rsync
[root@node1 ~]# yum install rsync

## 配置密码
[root@node1 ~]# vi /etc/rsync.password
71

## 配置文件权限
[root@node1 ~]# chmod 600 /etc/rsync.password
```



#### 安装部署sersync

进入`node1`服务器，配置系统参数

```shell
## 查看系统是否支持
[root@node1 ~]# ll /proc/sys/fs/inotify

total 0
-rw-r--r-- 1 root root 0 Jul 21 10:47 max_queued_events
-rw-r--r-- 1 root root 0 Jul 21 11:18 max_user_instances
-rw-r--r-- 1 root root 0 Jul 21 10:47 max_user_watches

## 查看系统默认参数值
[root@node1 ~]# sysctl -a | grep max_queued_events
fs.inotify.max_queued_events = 16384

[root@node1 ~]# sysctl -a | grep max_user_instances
fs.inotify.max_user_instances = 128

[root@node1 ~]# sysctl -a | grep max_user_watches
fs.inotify.max_user_watches = 8192

## 修改参数，如果修改过以下参数则不用再次修改
[root@node1 ~]# echo "fs.inotify.max_queued_events = 99999999" >> /etc/sysctl.conf
[root@node1 ~]# echo "fs.inotify.max_user_watches = 99999999" >> /etc/sysctl.conf
[root@node1 ~]# echo "fs.inotify.max_user_instances = 65535" >> /etc/sysctl.conf
[root@node1 ~]# sysctl -p
```

`inotify`参数说明：

- max_queued_events：inotify队列最大长度，如果值太小，会出现"** Event Queue Overflow **"错误，导致监控文件不准确
- max_user_watches：要同步的文件包含多少目录，可以用：find /home/wwwroot/ -type d | wc -l 统计，必须保证max_user_watches值大于统计结果（这里/home/wwwroot/为同步文件目录）
- max_user_instances：每个用户创建inotify实例最大值

##### 下载sersync

```shell
[root@node1 ~]# wget --no-check-certificate https://raw.githubusercontent.com/orangle/sersync/master/release/sersync2.5.4_64bit_binary_stable_final.tar.gz
```

##### 配置sersync

```shell
[root@node1 ~]# tar xf sersync2.5.4_64bit_binary_stable_final.tar.gz 
[root@node1 ~]# mv GNU-Linux-x86/ /usr/local/sersync
[root@node1 ~]# cd /usr/local/sersync/
[root@node1 ~]# cp confxml.xml confxml.xml_bak
```

修改配置文件如果有多个同步模块，则按下面格式依次去写，仅更改

```shell
[root@node1 ~]# vi /usr/local/sersync/confxml.xml

<localpath watch="/nfs/data/">			## 需要同步的目录
	<remote ip="172.24.2.62" name="71"/>	## 备份服务器IP
</localpath>

## rsync认证的账号和密码，start-true打开校验
<auth start="true" users="root" passwordfile="/etc/rsync.password"/>  

## 错误日志存放位置和触发时间60秒
<failLog path="/usr/local/sersync/logs/rsync_fail_log.sh" timeToExecute="60"/>

## 创建日志目录
[root@node1 ~]#  mkdir -p /usr/local/sersync/logs/
[root@node1 ~]#  touch /usr/local/sersync/logs/rsync_fail_log.sh
```

##### 启动sersync

```shell
[root@node1 ~]#  echo "PATH=$PATH:/usr/local/sersync/" >> /etc/profile
[root@node1 ~]#  source /etc/profile
[root@node1 ~]#  /usr/local/sersync/sersync2 -d -r -o /usr/local/sersync/confxml.xml
```

