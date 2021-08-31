title: K8s搭建-CoreDns
author: Figthing
tags:
  - docker
  - k8s
categories:
  - docker
  - k8s
date: 2019-06-28 16:21:00
---
## CoreDNS简介

CoreDNS 其实就是一个 DNS 服务，而 DNS 作为一种常见的服务发现手段，所以很多开源项目以及工程师都会使用 CoreDNS 为集群提供服务发现的功能，Kubernetes 就在集群中使用 CoreDNS 解决服务发现的问题。

如果想要在分布式系统实现服务发现的功能，CoreDNS 其实是一个非常好的选择，CoreDNS作为一个已经进入CNCF并且在Kubernetes中作为DNS服务使用的应用，其本身的稳定性和可用性已经得到了证明，同时它基于插件实现的方式非常轻量并且易于使用，插件链的使用也使得第三方插件的定义变得非常的方便。

### Coredns 架构
整个 CoreDNS 服务都建立在一个使用 Go 编写的 HTTP/2 Web 服务器 Caddy 。

### Coredns 项目下载
下载地址1：

wget https://github.com/coredns/deployment/archive/master.zip
unzip master.zip

下载地址2：
git clone https://github.com/coredns/deployment.git

<!--more-->

### 安装部署
确认是否存在已运行dns服务

```shell
kubectl  get pods -o wide -n=kube-system
```

删除命令
```shell
kubectl delete --namespace=kube-system deployment ****-dns
```

生成安装配置文件
```shell
cd /workspace/deployment/kubernetes
./deploy.sh -r 10.254.0.0/16 -i 10.254.0.2  -d cluster.local -t coredns.yaml.sed -s >coredns.yaml
```

验证配置文件核心配置
```shell
cat coredns.yaml
```

![1561709072968](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561709072968.png)



![1561709098483](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561709098483.png)

执行安装

```shell
kubectl create -f coredns.yaml
```

验证安装

```shell
kubectl get svc -o wide -n=kube-system

NAME                   TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)               
heapster               ClusterIP   10.109.196.102   <none>        80/TCP                
kube-dns               ClusterIP   10.96.0.2        <none>        53/UDP,53/TCP,9153/TCP
kubernetes-dashboard   NodePort    10.110.231.202   <none>        443:30001/TCP         
monitoring-grafana     NodePort    10.98.242.145    <none>        80:30108/TCP          
monitoring-influxdb    ClusterIP   10.100.60.54     <none>        8086/TCP              
```

查看coredns详细

![1561709283565](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561709283565.png)

### 设置master和节点DNS

使用命令查看kubelet配置的位置

```shell
systemctl status kubelet -l
```

![1561709534383](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561709534383.png)

修改`/var/lib/kubelet/config.yaml`文件的内容

```shell
clusterDNS:
- 10.96.0.2
clusterDomain: cluster.local.
```

重启

```shell
systemctl daemon-reload
systemctl restart kubelet
```

### 验证DNS

1、创建一个curl

```shell
kubectl run -it --image=registry.cn-shenzhen.aliyuncs.com/zhouqi-kubernetes/busyboxplus:v1 curl --rm
```

![1561706009470](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561706009470.png)

这个地方如果要验证各节点，可以伸缩多个，会运行在不同的节点中

![1561706093086](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561706093086.png)

对节点的验证，只需要进入容器使用`nslookup kubernetes`命令，就可以显示下图

![1561706175310](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/1561706175310.png)



文献参考：<https://blog.51cto.com/michaelkang/2367800>