---
layout: post
title: skywalking本地调试
subtitle: 介绍skywalking本地调试搭建过程
date: 2022-05-22T00:00:00+08:00
author: gregorius
header-img: img/post-bg-universe.jpg
tags: [实践]
catalog: null
---

### 源代码地址

skywalking是一款很优秀的监控系统，采用代理方式基于切面编程来拦截请求进行监控数据记录，架构采用CS模式

- [server端地址](https://github.com/apache/skywalking)
- [java客户端地址](https://github.com/apache/skywalking-java)

### [server端构建过程](https://github.com/apache/skywalking/blob/master/docs/en/guides/How-to-build.md#build-from-github)

此文不做原理解析，只讲大概入门思路，server里面同时也包含了后端管理和监控UI页面，基于spring boot开发，项目名称是apm-webapp

#### 工具

工欲善其事必先利其器，因此在构建之前需要说明一些需要的工具：

JDK 8：官方提倡
Maven3
Git
npm
IDEA

#### 下载源代码

``` text
git clone --recurse-submodules https://github.com/apache/skywalking.git
cd skywalking/
./mvnw clean package -DskipTests

或

git clone https://github.com/apache/skywalking.git
cd skywalking/
git submodule init
git submodule update
./mvnw clean package -DskipTests
```

#### 编译源代码

设置源码目录
分别将下边5个目录设置为源码目录：

apm-protocol/apm-network/target/generated-sources/protobuf
oap-server/server-core/target/generated-sources/protobuf
oap-server/server-receiver-plugin/receiver-proto/target/generated-sources/protobuf
oap-server/exporter/target/generated-sources/protobuf
oap-server/server-configuration/grpc-configuration-sync/target/generated-sources/protobuf
oap-server/oal-grammar/target/generated-sources
设置方法如下（以apm-protocol/apm-network/target/generated-sources/protobuf为例）：

在IDEA上找到该目录-->右键-->Mark Directory as-->Generated Source Root

设置后对应目录编程蓝色，则表明设置成功。

打开IDEA Terminal执行Maven编译命令：
 ./mvnw compile -Dmaven.test.skip=true
执行完成之后，会生成许多源码文件，因此我们需要将文件所在目录设置为源码目录，便于IDEA在编译时进行识别。

#### 启动OAP Server

运行OAP-server的org.apache.skywalking.oap.server.starter.OAPServerStartUp的#main(args)方法,启动SkyWalking OAP Server。

#### 启动SkyWalking UI

运行 apm-webapp 的 org.apache.skywalking.apm.webapp.ApplicationStartUp 的 #main(args) 方法，启动 SkyWalking UI 。
浏览器打开 <http://127.0.0.1:8080>

#### 演示项目

从<https://github.com/SkyAPMTest/skywalking-live-demo> 下载演示项目

``` text
> git clone https://github.com/SkywalkingTest/skywalking-live-demo.git
> cd skywalking-live-demo 
> mvn clean package # build the live demo archive
```

演示项目基于Spring Boot搭建，ProjectA调用ProjectB,ProjectC,ProjectD，首先通过

``` text
test.skywalking.springcloud.test.projecta.ProjectA#main
```

需要启动ProjectA
浏览器打开 <http://localhost:8764/projectA/test>

##### 注意事项

上面的文章可能是基于skywalking比较老的版本搭建，启动时maven项目依赖的**log4j包和server端有冲突**，所有需要修改pom.xml文档，将pom文件版本号改为**2.6.2**，同时文章是基于老版本编码，agent的构建过程有差异，需要基于java客户端进行构建，构建完成后会在skywalking-agent目录生成skywalking-agent.jar，同时这个demo项目是由几个spring boot 项目组成，A调用B，C，初始为避免复杂性可以先将调用B,C项目代码注释。

同时支持其实A项目的端口是**80，B，C也是80**，如果需要同时启动需要修改B，C端口分别为**81，82**
