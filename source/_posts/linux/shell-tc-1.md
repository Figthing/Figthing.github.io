title: TC基于CBQ队列的流量管理范例
author: Figthing
tags:
  - linux
  - shell
  - tc
categories:
  - linux
date: 2018-01-22 09:57:00
---
### 简介

参考了TC的很多文档，自己也整理了一篇配置记录。在实际使用过程中效果还不错，在此分享给大家以备参考。
环境：局域网规模不是很大40多台机器。 NAT共享上网（内网：eth0 外网：eth2）
CBQ是通过硬件的闲置时间来计算队列，硬件不同，效果也不同，对于比较大的网络使用HTB比较好。以下限制上传和下载的方法可以写成脚本，通过mrtg发现流量的异常情况，然后通过ntop查处是谁在干坏事，最后用写好的tc脚本限制他的流量，避免影响其他人的网络使用。

 <!--more-->
 
### 示例

#### 针对网络物理设备绑定一个CBQ队列

 ```shell
 $ tc qdisc add dev eth0 root handle 1: cbq bandwidth 10Mbit avpkt 1000 cell 8 mpu 64
 ```

 将一个cbq队列绑定到网络物理设备eth0上，其编号为1:0；网络物理设备eth0的实际带宽为10Mbit，包的平均大小为1000字节；包间隔发送单元的大小为8字节，最小传输包大小为64字节。
 
#### 在该队列上建立分类

 ```shell
 $ tc class add dev eth0 parent 1:0 classid 1:1 cbq bandwidth 10Mbit rate 10Mbit maxburst 20 allot 1514 prio 1 avpkt 1000 cell 8 weight 1Mbit
 ```

 创建根分类1:1；分配带宽为10Mbit，优先级别为1。该队列的最大可用带宽为10Mbit，实际分配的带宽为10Mbit，可接收冲突的发送最长包数目为20字节；最大传输单元加MAC头的大小为1514字节，优先级别为1，包的平均大小为1000字节，包间隔发送单元的大小为8字节，相应于实际带宽的加权速率为1Mbit。

 > ** 创建子分类 **
 - 创建分类1:2，其父分类为1:1，分配带宽为64Kbit，优先级别为8。该队列的最大可用带宽为10Mbit，实际分配的带宽为64Kbit，可接收冲突的发送最长包数目为20字节；最大传输单元加MAC头的大小为1514字节，优先级别为8，包的平均大小为1000字节，包间隔发送单元的大小为8字节，相应于实际带宽的加权速率为100Kbit，且不可借用未使用带宽。
 ` $ tc class add dev eth0 parent 1:1 classid 1:2 cbq bandwidth 10Mbit rate 64Kbit maxburst 20 allot 1514 prio 8 avpkt 1000 cell 8 weight 100Kbit bounded `
 - 创建分类1:3，其父分类为1:1，分配带宽为64Kbit，优先级别为9。该队列的最大可用带宽为10Mbit，实际分配的带宽为64Kbit，可接收冲突的发送最长包数目为20字节；最大传输单元加MAC头的大小为1514字节，优先级别为9，包的平均大小为1000字节，包间隔发送单元的大小为8字节，相应于实际带宽的加权速率为100Kbit，且不可借用未使用带宽。
 `$ tc class add dev eth0 parent 1:1 classid 1:3 cbq bandwidth 10Mbit rate 64Kbit maxburst 20 allot 1514 prio 9 avpkt 1000 cell 8 weight 100Kbit bounded`
 
#### 在子分类地下创建队列，使用sfq随机公平队列 

 ```shell
 $ tc qdisc add dev eth0 parent 1:2 sfq quantum 1514b perturb 15
 $ tc qdisc add dev eth0 parent 1:3 sfq quantum 1514b perturb 15
 ```
 在分类底下，创建队列，使用sfq随即公平队列
 
#### 为每一分类建立一个基于路由的过滤
 
 ```shell
 $ tc filter add dev eth0 parent 1:0 protocol ip prio 1 u32 match ip dst 192.111.1.116 flowid 1:2
 $ tc filter add dev eth0 parent 1:0 protocol ip prio 1 u32 match ip dst 192.111.1.66 flowid 1:3
 ```
 限制各ip地址的下载带宽，使用u32过滤器，对目的地址进行分类，对应已经创建的队列需要添加新的被限制ip的下载带宽，需要先要创建新的分类(比如1:4),然后根据新的分类创建新的sfq队列，最后使用u32过滤器对目的地址进行带宽限制。需要对几个ip限制下载带宽，就需要创建几个分类、队列、过滤器
 
#### 限制上传

 将一个cbq队列绑定到网络物理设备eth2上，其编号为2:0；网络物理设备eth2的实际带宽为2Mbit，包的平均大小为1000字节；包间隔发送单元的大小为8字节，最小传输包大小为64字节。
 ```shell
 $ tc qdisc add dev eth2 root handle 2: cbq bandwidth 2Mbit avpkt 1000 cell 8 mpu 64
 ```
 创建根分类2:1；分配带宽为2Mbit，优先级别为1。该队列的最大可用带宽为2Mbit，实际分配的带宽2Mbit，
 可接收冲突的发送最长包数目为20字节；最大传输单元加MAC头的大小为1514字节，优先级别为1，包的平均
 大小为1000字节，包间隔发送单元的大小为8字节，相应于实际带宽的加权速率为200Kbit。
 
 ```shell
 $ tc class add dev eth2 parent 2:0 classid 2:1 cbq bandwidth 2Mbit rate 2Mbit maxburst 20 allot 1514 prio 1 avpkt 1000 cell 8 weight 200Kbit
 ```
 创建分类2:2，其父分类为2:1，分配带宽为64Kbit，优先级别为8。该队列的最大可用带宽为2Mbit，实际分配的带宽为64Kbit，可接收冲突的发送最长包数目为20字节；最大传输单元加MAC头的大小为1514字节，优先级别为8，包的平均大小为1000字节，包间隔发送单元的大小为8字节，相应于实际带宽的加权速率为100Kbit，且不可借用未使用带宽。
 ```shell
 $ tc class add dev eth2 parent 2:1 classid 2:2 cbq bandwidth 2Mbit rate 64Kbit maxburst 20 allot 1514 prio 8 avpkt 1000 cell 8 weight 200Kbit bounded
 ```

 在分类底下，创建队列，使用sfq随即公平队列
 ```shell
 $ tc qdisc add dev eth2 parent 2:2 sfq quantum 1514b perturb 15
 ```

 应用路由分类器到cbq队列的根，过滤协议为ip，优先级为100
 ```shell
 $ tc filter add dev eth2 parent 2:0 protocol ip prio 1 handle 2 fw classid 2:2
 ```

 给数据包打标签,可以通过RETURN方法避免遍历所有的规则，加快处理速度
 ```shell
 $ iptables –t mangle –A PREROUTING –i eth0 –s 192.111.1.xxx –j MARK --set-mark 2
 $ iptables –t mangle –A PREROUTING –i eth0 –s 192.111.1.xxx –j RETURN 
 ```
 nat模式
 ```shell
 $ iptables -t nat -A POSTROUTING -s 192.111.1.0/24 -o eth2 -j SNAT --to 外网IP
 ```
 > 需要添加新的被限制ip的上传带宽，需要先要创建新的分类(比如2:3),然后根据新的分类创建新的sfq队列，最后使用路由过滤器，过滤协议为ip，给原地址是需要限制的ip地址来的数据包打标记。
需要对几个ip限制下载带宽，就需要创建几个分类、队列、路由过滤器、iptable的mangle表的PREROUTING链

 另外还有其他的过滤器比如
 ```shell
 $ tc filter add dev eth0 parent 1:0 protocol ip prio 100 route to 2 flowid 1:2 ip route add 192.111.1.24 dev eth0 via 192.111.1.4 realm 2
 ```
#### 维护

 主要包括对队列、分类、过滤器和路由的增添、修改和删除。 增添动作一般依照"队列->分类->过滤器->路由"的顺序进行；修改动作则没有什么要求；删除则依照"路由->过滤器->分类->队列"的顺序进行。

 简单显示指定设备的队列状况
 ```shell
 $ tc qdisc ls dev eth0
 ```

 详细显示指定设备的队列状况
 ```shell
 $ tc –s qdisc ls dev eth0
 ```

 简单显示指定设备的分类状况
 ```shell
 $ tc class ls dev eth0
 ```

 详细显示指定设备的分类状况
 ```shell
 $ tc –s class ls dev eth0
 ```

 显示过滤器的状况
 ```shell
 $ tc –s filter ls dev eth0
 ```

 队列的维护 
 一般对于一台流量控制器来说，出厂时针对每个以太网卡均已配置好一个队列了，通常情况下对队列无需进行增添、修改和删除动作了。

#### 分类的维护

 增添动作通过`tc class add`命令实现。

 修改动作通过`tc class change`命令实现，如下所示：
 ```shell
 $ tc class change dev eth0 parent 1:1 classid 1:2 cbq bandwidth 10Mbit rate 64Kbit maxburst 20 allot 1514 prio 8 avpkt 1000 cell 8 weight 100Kbit bounded
 ```
 对于bounded命令应慎用，一旦添加后就进行修改，只可通过删除后再添加来实现。

#### 过滤器的维护

 增添动作通过`tc filter add`命令实现。
 
 修改动作通过`tc filter change`命令实现，如下所示：
 ```shell
 $ tc filter change dev eth0 parent 1:0 protocol ip prio 1 u32 match ip dst 192.111.1.116 flowid 1:2
 ```

 删除动作通过`tc filter del`命令实现，如下所示：
 ```shell
 $ tc filter del dev eth0 parent 1:0 protocol ip prio 1 u32 match ip dst 192.111.1.116 flowid 1:
 ```













 
 
 
 
 