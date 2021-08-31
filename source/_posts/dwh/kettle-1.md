title: kettle
author: Figthing
tags:
  - dwh
  - etl
  - kettle
  - ''
categories:
  - dwh
  - etl
date: 2019-10-08 18:08:00
---
### Kettle

#### 概述

Kettle是一款国外开源的ETL工具，纯java编写，可以在Window、Linux、Unix上运行，绿色无需安装，数据抽取高效稳定。中文名称叫水壶，该项目的主程序员MATT 希望把各种数据放到一个壶里，然后以一种指定的格式流出。



#### 产品家族

Kettle家族目前包括4个产品：Spoon、Pan、CHEF、Kitchen

- **SPOON** 允许你通过图形界面来设计ETL转换过程（Transformation）。

- **PAN** 允许你批量运行由Spoon设计的ETL转换 (例如使用一个时间调度器)。Pan是一个后台执行的程序，没有图形界面。

- **CHEF** 允许你创建任务（Job）。 任务通过允许每个转换，任务，脚本等等，更有利于自动化更新数据仓库的复杂工作。任务通过允许每个转换，任务，脚本等等。任务将会被检查，看看是否正确地运行了。

- **KITCHEN** 允许你批量使用由Chef设计的任务 (例如使用一个时间调度器)。KITCHEN也是一个后台运行的程序。

<!--more-->


#### 手动编译和运行

##### 准备工作

- JDK1.8+

- 下载源码：[地址](https://github.com/pentaho/pentaho-kettle)

- 版本号：pentaho-kettle-8.3.0.4-R
- Maven version 3+

- [settings.xml](https://raw.githubusercontent.com/pentaho/maven-parent-poms/master/maven-support-files/settings.xml)配置



##### 编译

在源码的根目录中执行，会等一段时间

```shell
mvn clean install -DskipTests
```



##### 运行

编译成功后，会在`assemblies/client/target`生成一个`pdi-ce-*.zip`压缩包文件，解压后双击运行`Spoon.bat`就启动了。



#### 问题

- 问题：编译时文件过大，通过maven无法下载文件

  解决：可以根据拉取的地址进行直接下载，并发动maven本地仓库位置



- 问题：Spoon.bat启动直接未响应

  解决：修改该文件中的`-Xms`，`-Xmx`

  ```shell
  if "%PENTAHO_DI_JAVA_OPTIONS%"=="" set PENTAHO_DI_JAVA_OPTIONS="-Xms512m" "-Xmx512m" "-XX:MaxPermSize=256m"
  ```