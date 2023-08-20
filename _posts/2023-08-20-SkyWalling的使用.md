---
layout: post
title: SkyWalking的使用
subtitle: 
date: 2023-08-20T00:00:00+08:00
author: gregorius
header-img: img/post-bg-universe.jpg
tags: [实践]
catalog: true
---

### 监控界面

基于上一篇[Skywalking调试环境搭建](https://www.jianshu.com/p/2bc101cce583),启动本地服务之后，可以看到管理控制台注册了4个服务

![skservice](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureskservice.PNG)

可以看到上面有4个页签，分别是服务，拓扑，跟踪信息，以及日志，这里是所有服务的管理入口，我们需要查询整体服务的信息，可以根据这4个页签进行查询。

点击具体的服务超连接进去之后可以看到具体服务的监控情况，里面的指标更加的详细

![skservicedetail](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureskservicedetail.PNG)

可以看到这里有页签，有模块，最上面是按时间维度统计的服务运行情况，页签分为概览，实例，端点，拓扑，跟踪信息，抽样以及日志等。

概览里面主要是服务的一些成功率以及性能相关的数据，实例统计的是服务所在的服务器信息，里面包括了JVM相关的数据等

![JVM](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureJVM.PNG)

端点就是服务所提供的具体的功能，比如：服务的接口路径分别为 projectA/test,projectB/test

![endpoint](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureendpoint.PNG)

端点里面的Trace信息更加的详细可读

![endpointtrace](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureendpointtrace.PNG)

拓扑就是服务的调用路径图

![Topology](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/pictureTopology.PNG)

Trace就是服务的调试链信息

![trace](https://raw.githubusercontent.com/gregoriusxu/gallery/main/blogs/picturetrace.PNG)

### 日志上传设置

SkyWalking基于log4j2实现日志上报，客户端如果上报日志到服务端需要设置相应的Appender如下：

``` xml

<Configuration status="debug">
    <Appenders>
        <GRPCLogClientAppender name="grpc-log"/>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout charset="UTF-8" pattern="[%traceId] - %m%n"/>
        </Console>
    </Appenders>
    <Loggers>
        <logger name="com.a.eye.skywalking.ui" level="debug" additivity="false">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="grpc-log"/>
        </logger>
        <logger name="org.apache.kafka" level="OFF"></logger>
        <logger name="org.apache.skywalking.apm.dependencies" level="OFF"></logger>
        <Root level="debug">
            <AppenderRef ref="Console"/>
            <AppenderRef ref="grpc-log"/>
        </Root>
    </Loggers>
</Configuration>
```

### 告警设置

告警设置需要设置在service-starter项目的alarm-settings.yml,下面代码为配置告警规则的代码，skywalking 还支持使用者配置告警接口，来及时发送通知，如发送短信 / 邮件等。如配置文件中的 webhooks 中。

``` yml

# Licensed to the Apache Software Foundation (ASF) under one
# or more contributor license agreements.  See the NOTICE file
# distributed with this work for additional information
# regarding copyright ownership.  The ASF licenses this file
# to you under the Apache License, Version 2.0 (the
# "License"); you may not use this file except in compliance
# with the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Sample alarm rules.
rules:
  # Rule unique name, must be ended with `_rule`.
  service_resp_time_rule:
    metrics-name: service_resp_time
    op: ">"
    threshold: 1000
    period: 10
    count: 3
    silence-period: 5
    message: Response time of service {name} is more than 1000ms in 3 minutes of last 10 minutes.
  service_sla_rule:
    # Metrics value need to be long, double or int
    metrics-name: service_sla
    op: "<"
    threshold: 8000
    # The length of time to evaluate the metrics
    period: 10
    # How many times after the metrics match the condition, will trigger alarm
    count: 2
    # How many times of checks, the alarm keeps silence after alarm triggered, default as same as period.
    silence-period: 3
    message: Successful rate of service {name} is lower than 80% in 2 minutes of last 10 minutes
  service_p90_sla_rule:
    # Metrics value need to be long, double or int
    metrics-name: service_p90
    op: ">"
    threshold: 1000
    period: 10
    count: 3
    silence-period: 5
    message: 90% response time of service {name} is more than 1000ms in 3 minutes of last 10 minutes
  service_instance_resp_time_rule:
    metrics-name: service_instance_resp_time
    op: ">"
    threshold: 1000
    period: 10
    count: 2
    silence-period: 5
    message: Response time of service instance {name} is more than 1000ms in 2 minutes of last 10 minutes
#  Active endpoint related metrics alarm will cost more memory than service and service instance metrics alarm.
#  Because the number of endpoint is much more than service and instance.
#
  endpoint_avg_rule:
    metrics-name: endpoint_avg
    op: ">"
    threshold: 1000
    period: 10
    count: 2
    silence-period: 5
    message: Response time of endpoint {name} is more than 1000ms in 2 minutes of last 10 minutes

#webhooks:
#  - http://127.0.0.1/notify/
#  - http://127.0.0.1/go-wechat/
```

最新版本的除了默认加入了dubbo，以及钉钉，飞书，slack,pagerDuty等软件的上报告警支持，如下：

``` yml

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

# Sample alarm rules.
rules:
  # Rule unique name, must be ended with `_rule`.
#  endpoint_percent_rule:
#    # Metrics value need to be long, double or int
#    metrics-name: endpoint_percent
#    threshold: 75
#    op: <
#    # The length of time to evaluate the metrics
#    period: 10
#    # How many times after the metrics match the condition, will trigger alarm
#    count: 3
#    # How many times of checks, the alarm keeps silence after alarm triggered, default as same as period.
#    silence-period: 10
#    message: Successful rate of endpoint {name} is lower than 75%
  service_resp_time_rule:
    metrics-name: service_resp_time
    # [Optional] Default, match all services in this metrics
    include-names:
      - dubbox-provider
      - dubbox-consumer
    threshold: 1000
    op: ">"
    period: 10
    count: 1
    tags:
      level: WARNING

webhooks:
#  - http://127.0.0.1/notify/
#  - http://127.0.0.1/go-wechat/

gRPCHook:
#  target_host: 127.0.0.1
#  target_port: 9888

slackHooks:
  textTemplate: |-
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": ":alarm_clock: *Apache Skywalking Alarm* \n **%s**."
      }
    }
  webhooks:
#    - https://hooks.slack.com/services/x/y/zssss

wechatHooks:
  textTemplate: |-
    {
      "msgtype": "text",
      "text": {
        "content": "Apache SkyWalking Alarm: \n %s."
      }
    }
  webhooks:
#    - https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=dummy_key

dingtalkHooks:
  textTemplate: |-
    {
      "msgtype": "text",
      "text": {
        "content": "Apache SkyWalking Alarm: \n %s."
      }
    }
  webhooks:
#    - url: https://oapi.dingtalk.com/robot/send?access_token=dummy_token
#      secret: dummysecret

feishuHooks:
  textTemplate: |-
    {
    "msg_type": "text",
    # at someone with feishu_user_ids
    # "ats": "feishu_user_id_1,feishu_user_id_2",
    "content": {
      "text": "Apache SkyWalking Alarm: \n %s."
      }
    }
  webhooks:
#    - url: https://open.feishu.cn/open-apis/bot/v2/hook/dummy_token
#      secret: dummysecret

welinkHooks:
  textTemplate: "Apache SkyWalking Alarm: \n %s."
  webhooks:
#    # you may find your own client_id and client_secret in your app, below are dummy, need to change.
#    - client_id: "dummy_client_id"
#      client_secret: dummy_secret_key
#      access_token_url: https://open.welink.huaweicloud.com/api/auth/v2/tickets
#      message_url: https://open.welink.huaweicloud.com/api/welinkim/v1/im-service/chat/group-chat
#      # if you send to multi group at a time, separate group_ids with commas, e.g. "123xx","456xx"
#      group_ids: "dummy_group_id"
#      # make a name you like for the robot, it will display in group
#      robot_name: robot

pagerDutyHooks:
  textTemplate: "Apache SkyWalking Alarm: \n %s."
  integrationKeys:
#    # you can find your integration key(s) on the Events API V2 integration page for your PagerDuty service(s).
#    # (you may need to create an Events API V2 integration for your PagerDuty service if you don't have one yet)
#    # below are dummy keys that should be replaced with your own integration keys.
#    - dummy_key
#    - dummy_key2
```
