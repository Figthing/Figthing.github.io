title: Docker集群(二)
author: Figthing
tags:
  - docker
categories:
  - docker
date: 2019-01-16 18:02:00
---
#### Docker使用Portainer管理Swarm集群

在上一节中已经将了，如果将子节点加入到主节点中，本节将使用portainer来对swarm集群进行管理

##### 安装portainer管理工具
下载portainer工具，并启用docker服务
```shell
[neusoft@localhost ~/docker]$ docker pull registry.cn-shenzhen.aliyuncs.com/zhouqi/portainer:1.20.0

[neusoft@localhost ~/docker]$ docker service create \
> --name portainer \
> --publish 9000:9000 \
> --constraint 'node.role == manager' \
> --mount type=bind,src=//var/run/docker.sock,dst=/var/run/docker.sock \
> registry.cn-shenzhen.aliyuncs.com/zhouqi/portainer:1.20.0 \
> -H unix:///var/run/docker.sock
```

<!--more-->

查看docker的服务，已经启用成功
```shell
[neusoft@localhost ~/docker]$ docker service ls
ID            NAME       MODE        REPLICAS  IMAGE
y6ydozgujzjo  portainer  replicated  1/1       registry.cn-shenzhen.aliyuncs.com/zhouqi/portainer:1.20.0
```
如果想停用该服务可以使用以下命令：
```shell
[neusoft@localhost ~/docker]$ docker service rm portainer
portainer
```

##### 查看portainer并进行集群设置
访问`http://172.24.2.63:9000`地址，首先进行设置密码
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/8.png)

登录进去后，能查看到首页的信息
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/9.png)

添加一个子节点到portainer中
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/10.png)

现在可以查看添加进去的节点信息了
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/11.png)