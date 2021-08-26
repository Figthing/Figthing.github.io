title: 携程Apollo配置中心
author: Figthing
tags:
  - spring cloud
  - apollo
  - ''
categories: []
date: 2019-04-09 13:13:00
---
## 携程 Apollo 配置中心

在Spring Boot 2.0 整合携程Apollo配置中心一文中，我们在本地快速部署试用了Apollo。本文将介绍如何按照分布式部署（采用Docker部署）的方式编译、打包、部署Apollo配置中心，从而可以在开发、测试、生产等环境分别部署运行。

### 部署准备
- docker 安装
- docker-compose 安装
- mysql 安装

### Apollo工程

| 工程名 | 优先级 | 说明 | 端口 | sql |
|-------------------------|
| apollo-configservice | 1 | 服务 | 8090 | [apolloconfigdb.sql]((https://github.com/ctripcorp/apollo/blob/master/scripts/docker-quick-start/sql/apolloconfigdb.sql) |
| apollo-adminservice | 2 | 管理 | 8091 | [apolloconfigdb.sql]((https://github.com/ctripcorp/apollo/blob/master/scripts/docker-quick-start/sql/apolloconfigdb.sql) |
| apollo-portal | 3 | 界面 | 8092 | [apolloportaldb.sql](https://github.com/ctripcorp/apollo/blob/master/scripts/docker-quick-start/sql/apolloportaldb.sql) |

<!-- more -->


### 部署策略
Apollo目前支持以下环境：
- DEV 开发环境
- FAT 测试环境，相当于alpha环境(功能测试)
- UAT 集成环境，相当于beta环境（回归测试）
- PRO 生产环境

### 网络策略（官网）
分布式部署的时候，apollo-configservice和apollo-adminservice需要把自己的IP和端口注册到Meta Server（apollo-configservice本身）。
Apollo客户端和Portal会从Meta Server获取服务的地址（IP+端口），然后通过服务地址直接访问。
所以如果实际部署的机器有多块网卡（如docker），或者存在某些网卡的IP是Apollo客户端和Portal无法访问的（如网络安全限制），那么我们就需要在apollo-configservice和apollo-adminservice中做相关限制以避免Eureka将这些网卡的IP注册到Meta Server。
如下面这个例子就是对于apollo-configservice，把docker0和veth.* 的网卡在注册到Eureka时忽略掉。

```shell
spring:
  application:
      name: apollo-configservice
  profiles:
    active: ${apollo_profile}
  cloud:
    inetutils:
      ignoredInterfaces:
        - docker0
        - veth.*
```

另外一种方式是直接指定要注册的IP，可以修改startup.sh，通过JVM System Property在运行时传入，如-Deureka.instance.ip-address=${指定的IP}，或者也可以修改apollo-adminservice或apollo-configservice 的bootstrap.yml文件，加入以下配置

```shell
eureka:
  instance:
    ip-address: ${指定的IP}
```

最后一种方式是直接指定要注册的IP+PORT，可以修改startup.sh，通过JVM System Property在运行时传入，如-Deureka.instance.homePageUrl=http://${指定的IP}:${指定的Port}，或者也可以修改apollo-adminservice或apollo-configservice 的bootstrap.yml文件，加入以下配置

```shell
eureka:
  instance:
    homePageUrl: http://${指定的IP}:${指定的Port}
    preferIpAddress: false
```

如果Apollo部署在公有云上，本地开发环境无法连接，但又需要做开发测试的话，客户端可以升级到0.11.0版本及以上，然后通过-Dapollo.configService=http://config-service的公网IP:端口来跳过meta service的服务发现

### 部署步骤
部署步骤共四步：

- 创建数据库，所有Apollo服务端都依赖于MySQL数据库，所以在启动时，应先配置数据才能启动服务；
- 获取安装包：通过源码构建
- 构建docker镜像：为apollo-configservice, apollo-adminservice, apollo-portal构建Docker镜像
- 部署Apollo服务端：构建镜像后通过docker compose就可以部署到公司的测试和生产环境了

#### 创建数据库

Apollo服务端共需要两个数据库：ApolloPortalDB和ApolloConfigDB，官网把数据库、表的创建和样例数据都分别准备了sql文件，只需要导入数据库即可。

| 服务器 | 数据库 | 端口 | 环境 |
|----------------------------|
| 172.24.2.65 | ApolloConfigDB | 3306 | dev |
| 172.24.2.65 | ApolloPortalDB | 3306 | dev |

#####  调整服务端配置
Apollo自身的一些配置是放在数据库里面的，所以需要针对实际情况做一些调整

- ApolloPortalDB
 > - 配置项统一存储在ServerConfig表中，也可以通过管理员工具 - 系统参数页面进行配置。
 > - apollo.portal.envs - 可支持的环境列表，默认值是dev，如果portal需要管理多个环境的话，以逗号分隔即可（大小写不敏感）

- ApolloConfigDB
 > - 配置项统一存储在ServerConfig表中，需要注意每个环境的ApolloConfigDB.ServerConfig都需要单独配置。
 > - eureka.service.url - Eureka服务Url，不管是apollo-configservice还是apollo-adminservice都需要向eureka服务注册，所以需要配置eureka服务地址。 按照目前的实现，apollo-configservice本身就是一个eureka服务，所以只需要填入apollo-configservice的地址即可，如有多个，用逗号分隔（注意不要忘了/eureka/后缀）。这里我填写http://172.24.2.63:8090/eureka。
 
#### 获取安装包
到github上进行[源码下载](https://github.com/ctripcorp/apollo)，如果github下载比较慢，可以去[ipaddress](https://www.ipaddress.com/)把CDN信息搜索出来，添加到hosts中
- 151.101.185.194 github.global.ssl.fastly.net 
- 192.30.253.113  github.com
- 192.30.253.120	codeload.github.com

##### 调整源码

配置数据库连接信息`scripts/build.bat`

```java
# config
set apollo_config_db_url="jdbc:mysql://172.24.2.65:3306/ApolloConfigDB?characterEncoding=utf8"
set apollo_config_db_username="apollo-config"
set apollo_config_db_password="apollo-config"

# portal
set apollo_portal_db_url="jdbc:mysql://172.24.2.65:3306/ApolloPortalDB?characterEncoding=utf8"
set apollo_portal_db_username="apollo-portal"
set apollo_portal_db_password="apollo-portal"

# dev_meta
set dev_meta="http://172.24.2.63:8090"
set META_SERVERS_OPTS=-Ddev_meta=%dev_meta%
```
###### 调整apollo-configservice工程
- 将所有端口改为8090
- `bootstrap.yml`调整`defaultZone: http://${eureka.instance.hostname}:8090/eureka/`
- `startup.sh` 新增 `export JAVA_OPTS="-Deureka.instance.ip-address=172.24.2.63"`

###### 调整apollo-configservice工程
- 将所有端口改为8091
- `bootstrap.yml`调整 `defaultZone: http://${eureka.instance.hostname}:8090/eureka/`
- `startup.sh` 新增 `export JAVA_OPTS="-Deureka.instance.ip-address=172.24.2.63"`

###### 调整apollo-portal工程
- 将所有端口改为8092
- `apollo-env.properties`调整`local.meta=http://172.24.2.63:8090`

##### 执行编译、打包
做完上述配置后，就可以执行编译和打包了。执行/scripts目录下build.sh脚本，该脚本会依次打包apollo-configservice, apollo-adminservice, apollo-portal。


##### 构建docker镜像
将target下的apollo-*-github.zip和Dockerfile上传到服务器，形成结构
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/apollo/1.png)

构建镜像
- docker build -t apollo-configservice:1.0.0 .
- docker build -t apollo-adminservice:1.0.0 .
- docker build -t apollo-portal:1.0.0 .
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/apollo/2.png)

##### 部署Apollo服务端
![](https://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/apollo/3.png)

在配置文件目录执行如下命令启动服务：
```shell
docker-compose up
```
