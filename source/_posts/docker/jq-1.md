title: Docker集群(一)
author: Figthing
tags:
  - docker
categories:
  - docker
date: 2019-01-16 17:41:00
---
#### Docker使用Swarm搭建集群

##### selinux
查看selinux是否关闭
```shell
[root@localhost ~]# /usr/sbin/sestatus -v
SELinux status:                 disabled
```
如果没有关闭，可以直接修改`/etc/selinux/config`中的`SELINUX`为`disabled`：
```shell
[neusoft@localhost ~/docker]$ vi /etc/selinux/config
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
#SELINUX=enforcing
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected. 
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

**注意：修改xelinux后需重启服务器**

打开docker.service中的2375端口
```shell
[neusoft@localhost ~]$ sudo vi /usr/lib/systemd/system/docker.service

[Unit]
Description=Docker Application Container Engine
Documentation=http://docs.docker.com
After=network.target rhel-push-plugin.socket registries.service
Wants=docker-storage-setup.service
Requires=docker-cleanup.timer

[Service]
Type=notify
NotifyAccess=all
EnvironmentFile=-/run/containers/registries.conf
EnvironmentFile=-/etc/sysconfig/docker
EnvironmentFile=-/etc/sysconfig/docker-storage
EnvironmentFile=-/etc/sysconfig/docker-network
Environment=GOTRACEBACK=crash
Environment=DOCKER_HTTP_HOST_COMPAT=1
Environment=PATH=/usr/libexec/docker:/usr/bin:/usr/sbin
ExecStart=/usr/bin/dockerd-current \
          --add-runtime docker-runc=/usr/libexec/docker/docker-runc-current \
          --default-runtime=docker-runc \
          --exec-opt native.cgroupdriver=systemd \
          --userland-proxy-path=/usr/libexec/docker/docker-proxy-current \
          --init-path=/usr/libexec/docker/docker-init-current \
          --seccomp-profile=/etc/docker/seccomp.json \
          $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $ADD_REGISTRY \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY \
          $REGISTRIES \
          -H unix:///var/run/docker.sock -H 0.0.0.0:2375
```

重启docker

<!--more-->

##### 创建swarm管理节点
下面开始创建swarm。登录到centos7主机上，执行如下命令：
```shell
[neusoft@localhost ~/docker]$ docker swarm init --advertise-addr 172.24.2.63
Swarm initialized: current node (5levgh6t0qizmgwyn789dcuqx) is now a manager.

To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-5g3hxk4q7eg3atnuw45ijrqei21y2aopiqha31v04jwmnz5svt-bsudrrah0zgdx7qzjcm6xvfy3 \
    172.24.2.63:2377

To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

如果你不知道或者忘记了Swarm Manager节点的Token信息， 你可以在Manager节点上执行以下命令查看Worker节点连接所需要的Token信息。
```shell
[neusoft@localhost ~/docker]$ docker swarm join-token worker
To add a worker to this swarm, run the following command:

    docker swarm join \
    --token SWMTKN-1-5g3hxk4q7eg3atnuw45ijrqei21y2aopiqha31v04jwmnz5svt-bsudrrah0zgdx7qzjcm6xvfy3 \
    172.24.2.63:2377
```

使用docker info和docker node ls查看集群中的相关信息：
```shell
[neusoft@localhost ~/docker]$ docker node ls
ID                           HOSTNAME               STATUS  AVAILABILITY  MANAGER STATUS
5levgh6t0qizmgwyn789dcuqx *  localhost.localdomain  Ready   Active        Leader
```
##### 把docker中的`selinux`去掉，然后重启docker
```shell
[root@localhost ~]# vi /etc/sysconfig/docker

# /etc/sysconfig/docker

# Modify these options if you want to change the way the docker daemon runs
#OPTIONS='--selinux-enabled --log-driver=journald --signature-verification=false'
OPTIONS='--log-driver=journald --signature-verification=false'
if [ -z "${DOCKER_CERT_PATH}" ]; then
    DOCKER_CERT_PATH=/etc/docker
fi

[root@localhost ~]# systemctl restart docker
```
##### 将Worker节点加入swarm集群
登录到172.24.2.62主机上，执行前面创建swarm时输出的命令：
```shell
[root@localhost ~]# docker swarm join \
>     --token SWMTKN-1-5g3hxk4q7eg3atnuw45ijrqei21y2aopiqha31v04jwmnz5svt-bsudrrah0zgdx7qzjcm6xvfy3 \
>     172.24.2.63:2377
This node joined a swarm as a worker.
```

去`master`查看节点信息
```shell
[neusoft@localhost ~/docker]$ docker node ls
ID                           HOSTNAME               STATUS  AVAILABILITY  MANAGER STATUS
5levgh6t0qizmgwyn789dcuqx *  localhost.localdomain  Ready   Active        Leader
i8x5wur7ux01rewkffavfyw3x    localhost.localdomain  Ready   Active        
```
这个时候能看见有2个节点了，但是hostname无法识别，所以我们可以使用命令来修改hostname的内容
```shell
[root@localhost ~]# hostnamectl set-hostname 172.24.2.62
[root@localhost ~]# systemctl restart docker
```
我们在去`master`中查看节点信息
```shell
[neusoft@localhost ~/docker]$ docker node ls
ID                           HOSTNAME               STATUS  AVAILABILITY  MANAGER STATUS
5levgh6t0qizmgwyn789dcuqx *  localhost.localdomain  Ready   Active        Leader
i8x5wur7ux01rewkffavfyw3x    172.24.2.62            Ready   Active
```
那么节点加入成功了。