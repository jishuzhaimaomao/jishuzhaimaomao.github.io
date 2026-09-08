---
title: "[FastBee·技术篇] 总体架构与模块清单"
published: 2026-09-08
description: "FastBee 是一个单体分层 + 模块化 Maven 工程 + 内置协议服务的架构："
tags: [FastBee,FastBee·技术篇,系统架构]
category: FastBee·技术篇
draft: false
slug: fastbee-tech-1
---
# 总体架构与模块清单

## 1. 一句话架构

FastBee 是一个**单体分层 + 模块化 Maven 工程 + 内置协议服务**的架构：

- 部署上只有 1 个 Spring Boot Jar（`fastbee-admin`），却同时承载：
  - Web REST API（若依 Web 层）
  - 内置 Netty MQTT Broker（TCP 1883 / WebSocket 8083）
  - SIP/GB28181 信令服务（UDP 5061）
  - 消息处理、规则引擎、业务服务、数据访问
- 进程内模块通过 Spring Bean 直接调用；跨节点/外部 MQTT 场景设计了 Redis Channel/内存队列兼容层。

## 2. 分层总览

```text
┌────────────────────────────── Web 前端 Vue2 + Element-UI ──────────────────────────────┐
│  REST(HTTP/HTTPS)                MQTT over WebSocket                                   │
└──────────────┬───────────────────────────────────────────┬───────────────────────────┘
               │ /prod-api 等                              │ /mqtt (8083)
┌──────────────▼───────────────────────────────────────────▼───────────────────────────┐
│                        fastbee-admin：Spring Boot 可执行 Jar                           │
│  ┌──────────────┐  ┌────────────────────┐  ┌───────────────────────────┐            │
│  │ Web 控制器层  │  │ 内置 MQTT Broker     │  │ SIP/GB28181 服务            │            │
│  │ fastbee-open │  │ mqtt-broker + boot  │  │ sip-server                 │            │
│  │ -api         │  │ -strap + base-server│  └────────────┬──────────────┘            │
│  └──────┬───────┘  └─────────┬──────────┘               │ ZLMediaKit                 │
│         │                    │ 消息/状态                  │                           │
│  ┌──────▼────────────────────▼───────────────────────────▼───────────────────────┐   │
│  │              fastbee-mq / fastbee-service / fastbee-plugs                     │   │
│  │  规则脚本 → 设备服务 → 物模型/缓存 → 时序存储抽象 → Mapper → DB                 │   │
│  └──────────────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

## 3. Maven 多模块关系

根 POM 声明模块（`springboot/pom.xml`），最终由 `fastbee-admin` 聚合打包：

| 模块 | 职责 | 关键子包/依赖 |
| --- | --- | --- |
| `fastbee-admin` | 启动入口、Web Controller 装配 | 依赖 framework、open-api、boot-strap、gateway-boot 等 |
| `fastbee-common` | 通用常量/枚举/工具/消息模型/Redis | `com.fastbee.common.*` |
| `fastbee-framework` | 若依框架层：Security、切面、数据源、异常 | `com.fastbee.framework.*` |
| `fastbee-open-api` | “开放 API”实际上是所有 REST Controller | `com.fastbee.data.controller.*` |
| `fastbee-service/fastbee-iot-service` | IoT 业务领域、Service、Mapper、TSDB 抽象 | `com.fastbee.iot.*` |
| `fastbee-service/fastbee-system-service` | 系统服务（用户等）与 MyBatis 类型处理器 | `com.fastbee.system.*` |
| `fastbee-server/mqtt-broker` | Netty MQTT Broker | `com.fastbee.mqtt.*` |
| `fastbee-server/base-server` | Netty 通用会话/协议抽象 | `com.fastbee.base.*` |
| `fastbee-server/sip-server` | GB28181 SIP 信令 | `com.fastbee.sip.*` |
| `fastbee-server/iot-server-core` | `Server` 抽象与 Netty 配置 | `com.fastbee.server.*` |
| `fastbee-server/boot-strap` | 启动 MQTT/WebSocket Server Bean | `MQTTBootStrap` |
| `fastbee-gateway/fastbee-mq` | 消息服务接口、消息处理器、Redis Channel/队列 | `com.fastbee.mq.*` |
| `fastbee-gateway/gateway-boot` | 启动内部 MQTT 客户端与监听 | `com.fastbee.gateway.boot.*` |
| `fastbee-plugs/fastbee-ruleEngine` | LiteFlow 规则引擎装配 | `com.fastbee.ruleEngine.*` |
| `fastbee-plugs/fastbee-quartz` | Quartz 定时任务 | `com.fastbee.quartz.*` |
| `fastbee-plugs/fastbee-generator` | 代码生成器 | `com.fastbee.generator.*` |
| `fastbee-plugs/fastbee-http` | Forest HTTP 客户端示例/规则节点 | `com.fastbee.http.*` |
| `fastbee-plugs/fastbee-mqtt-client` | 平台内部 Paho MQTT 客户端（下发/外接 EMQX） | `com.fastbee.mqttclient.*` |
| `fastbee-protocol/fastbee-protocol-collect` | 内置 JSON 协议编解码与工具 | `com.fastbee.json.*` |

“服务端是多模块”不等于“微服务”。这一点非常重要：所有模块最终都编译进同一个 JVM，所以模块之间可以使用 `@Autowired` 直接注入，不需要 RPC。

## 4. 源码包地图（按调用链）

一条典型的“前端请求”代码路径：

```text
fastbee-open-api
  com.fastbee.data.controller.{iot,runtime,media,...}
      ↓
fastbee-iot-service
  com.fastbee.iot.service.(I...) / .impl
      ↓
  com.fastbee.iot.mapper + resources/mapper/**Mapper.xml
      ↓
MySQL（master 数据源） / TSDB（taos 等）
```

一条典型的“设备上报”代码路径：

```text
fastbee-server/mqtt-broker
  MqttMessageAdapter → MqttMessageDelegate → MqttPublish
      ↓
  IDeviceReportMessageService.parseReportMsg
      ↓
fastbee-gateway/fastbee-mq（接口与模型）
  IDataHandler → DataHandlerImpl
      ↓
fastbee-iot-service
  DeviceServiceImpl.reportDeviceThingsModelValue
      ↓
  tsdb.service.ILogService（按配置选择 MySQL/TDengine/...）
```

## 5. 关键设计判断

### 5.1 为什么 Broker 与业务在同一进程？

对中小规模场景，这是**最简单可靠**的方案：没有网络 RPC 的序列化/故障开销，设备消息直接落库，代码量小、易部署。代价是 Broker 处理吞吐受应用 JVM 影响、无法独立扩 Broker、单机内存状态无法直接横向扩容。

### 5.2 “消息通道”兼容层

`fastbee-gateway/fastbee-mq` 中同时存在：

- `IDeviceReportMessageService`（内置 Broker 直连处理入口）
- `MessageProducer/DeviceOtherQueue/DeviceOtherMsgHandler`（模拟/外部 EMQX 兼容入口）
- `RedisConsumeConfig`（Redis pub/sub 集群通道）
- `IEmqxMessageProducer`（EMQX v5 webhook 转发）

这就是开源版“业务&协议解耦、网络协议可横向扩展”的第一步：Broker 只负责收发与转发，业务消息通过接口层进入 IoT 服务。

## 6. 技术栈速查表

| 关注点 | 技术选型 |
| --- | --- |
| Web 服务 | Spring MVC（内嵌 Tomcat），端口 8080 |
| 安全 | Spring Security + JWT + 若依 `SecurityUtils` |
| ORM | MyBatis-Plus + PageHelper + MyBatis XML |
| 多数据源 | dynamic-datasource-spring-boot-starter，`@DS("taos")` |
| 连接池/监控 | Druid + `/druid/*` |
| 缓存/会话 | Redis + Spring Data Redis |
| 消息 | Netty 内置 MQTT；Redis Channel（集群预留）；JVM 队列（DeviceOtherQueue） |
| 规则 | LiteFlow（链 + 脚本节点，Groovy/QLExpress/JS 等语言声明） |
| 定时 | Quartz（系统任务 + 设备任务 + 场景定时） |
| 协议 | MQTT 3.1/3.1.1（Netty codec）、GB28181（JAIN SIP + ZLMediaKit） |
| 文档 | Swagger/Springfox 3 |
| 前端 | Vue 2 + Element-UI + Vuex + ECharts + mqtt.js |
| 构建 | Maven、npm、Docker Compose |

## 7. 读代码入口建议

按依赖轻重排序：

1. `FastBeeApplication`（入口）
2. `springboot/fastbee-admin/src/main/resources/application*.yml`（配置总纲）
3. `fastbee-open-api/.../DeviceController` + `DeviceRuntimeController`（看 REST）
4. `mqtt-broker/.../handler/adapter/MqttMessageAdapter`（看协议入口）
5. `iot-service/.../DeviceServiceImpl`（看核心业务）

下一章：[后端框架与启动装配](/posts/fastbee-tech-2/)

