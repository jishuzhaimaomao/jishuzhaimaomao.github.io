---
title: "系列学习文章：导读与学习路线"
published: 2026-09-08
description: "前面的业务篇、技术篇、测试篇是按“文档分类”组织的，适合查阅；这个系列则是按“认知主线”组织，适合一口气读完，把 FastBee 真正“吃透”。"
tags: [FastBee,学习路线]
category: FastBee
draft: false
slug: fastbee-series-00
---
# 系列学习文章：导读与学习路线

## 这个系列要解决什么问题

前面的业务篇、技术篇、测试篇是按“文档分类”组织的，适合查阅；这个系列则是按“**认知主线**”组织，适合一口气读完，把 FastBee 真正“吃透”。

每条主线都会追问三个问题：

1. 物联网平台为什么要这样设计？
2. FastBee 在代码里怎么落地？
3. 换到真实项目/二次开发，你会踩什么坑？

## 文章路线

### 第一阶段：理解平台存在的意义

- [01-为什么需要物联网平台](01-为什么需要物联网平台.md)

### 第二阶段：理解建模与通信

- [02-物模型：平台的语义中枢](02-物模型-平台的语义中枢.md)
- [03-自研 MQTT Broker：从“能用”到“懂它”](03-自研MQTT-Broker-从-能用-到-懂它.md)

### 第三阶段：跟着数据走一遍

- [04-一条上报数据的旅程](04-一条上报数据的旅程.md)
- [05-平台如何控制设备](05-平台如何控制设备.md)
- [06-设备影子与在线状态管理](06-设备影子与在线状态管理.md)
- [07-时序数据：存得下、查得快](07-时序数据-存得下查得快.md)

### 第四阶段：平台的高级能力

- [08-规则引擎：让数据流动起来](08-规则引擎让数据流动起来.md)
- [09-视频监控设备如何接入](09-视频监控设备如何接入.md)
- [10-网关与子设备：采集那点事](10-网关与子设备那些事.md)

### 第五阶段：动手与落地

- [11-从若依到 FastBee：二次开发心法](11-从若依到FastBee-二次开发入门.md)
- [12-把 FastBee 跑起来的部署实战](12-把FastBee跑起来的部署实战.md)

## 建议节奏

| 目标 | 建议 |
| --- | --- |
| 3 小时快速建立全景 | 01→02→04→05→08，配合部署体验 |
| 1 天源码级学习 | 全部文章 + 打开对应源码走读 |
| 准备做二次开发 | 02→04→05→11→12，实践后回看 03/07 |
| 准备做测试 | 先 01~10 了解系统，再执行测试篇手册 |

## 阅读时的“自测”问题

读完每个主题，问自己：

- 能否不查代码说出“从设备发一条温度到 Web 曲线”经过的所有类？
- 能否指出 FastBee 把哪些状态放在 Redis、哪些放在 MySQL、哪些放在 JVM 内存？
- 能否说出开源版哪些能力是“框架完整、业务未闭合”（如 OTA）？

如果都能回答，这个项目就真的吃透了。

## 源码速查

系列文章频繁引用以下文件，建议在编辑器中固定收藏：

```text
springboot/fastbee-server/mqtt-broker/src/main/java/com/fastbee/mqtt/handler/MqttPublish.java
springboot/fastbee-server/mqtt-broker/src/main/java/com/fastbee/mqtt/service/impl/DeviceReportMessageServiceImpl.java
springboot/fastbee-server/mqtt-broker/src/main/java/com/fastbee/mqtt/service/impl/DataHandlerImpl.java
springboot/fastbee-service/fastbee-iot-service/src/main/java/com/fastbee/iot/service/impl/DeviceServiceImpl.java
springboot/fastbee-service/fastbee-iot-service/src/main/java/com/fastbee/iot/tsdb/service/impl/MySqlLogServiceImpl.java
springboot/fastbee-gateway/fastbee-mq/src/main/java/com/fastbee/mq/service/impl/FunctionInvokeImpl.java
springboot/fastbee-server/mqtt-broker/src/main/java/com/fastbee/mqtt/service/impl/MqttMessagePublishImpl.java
springboot/fastbee-service/fastbee-iot-service/src/main/java/com/fastbee/iot/service/impl/SceneServiceImpl.java
springboot/fastbee-common/src/main/java/com/fastbee/common/enums/TopicType.java
springboot/sql/fastbee.sql
```
