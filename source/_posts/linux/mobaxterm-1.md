title: Mobaxterm session stopped
author: Figthing
tags:
  - linux
  - mobaxterm
categories: []
date: 2018-05-29 10:05:00
---
#### Mobaxterm Session Stopped

1. 点击当前的Session
2. 编辑Edit Session
3. 点击Telnet设置Remote host地址为你的SSH地址
4. 在`Advanced Telnet settings`选项卡的`Telnet Client`中选择`Busybox telnet`
5. 尝试连接（肯定没效果）
6. 切换到SSH，重新输入Remote host，点击连接，输入账号密码，ok连上了！