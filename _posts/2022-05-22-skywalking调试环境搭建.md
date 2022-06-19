---
layout: post
title: skywalking本地调试
subtitle: 介绍skywalking本地调试搭建过程
date: 2022-05-22T00:00:00+08:00
author: gregorius
header-img: img/post-bg-swift.jpg
catalog: true
tags: [实践]
---

### 源代码地址
skywalking是一款很优秀的监控系统，采用代理方式基于切面编程来拦截请求进行监控数据记录，架构采用CS模式
- [server端地址](https://github.com/apache/skywalking)
- [java客户端地址](https://github.com/apache/skywalking-java)

### [server端构建过程](https://github.com/apache/skywalking/blob/master/docs/en/guides/How-to-build.md#build-from-github)

此文不做原理解析，只讲大概入门思路，server里面同时也包含了后端管理和监控UI页面，基于spring boot开发，项目名称是apm-webapp,如果想本地调试skywalking源代码，可以依据[这篇文章](ttp://soiiy.com/java/14180.html)来进行搭建，但是搭建过程需要一些坑，下面指出。

上面的文章可能是基于skywalking比较老的版本搭建，启动时maven项目依赖的**log4j包和server端有冲突**，所有需要修改pom.xml文档，将pom文件版本号改为**2.6.2**，同时文章是基于老版本编码，agent的构建过程有差异，需要基于java客户端进行构建，构建完成后会在skywalking-agent目录生成skywalking-agent.jar，同时这个demo项目是由几个spring boot 项目组成，A调用B，C，初始为避免复杂性可以先将调用B,C项目代码注释。

同时支持其实A项目的端口是**80，B，C也是80**，如果需要同时启动需要修改B，C端口分别为**81，82**，同时UI项目的启动此文没有提及，启动UI需要启动
``` Java
org.apache.skywalking.oap.server.webapp.ApplicationStartUp
```

[UI](https://skywalking.apache.org/zh/2020-04-19-skywalking-quick-start/)

UI的地址端口是8080，由于是最新版本，界面会有差异，目前最新的界面服务在普通服务页签，同时里面可以通过trace页签来查看调用链。
