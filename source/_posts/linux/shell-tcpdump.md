title: Tcpdump命令
author: Figthing
tags:
  - linux
  - shell
  - tcpdump
categories:
  - linux
date: 2018-01-19 16:22:00
---
### 简介

用简单的话来定义tcpdump，就是：dump the traffic on a network，根据使用者的定义对网络上的数据包进行截获的包分析工具。 tcpdump可以将网络中传送的数据包的“头”完全截获下来提供分析。它支持针对网络层、协议、主机、网络或端口的过滤，并提供and、or、not等逻辑语句来帮助你去掉无用的信息。

### 参数说明

> 使用格式

 `
 $ tcpdump [ -DenNqvX ] [ -c count ] [ -F file ] [ -i interface ] [ -r file ] [ -s snaplen ] [ -wfile ] [ expression ]  
 `
 
> 抓包选项  
 
 ```shell
 -c：指定要抓取的包数量。注意，是最终要获取这么多个包。例如，指定"-c 10"将获取10个包，但可能已经 处理了100个包，
 只不过只有10个包是满足条件的包。 
 -i interface：指定tcpdump需要监听的接口。若未指定该选项，将从系统接口列表中搜寻编号最小的已配置
 好的接口(不包括loopback接口，要抓取loopback接口使用tcpdump -i lo)，一旦找到第一个符合条件的接
 口，搜寻马上结束。可以使用'any'关键字表示所有网络接口。  
 -n：对地址以数字方式显式，否则显式为主机名，也就是说-n选项不做主机名解析。
 -nn：除了-n的作用外，还把端口显示为数值，否则显示端口服务名。
 -N：不打印出host的域名部分。例如tcpdump将会打印'nic'而不是'nic.ddn.mil'。
 -P：指定要抓取的包是流入还是流出的包。可以给定的值为"in"、"out"和"inout"，默认为"inout"。
 -s len：设置tcpdump的数据包抓取长度为len，如果不设置默认将会是65535字节。对于要抓取的数据包较大
 时，长度设置不够可能会产生包截断，若出现包截断，输出行中会出现"[|proto]"的标志(proto实际会显示
 为协议名)。但是抓取len越长，包的处理时间越长，并且会减少tcpdump可缓存的数据包的数量，从而会导致
 数据包的丢失，所以在能抓取我们想要的包的前提下，抓取长度越小越好。
```

> 输出选项

 ```shell
 -e：输出的每行中都将包括数据链路层头部信息，例如源MAC和目标MAC。
 -q：快速打印输出。即打印很少的协议相关信息，从而输出行都比较简短。
 -X：输出包的头部数据，会以16进制和ASCII两种方式同时输出。
 -XX：输出包的头部数据，会以16进制和ASCII两种方式同时输出，更详细。
 -v：当分析和打印的时候，产生详细的输出。
 -vv：产生比-v更详细的输出。
 -vvv：产生比-vv更详细的输出。
 ```

> 其他功能性选项

```shell
-D：列出可用于抓包的接口。将会列出接口的数值编号和接口名，它们都可以用于"-i"后。
-F：从文件中读取抓包的表达式。若使用该选项，则命令行中给定的其他表达式都将失效。
-w：将抓包数据输出到文件中而不是标准输出。可以同时配合"-G time"选项使得输出文件每time秒就自动切
换到另一个文件。可通过"-r"选项载入这些文件以进行分析和打印。
-r：从给定的数据包文件中读取数据。使用"-"表示从标准输入中读取。
```

<!--more-->

### 示例

1. ** 监视指定网络接口的数据包 **

 ```shell
 $ tcpdump -i eth1
 ```
 如果不指定网卡，默认tcpdump只会监视第一个网络接口，如eth0。

2. ** 监视指定主机的数据包，例如所有进入或离开longshuai的数据包 **

 ```shell
 $ tcpdump host longshuai
 ```

3. ** 打印helios<-->hot或helios<-->ace之间通信的数据包 **

 ```shell
 $ tcpdump host helios and \( hot or ace \)
 ```

4. ** 打印ace与任何其他主机之间通信的IP数据包,但不包括与helios之间的数据包 **

 ```shell
 $ tcpdump ip host ace and not helios
 ```

5. ** 截获主机hostname发送的所有数据 **

 ```shell
 $ tcpdump src host hostname
 ```

6. ** 监视所有发送到主机hostname的数据包 **
 ```shell
 $ tcpdump dst host hostname
 ```
7. ** 监视指定主机和端口的数据包 **
 ```shell
 $ tcpdump tcp port 22 and host hostname
 ```
8. ** 对本机的udp 123端口进行监视(123为ntp的服务端口) **

 ```shell
 $ tcpdump udp port 123
 ```
9. ** 监视指定网络的数据包，如本机与192.168网段通信的数据包，"-c 10"表示只抓取10个包 **
 ```shell
 $ tcpdump -c 10 net 192.168
 ```
10. ** 打印所有通过网关snup的ftp数据包(注意,表达式被单引号括起来了,这可以防止shell对其中的括号进行错误解析) **
```shell
$ tcpdump 'gateway snup and (port ftp or ftp-data)'
```
11. ** 抓取ping包 **

 ```shell
$ tcpdump -c 5 -nn -i eth0 icmp 
```

 如果明确要抓取主机为192.168.100.70对本机的ping，则使用and操作符。  

 ``` shell
 $ tcpdump -c 5 -nn -i eth0 icmp and src 192.168.100.62
 ```
 注意不能直接写icmp src 192.168.100.70，因为icmp协议不支持直接应用host这个type。
 
12. ** 抓取到本机22端口包 **

 ```shell
 $ tcpdump -c 10 -nn -i eth0 tcp dst port 22  
 ```
13. ** 解析包数据 **

 ```shell
 $ tcpdump -c 2 -q -XX -vvv -nn -i eth0 tcp dst port 22
 ```