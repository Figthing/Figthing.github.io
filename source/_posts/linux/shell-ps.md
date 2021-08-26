title: PS命令
author: Figthing
tags:
  - linux
  - shell
  - ps
categories:
  - linux
copyright: true
date: 2018-01-07 23:07:00
updated: 2018-01-08 13:18:54
---
### 简介

要对进程进行监测和控制,首先必须要了解当前进程的情况,也就是需要查看当前进程,而ps命令就是最基本同时也是非常强大的进程查看命令.使用该命令可以确定有哪些进程正在运行和运行的状态、进程是否结束、进程有没有僵尸、哪些进程占用了过多的资源等等.总之大部分信息都是可以通过执行该命令得到的.

### 参数说明

``` shell
-a：显示现行终端机下的所有进程，包括其他用户的进程
-A：所有的进程均显示出来，与 -e 具有同样的效用
-u：以用户为主的进程状态
-f：做一个更为完整的输出
-e：等于“-A”
x： 通常与 a 这个参数一起使用，可列出较完整信息
l： 较长、较详细的将该 PID 的的信息列出
j： 工作的格式 (jobs format)
```

<!--more-->

### PS L字段说明


![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/linux/1.png)

> 
F	：代表这个程序的旗标 (flag)， 4 代表使用者为 super user
S	：代表这个程序的状态 (STAT)，关于各 STAT 的意义将在内文介绍
UID ：程序被该 UID 所拥有
PID ：就是这个程序的 ID ！
PPID ：则是其上级父程序的ID
C	：CPU 使用的资源百分比
PRI ：这个是 Priority (优先执行序) 的缩写，详细后面介绍
NI	：这个是 Nice 值，在下一小节我们会持续介绍
ADDR ：这个是 kernel function，指出该程序在内存的那个部分。如果是个 running的程序，一般就是 "-"
SZ	：使用掉的内存大小
WCHAN ：目前这个程序是否正在运作当中，若为 - 表示正在运作
TTY ：登入者的终端机位置
TIME ：使用掉的 CPU 时间。
CMD ：所下达的指令为何

### PS AUX字段说明

![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/linux/2.png)

> 
USER：该进程属于那个使用者账号的？
PID ：该进程的进程ID号。
%CPU：该进程使用掉的 CPU 资源百分比；
%MEM：该进程所占用的物理内存百分比；
VSZ ：该进程使用掉的虚拟内存量 (Kbytes)
RSS ：该进程占用的固定的内存量 (Kbytes)
TTY ：该进程是在那个终端机上面运作，若与终端机无关，则显示 ?，另外， tty1-tty6 是本机上面的登入者程序，若为 pts/0 等等的，则表示为由网络连接进主机的程序。
START：该进程被触发启动的时间；
TIME ：该进程实际使用 CPU 运作的时间。
COMMAND：该程序的实际指令为什么？
 >> STAT：该程序目前的状态，主要的状态有：
   R ：该程序目前正在运作，或者是可被运作；
   S ：该程序目前正在睡眠当中 (可说是 idle 状态啦！)，但可被某些讯号(signal) 唤醒。
   T ：该程序目前正在侦测或者是停止了；
   Z ：该程序应该已经终止，但是其父程序却无法正常的终止他，造成 zombie (疆尸) 程序的状态
   
   
### 示例

#### 显示所有进程信息

```shell
$ ps -A
```

#### 显示指定用户信息

```shell
$ ps -u root
```
#### 显示所有进程信息，连同命令行

```shell
$ ps -ef
```

#### ps 与grep 常用组合用法，查找特定进程

```shell
$ ps -ef|grep ssh
```

#### 列出类似程序树的程序显示

```shell
$ ps -axjf
```

#### 找出与 cron 与 syslog 这两个服务有关的 PID 号码

```shell
$ ps aux | egrep '(cron|syslog)'
```