title: Linux设置全路径显示
author: Figthing
tags:
  - linux
categories:
  - linux
date: 2018-01-12 13:48:00
---
在Linux中，编辑`vi /etc/bashrc`文件，搜索`PS1="[\u@\h \W]`，将大写的W改为w

> 
修改前：
![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/linux/7.png)

> 
修改后:
![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/linux/6.png)

> 
效果显示:
![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/linux/8.png)