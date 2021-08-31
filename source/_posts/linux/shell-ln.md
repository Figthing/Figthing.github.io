title: LN命令
author: Figthing
tags:
  - linux
  - shell
  - ln
categories:
  - linux
date: 2018-01-09 17:15:00
---
### 简介

ln命令用来为文件创件连接，连接类型分为硬连接和符号连接两种，默认的连接类型是硬连接。如果要创建符号连接必须使用`-s`选项。

注意：符号链接文件不是一个独立的文件，它的许多属性依赖于源文件，所以给符号链接文件设置存取权限是没有意义的。

### 参数说明

```shell
-b: 删除，覆盖以前建立的链接
-d: 允许超级用户制作目录的硬链接
-f: 强制执行
-i: 交互模式，文件存在则提示用户是否覆盖
-n: 把符号链接视为一般目录
-s: 软链接(符号链接)
-v: 显示详细的处理过程
-S: “-S<字尾备份字符串> ”或 “--suffix=<字尾备份字符串>”
-V: “-V<备份方式>”或“--version-control=<备份方式>”
--help: 显示帮助信息
--version: 显示版本信息
```

<!--more-->

### 示例
#### 建立一个符号链接

> 
在目录`/usr/liu`下建立一个符号链接文件`abc`，使它指向目录`/usr/mengqc/mub1`

```shell
$ ln -s /usr/mengqc/mub1 /usr/liu/abc
```

#### 删除一个符号链接

> 将目录`/usr/liu/`下的`abc`链接删除，注意不是`rm -rf symbolic_name/ `

```shell
$ cd /usr/liu/
$ rm -rf symbolic_name 
```