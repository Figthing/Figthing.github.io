title: DF命令
author: Figthing
tags:
  - linux
  - shell
  - df
categories:
  - linux
date: 2018-01-12 13:36:00
---
### 简介

df命令用于显示磁盘分区上的可使用的磁盘空间。默认显示单位为KB。可以利用该命令来获取硬盘被占用了多少空间，目前还剩下多少空间等信息。

### 参数说明

```shell
-a或--all：包含全部的文件系统；
--block-size=<区块大小>：以指定的区块大小来显示区块数目；
-h或--human-readable：以可读性较高的方式来显示信息；
-H或--si：与-h参数相同，但在计算时是以1000 Bytes为换算单位而非1024 Bytes；
-i或--inodes：显示inode的信息；
-k或--kilobytes：指定区块大小为1024字节；
-l或--local：仅显示本地端的文件系统；
-m或--megabytes：指定区块大小为1048576字节；
--no-sync：在取得磁盘使用信息前，不要执行sync指令，此为预设值；
-P或--portability：使用POSIX的输出格式；
--sync：在取得磁盘使用信息前，先执行sync指令；
-t<文件系统类型>或--type=<文件系统类型>：仅显示指定文件系统类型的磁盘信息；
-T或--print-type：显示文件系统的类型；
-x<文件系统类型>或--exclude-type=<文件系统类型>：不要显示指定文件系统类型的磁盘信息；
--help：显示帮助；
--version：显示版本信息。
```

<!--more-->

### 示例

#### 查看系统磁盘设备，默认是KB为单位

```shell
$ df
```

> 结果显示：
![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/linux/4.png)

> 
字段说明：
Filesystem:		文件系统
1K-blocks:		1K-块        
Used:			已用     
Available:		可用 
Use%:			已用% 
Mounted on:		挂载点


#### 使用-h选项以KB以上的单位来显示，可读性高

```shell
$ df -h
```

> 结果显示：
![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/linux/5.png)