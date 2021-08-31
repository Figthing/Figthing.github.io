title: 搭建Shadowsocks服务
author: Figthing
tags:
  - software
  - ss
categories:
  - software
  - ss
date: 2019-10-13 23:49:00
---
### 搭建Shadowsocks服务

#### 概念

 shadowsocks可以指一种SOCKS5的加密传输协议，也可以指基于这种加密协议的各种数据传输包。 

**shadowsocks实现科学上网原理？**shadowsocks正常工作需要服务器端和客户端两端合作实现，首先，客户端（本机）通过ss（shadowsocks）对正常的访问请求进行SOCK5加密，将加密后的访问请求传输给ss服务器端，服务器端接收到客户端的加密请求后，解密得到原始的访问请求，根据请求内容访问指定的网站（例如Google，YouTube，Facebook，instagram等），得到网站的返回结果后，再利用SOCKS5加密并返回给客户端，客户端通过ss解密后得到正常的访问结果，于是就可以实现你直接访问该网站的“假象”。

**为什么选择shadowsocks？**不限终端（安卓，苹果，Windows，Mac都可用），流量便宜（服务器500G只要15元），方便（一键脚本，不需要专业知识）。

<!-- more -->

**为什么要自己搭建ss/ssr？**你也许会觉得买ss服务也很方便，但是你得要考虑以下几个问题。首先，买的ss服务，限制很多，终端可能只能同时在线2个，每个月就一点点流量可能价格却不便宜，有时候还被别人做手脚，流量跑的贼快；其次，别人收钱跑路怎么办？很多这种情况的；更重要的是，如第一个问题中描述的shadowsocks原理，如果有心人做了一点手脚，是可以得到你的访问记录的；而自己搭建ss/ssr服务，一键脚本也就10来分钟就可以搞定。



#### 代理服务器购买

作为跳板的代理服务器推荐Vultr和搬瓦工，一是因为本脚本在这两家的所有VPS都做了测试，二是因为都是老牌VPS服务商，不怕跑路。

Vultr和搬瓦工比较：

1. Vultr月付，3.5美元/月起步，搬瓦工年付，年付46.87美元起步；
2. 搬瓦工线路好，提供CN2 GT/CN2 GIA多条线路，并且保证开到的IP绝对可用；

如果你是长期使用，那么推荐你使用搬瓦工，线路比Vultr要好的，尤其是联通和电信用户，所谓线路好就是速度快的意思。

搬瓦工提供CN2和CN2 GIA多种优化线路：

1. 速度自然是CN2 GIA > CN2 > KVM，相应的价格最贵的是CN2 GIA（包括CN2 GIA-E），但是CN2和KVM的价格却一样，所以如果不买CN2 GIA，则买CN2，不要考虑KVM；
2. CN2 GIA的意思是三网CN2直连的意思，优势就是晚高峰也快，价格是最便宜的119.99美元/年，贵但是值。

对于搬瓦工的推荐：

1. 不是很在乎120美元/年和50美元/年之间的差距，一年也就差500元其实，那么直接选择CN2 GIA-E，用过的都说好；
2. 如果不是很宽裕，则选择50美元/年的搬瓦工CN2。

 搬瓦工购买与优惠码使用： https://www.vultr.com/?ref=8289861



#### 一键搭建Shadowsocks

 连上购买的VPS后（以Xshell为例），你将看到如下图所示的界面 

![1](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/software/ss/1.png)

 如红框中所示，root@vult（root@ubuntu）说明已经连接成功了，之后你只需要**在绿色光标处直接复制以下代码并回车**就可以了（直接复制即可，如每段代码下方截图中所示）。 



1、安装git

```shell
yum -y install git
```

2、将代码clone到本地

```shell
git clone -b master https://github.com/flyzy2005/ss-fly
```

3、运行搭建ss脚本代码 

```shell
ss-fly/ss-fly.sh -i 密码 端口号
```

界面如下就表示一键搭建ss成功了：

![2](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/software/ss/2.png)

4、 相关ss操作 

```shel
修改配置文件：vim /etc/shadowsocks.json
停止ss服务：ssserver -c /etc/shadowsocks.json -d stop
启动ss服务：ssserver -c /etc/shadowsocks.json -d start
重启ss服务：ssserver -c /etc/shadowsocks.json -d restart
```

5、 卸载ss服务 

```shell
sss -fly/ss-fly.sh -uninstall
```