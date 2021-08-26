title: Spring Cloud 配置中心 Github SSH验证（一）
author: Figthing
tags:
  - java
  - spring-cloud
categories:
  - java
  - spring-cloud
date: 2020-01-30 17:28:00
---
### Spring Cloud 配置中心 Github SSH验证

#### 概述

Spring Cloud Config为分布式系统中的外部配置提供服务器和客户端支持，方便部署与运维。

目前有一些用的比较多的开源的配置中心，比如携程的 Apollo、阿里Nacos、百度的 Disconf 等，对比 Spring Cloud Config，这些配置中心功能更加强大。有兴趣的可以拿来试一试。

接下来，我们开始使用Spring Cloud来搭建一个配置中心，并以 github 作为配置存储。除了 git 外，还可以用数据库、svn、本地文件等作为存储。

<!--more-->

#### Github SSH配置

##### 密匙生成
Windows下生成密匙，可以直接使用`Git Bash`工具，执行`ssh-keygen -m PEM -t rsa -b 4096`命令

![image-20200130201553870](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/spring/spring-cloud/config/image-20200130201553870.png)

在弹出来的选项直接按回车就可以了，密匙生成目录`C:\Users\用户\.ssh`

- 公钥`id_rsa.pub`
- 私钥`id_rsa`

##### 配置SSH

点击右上角头像，弹出的菜单中，点击`Settings`进入设置

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/spring/spring-cloud/config/image-20200130205501744.png" alt="image-20200130205501744" style="zoom:50%;" />

点击`SSH and GPG keys`，进入SSH的key设置

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/spring/spring-cloud/config/image-20200130205550957.png" alt="image-20200130205550957" style="zoom:50%;" />

点击`New SSH key`，进行SSH的添加，记住这里的Title随便填写，Key就是我们的公钥，打开公钥内容全部复制粘贴到Key的输入框中，点击`Add SSH key`即可！公钥是`ssh-rsa `开头

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/spring/spring-cloud/config/image-20200130205705772.png" alt="image-20200130205705772" style="zoom:50%;" />

添加公钥后就会显示

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/spring/spring-cloud/config/image-20200130205930159.png" alt="image-20200130205930159" style="zoom:50%;" />



##### 校验密匙

这时我们回到Windows上，打开Git Bash，在执行`ssh -vT git@github.com`命令

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/spring/spring-cloud/config/image-20200130210138318.png" alt="image-20200130210138318" style="zoom: 67%;" />

只要出现`You've successfully authenticated`就证明，已经验证成功了。

> 如果是内网搭建的Gitlab，可以使用 `ssh -vT git@ip -p 端口`命令


#### Spring Cloud 配置中心

##### 配置文件存放

在github中新建一个仓库，这个仓库一定要是**干净**的，就是最好不要有其他文件存在，不然会报错。因为我们使用了SSH，所以这是一个**私库**，我存放的位置是`local/eureka.properties`。

##### POM依赖

```xml
	<dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-config-server</artifactId>
    </dependency>
```

##### 项目配置文件

**properties文件**

```properties
spring.cloud.config.server.git.uri=git@github.com:Figthing/hyper-config-repo.git
spring.cloud.config.server.git.strict-host-key-checking=false
spring.cloud.config.server.git.ignore-local-ssh-settings=true
spring.cloud.config.server.git.search-paths=/local
spring.cloud.config.server.git.private-key=-----BEGIN RSA PRIVATE KEY-----\n\
aaaa\
xxxx\n\
-----END RSA PRIVATE KEY-----
```

​	**参数说明**

- spring.cloud.config.server.git.uri：配置的Github仓库的ssh地址
- spring.cloud.config.server.git.strict-host-key-checking：true-使用用户名密码，false-使用ssh key
- spring.cloud.config.server.git.ignore-local-ssh-settings：true-ssh 登陆方式
- spring.cloud.config.server.git.search-paths：git仓库地址下的相对地址 多个用逗号","分割。
- spring.cloud.config.server.git.private-key：私钥内容（可以使用变量代替），这里要注意几点
  - 第一行使用`\n\`结尾
  - 内容行使用`\`结尾
  - End结束之前使用`\n\`结尾

**yaml文件**

因我项目使用的`properties`，至于yaml就自行百度吧，应该相差不大。

#### 启动验证

**main类文件**

```java
@EnableConfigServer
@SpringBootApplication
public class ConfigApplication {

	public static void main(String[] args) {
		SpringApplication.run(ConfigApplication.class, args);
	}
}
```

**访问配置**

我们在URL中输入`http://localhost:8802/eureka/local`

<img src="https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/spring/spring-cloud/config/image-20200130213543634.png" alt="image-20200130213543634" style="zoom:50%;" />


Spring cloud config 的URL与配置文件的映射关系如下:

> /{application}/{profile}[/{label}]
> /{application}-{profile}.yml
> /{label}/{application}-{profile}.yml
> /{application}-{profile}.properties
> /{label}/{application}-{profile}.properties

我这里的application = eureka，profile = local