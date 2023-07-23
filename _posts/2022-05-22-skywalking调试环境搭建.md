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

JDK 11：官方提倡
Maven3 >3.6
Git
npm
nodejs >12.0.0
IDEA

#### 下载源代码

``` text
git clone --recurse-submodules https://github.com/apache/skywalking.git
cd skywalking/
./mvnw clean package -DskipTests
```

或

``` text
git clone https://github.com/apache/skywalking.git
cd skywalking/
git submodule init
git submodule update
```

打开IDEA Terminal执行Maven编译命令：

``` text
 ./mvnw compile -Dmaven.test.skip=true
```

执行完成之后，会生成许多源码文件，因此我们需要将文件所在目录设置为源码目录，便于IDEA在编译时进行识别。

设置源码目录
分别将下边8个目录设置为源码目录：

``` text

grpc-java and java folders in apm-protocol/apm-network/target/generated-sources/protobuf
grpc-java and java folders in oap-server/server-core/target/generated-sources/protobuf
grpc-java and java folders in oap-server/server-receiver-plugin/receiver-proto/target/generated-sources/fbs
grpc-java and java folders in oap-server/server-receiver-plugin/receiver-proto/target/generated-sources/protobuf
grpc-java and java folders in oap-server/exporter/target/generated-sources/protobuf
grpc-java and java folders in oap-server/server-configuration/grpc-configuration-sync/target/generated-sources/protobuf
grpc-java and java folders in oap-server/server-alarm-plugin/target/generated-sources/protobuf
antlr4 folder in oap-server/oal-grammar/target/generated-sources
```

设置方法如下（以apm-protocol/apm-network/target/generated-sources/protobuf为例）：

在IDEA上找到该目录-->右键-->Mark Directory as-->Generated Source Root

设置后对应目录编程蓝色，则表明设置成功。


注意：

- 目前2023/7/23 的版本master最新的构建不过，只好checkout到8.8.x
- receive-proto项目依赖的flatbuffers-compiler在阿里云镜像找不到，需要将版本从1.12.0.1修改为2.0.8，执行完成再改回去执行一次编译就可以了！
- npm执行报错需要通过命令安装依赖或者升级npm到12以上 npm i polyfill-object.fromentries,主要原因是apm-webapp pom文件使用的node版本太老需要修改为最新的20.5.0

#### 编译源代码

分别对skywalking和skywalking-java进行编译，skywalking-java编译后会在skywalking-agent目录生成skywalking-agent.jar，后面我们会用到

``` text
./mvnw clean package -DskipTests
```

#### 启动OAP Server

运行OAP-server的org.apache.skywalking.oap.server.starter.OAPServerStartUp的#main(args)方法,启动SkyWalking OAP Server。

- 运行过程中报graphsql 初始化MetaDataQuery.getAllService报错No TypeDefinition for type name Service，经历调试发现是生成的jar包query-graphql-plugin里面query-protocol目录下的metadata.graphqls存在问题，重新新install server-query-plugin项目后正常运行。

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
启动参数设置

``` text
-javaagent:/xxx/skywalking-java/skywalking-agent/skywalking-agent.jar
-Dskywalking.collector.grpc_channel_check_interval=2
-Dskywalking.collector.app_and_service_register_check_interval=2
-Dcollector.discovery_check_interval=2
-Dskywalking.collector.backend_service=localhost:11800
-Dskywalking.agent.service_name=business-zone::projectA
-Dskywalking.logging.level=info
-Dskywalking.plugin.toolkit.log.grpc.reporter.server_host=localhost
-Dskywalking.plugin.toolkit.log.grpc.reporter.server_port=11800
-Dskywalking.plugin.toolkit.log.grpc.reporter.max_message_size=10485760
-Dskywalking.plugin.toolkit.log.grpc.reporter.upstream_timeout=30

然后通过下面的方法启动ProjectA

test.skywalking.springcloud.test.projecta.ProjectA#main
```

启动后
浏览器打开 <http://localhost/projectA/test>
