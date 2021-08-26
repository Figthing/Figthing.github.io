title: K8s-Dashboard安装
author: Figthing
tags:
  - docker
  - k8s
categories:
  - docker
  - k8s
date: 2019-06-18 20:43:00
---
## 安装Kubernetes-dashboard

### 概述

Kubernetes Dashboard是Kubernetes集群的基于Web的通用UI。它允许用户管理在群集中运行的应用程序并对其进行故障排除，以及管理群集本身。



### 创建

创建Dashboard的yaml文件，并设置端口为30001，拉取镜像文件地址

```shell
wget https://raw.githubusercontent.com/kubernetes/dashboard/v1.10.1/src/deploy/recommended/kubernetes-dashboard.yaml
sed -i 's/k8s.gcr.io/loveone/g' kubernetes-dashboard.yaml
sed -i '/targetPort:/a\ \ \ \ \ \ nodePort: 30001\n\ \ type: NodePort' kubernetes-dashboard.yaml
```



### 证书

#### 生成自签证书

1）生成证书请求的key

> openssl genrsa -out dashboard.key 2048



2）生成证书请求

```shell
openssl req -days 3650 -new -out dashboard.csr -key dashboard.key -subj '/CN=**172.24.2.69**'
```



3）生成自签证书

> openssl x509 -req -in dashboard.csr -signkey dashboard.key -out dashboard.crt

**以上都是在服务器上执行**



<!--more-->



#### 创建Secret

1）删除之前部署的Dashboard

> kubectl delete -f kubernetes-dashboard.yaml



2) 创建与KubernetesDashboard 部署文件中同名的secret

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/20200507145738.png?" style="zoom:80%;" />

```shell
kubectl create secret generic kubernetes-dashboard-certs --from-file=./dashboard.key --from-file=./dashboard.crt -n kube-system
```

**注意**

- 命名空间：`kube-system`

- 删除证书：`kubectl delete secret kubernetes-dashboard-certs`



3)注释 kubernetes-dashboard.yaml文件中关于Dashboard Secret部分

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/docker/k8s/20200507150647.png?" style="zoom:80%;" />



### 安装

1、部署Dashboard

```shell
kubectl create -f kubernetes-dashboard.yaml
```



2、创建完成后，检查相关服务运行状态

```shell
kubectl get deployment kubernetes-dashboard -n kube-system

kubectl get pods -n kube-system -o wide

kubectl get services -n kube-system

kubectl describe pod kubernetes-dashboard-fd674d9c5-dp5k6 --namespace=kube-system

netstat -ntlp|grep 30001
```



3、查看访问Dashboard的认证令牌

```shell
kubectl create serviceaccount  dashboard-admin -n kube-system
kubectl create clusterrolebinding  dashboard-admin --clusterrole=cluster-admin --serviceaccount=kube-system:dashboard-admin
kubectl describe secrets -n kube-system $(kubectl -n kube-system get secret | awk '/dashboard-admin/{print $1}')
```



### 问题集合

- 问题：Chrome无法访问

>  解决：chrome.exe" 在快捷方式中增加  --disable-infobars --ignore-certificate-errors

- 问题：Master的Token过期

> 使用命令：kubeadm token create重新生成Token在加入

- 问题：子节点加入后出现NotReady

> 在master中重启docker，方可解决