title: Spring Cloud 配置中心 Github SSH验证（二）
author: Figthing
tags:
  - java
  - spring-cloud
categories:
  - java
  - spring-cloud
date: 2020-02-02 16:53:00
---
### Spring Cloud 配置中心 Github SSH验证（二）

#### 概述

在上一章讲解了如何使用Spring Cloud配置中心读取Github SSH的文件后，发现如果将`spring.cloud.config.server.git.private-key=`配置的值设置为一个环境变量，在JVM:JAVA_OPS是不可行的，在网上找了很多资料国内的解决方案是使用`private_key_file`，但官方并未提供，最后找到了解决方案，下面将给出干货提供给大家。

**注意：前面工作不在叙述，请自行参考，[Spring Cloud 配置中心 Github SSH验证（一）](http://blog.appydm.com/java/spring-cloud/java/spring-cloud/config-server-github-ssh/)**


<!--more-->


#### 校验密钥

续密钥生成后，执行`ssh -vT git@github.com`进行连接的身份验证测试，然后在增加到github中，如果服务器要使用，直接将`id_rsa`，`known_hosts`，复制到服务器上就可以了。

默认情况下`.ssh/id_rsa`是在GIT SSH身份验证期间发送的，如果您在下面有另一个名为RSA的文件，那么你可以`.ssh`下创建一个`config`配置文件并标识该文件内容如下。

```shell
Host github.com
	IdentityFile ~/.ssh/mygitid_rsa
```

**生成SSH的时候，一定要输入密码，处于安全考虑**



#### Spring Cloud 配置中心properties

```properties
spring.cloud.config.server.git.uri=xxxx
spring.cloud.config.server.git.basedir=./tmp/configv	- 本地保存的配置地址
spring.cloud.config.server.git.clone-on-start=true		- 启动时就克隆配置缓存到本地
spring.cloud.config.server.git.force-pull=true		- 本地副本是脏的,强制从远程存储库拉取配置
spring.cloud.config.server.git.search-paths=/dev	- GIT搜索目录位置
spring.cloud.config.server.git.passphrase=123456	- 生成SSH的密码
```

现在可以尝试启动配置中心来获取配置信息了