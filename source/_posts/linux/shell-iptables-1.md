title: 'iptables之nat表应用——IP与端口的映射 '
author: Figthing
tags:
  - linux
  - shell
  - iptables
categories:
  - linux
date: 2018-01-22 09:36:00
---
1. ** 需求 **

 将192.168.3.195：80 映射到192.168.3.193：80，即访问192.168.3.195：80，得到192.168.3.193：80的结果，实现linux的路由。

2. ** 实现 **

 ```shell
 #!/bin/bash
 
 #打开转发功能
 echo "1" > /proc/sys/net/ipv4/ip_forward
          
 /sbin/iptables -F -t filter

 #清空iptables
 /sbin/iptables -F -t nat
     
 #去192.168.3.193的一条路
 /sbin/iptables -t nat -A PREROUTING -d 192.168.3.195 -p tcp --dport 80 -j DNAT --to-destination 192.168.3.193:80    

 #返回的时候的一条路，ip传输要有去有回才能连通 
 /sbin/iptables -t nat -A POSTROUTING  -s 192.168.3.0/24 -o eth0 -j SNAT --to 192.168.3.195  

 #用2网段访问的时候的回路
 /sbin/iptables -t nat -A POSTROUTING -s 192.168.2.0/24 -o eth0 -j SNAT --to 192.168.3.195  
 
 #用0网段访问的时候的回路
 /sbin/iptables -t nat -A POSTROUTING -s 192.168.0.0/24 -o eth0 -j SNAT --to 192.168.3.195  
 ```

 回路的那一条，或者只用下面这一句：
 `iptables -t nat -A POSTROUTING -s 0.0.0.0/0 -o eth0 -j SNAT --to 192.168.3.195 `或者
 `iptables -t nat -A POSTROUTING -o eth0 -j SNAT --to 192.168.3.195`把193上的80端口打开
 
3. ** 测试 **  
 在192.168.3.195上访问 192.168.3.195：80看到 195的apache主页，因为在本机上访问，没有走PREROUTING这条链。在其他主机上访问192.168.3.195：80 返回193的apache主页，路线为PREROUTING-->FORWARD-->PSOTROUTING.