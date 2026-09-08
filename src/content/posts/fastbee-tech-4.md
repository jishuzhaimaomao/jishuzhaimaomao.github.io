---
title: "自研 MQTT Broker 实现解析"
published: 2026-09-08
description: "FastBee 的卖点之一是“内置 MQTT Broker，无需 EMQX”。本章从代码结构拆开它，重点讲清“能做什么、怎么做的、单机边界在哪”。"
tags: [FastBee,MQTT]
category: FastBee
draft: false
slug: fastbee-tech-4
---
# 自研 MQTT Broker 实现解析

FastBee 的卖点之一是“内置 MQTT Broker，无需 EMQX”。本章从代码结构拆开它，重点讲清“能做什么、怎么做的、单机边界在哪”。

## 1. 代码结构

```text
fastbee-server/mqtt-broker/
├── annotation/Process.java              # 报文类型 → 处理类注解
├── auth/AuthService.java                # 客户端认证
├── codec/WebSocketMqttCodec.java        # WebSocket 封包适配
├── handler/
│   ├── MqttConnect / MqttDisConnect     # 连接/断开
│   ├── MqttPingreq                      # 心跳
│   ├── MqttSubscribe / MqttUnsubscribe  # 订阅
│   ├── MqttPublish                      # 发布（核心）
│   ├── MqttPubAck/Rec/Rel/Pubcomp       # QoS1/QoS2 流程
│   └── adapter/                         # Adapter → Delegate → Handler
├── manager/
│   ├── SessionManger / ClientManager    # 会话与主题-客户端映射
│   ├── RetainMsgManager / WillMessageManager
│   ├── ResponseManager / MqttRemoteManager
├── model/                               # ClientMessage/Subscribe/Will...
├── server/MqttServer.java               # Netty TCP Server
├── server/WebSocketServer.java          # MQTT over WebSocket
└── service/                             # 存储/订阅/消息处理/发布接口
```

## 2. Netty 装配

`MqttServer extends Server`，启动时创建：

- bossGroup = 1 线程；
- workerGroup = N 线程；
- 业务线程池（可选 `businessCore`）；
- Pipeline：

```text
IdleStateHandler(读/写/总空闲)
  → MqttDecoder(2MB 上限)
  → MqttEncoder
  → MqttMessageAdapter（@Sharable）
```

`WebSocketServer` 复用同一套 handler 前加 `WebSocketMqttCodec`，浏览器通过 `ws://host:8083/mqtt` 接入。

### 空闲检测

应用配置 `server.broker.keep-alive=70` 秒。`MqttMessageAdapter.userEventTriggered` 收到 Idle 事件直接关闭通道；`ClientManager.updatePing` 也会更新“最近活跃时间”，发布消息时用 `DEVICE_PING_EXPIRED` 判断客户端是否可投递。

## 3. 报文分发机制

`MqttMessageDelegate` 构造时扫描所有 `MqttHandler`：

```java
Process annotation = handler.getClass().getAnnotation(Process.class);
processor.put(annotation.type(), handler);
```

于是每种 `MqttMessageType` 对应一个 handler：

| MQTT 报文 | 处理类 |
| --- | --- |
| CONNECT | MqttConnect |
| PUBLISH | MqttPublish |
| SUBSCRIBE/UNSUBSCRIBE | MqttSubscribe / MqttUnsubscribe |
| PINGREQ | MqttPingreq |
| PUBACK/PUBREC/PUBREL/PUBCOMP | 对应四个类 |
| DISCONNECT | MqttDisConnect |

未 CONNECT 就发其他报文会抛“客户端未连接”。

## 4. 连接与会话

### 4.1 CONNECT 流程

1. 组装 `Session`（clientId/版本/cleanSession/username/ip/ServerType.MQTT）；
2. `AuthService.auth` 校验（见认证篇）；
3. 失败：回 CONNACK REFUSED 并关闭；
4. 成功：
   - clientId/session 写入 Channel attribute；
   - `SessionManger.removeClient(clientId)` 收敛旧连接；
   - `sessionStore.storeSession` 保存会话（Redis）；
   - 若带 Will，注册遗嘱消息；
   - 回 CONNACK；
   - 通知设备状态：非 server/web/phone 前缀 → `DeviceCache.updateDeviceStatusCache(ONLINE)` + `MqttRemoteManager.pushDeviceStatus`（发布 `/status/post`）。

### 4.2 Session 存储

`base-server/session` 定义 `Session`、`SessionManager`、`ISessionStore`；broker 的 `SessionManger` 再包一层。在线/主题映射存在 **JVM 内存**（`ClientManager.topicMap` 等），客户端最近时间等也在内存——这是单节点设计的主要依据。

## 5. 订阅与主题匹配

### 5.1 订阅

`MqttSubscribe.handler`：

1. 提取 topic list，`TopicsUtils.validTopicFilter` 校验 `#`/`+` 合法性；
2. `ClientManager.push(topic, session)` 写内存映射；
3. 回 SUBACK；
4. 若该 topic 有 Retain 消息，立即补推。

`MqttUnsubscribe` 反向删除映射。

### 5.2 通配符匹配

`TopicsUtils.searchTopic(topic)` 枚举可能的通配订阅形式，再查 `topicMap`；`matchTopic` 支持逐级匹配。这套算法简单直观，但不做 trie 索引，主题量大时性能需评估。

## 6. PUBLISH 处理（Broker 核心）

`MqttPublish.handler` 步骤：

```text
1. 若是模拟主题(含 simulate) → MessageProducer.sendOtherMsg（进队列）
2. 否则：
   a. pubRetain：写入/更新/清理 Retain（Redis 计数）
   b. callBack：按 QoS 回 PUBACK / PUBREC，QoS2 保存消息状态
   c. sendMessageToClients：ClientManager.pubTopic 推给所有匹配订阅者
   d. sendToMQ：进入平台业务处理
   e. 累计 Redis 接收消息数
```

### 6.1 QoS 实现

- QoS0：仅转发；
- QoS1：PUBLISH 后回 PUBACK；
- QoS2：PUBREC → 对方 PUBREL → PUBCOMP，消息 ID 状态保存在 `MessageStoreImpl`；
- 离线/重复消息采用内存集合简单去重，未实现完整持久化队列。

### 6.2 Retain / Will

- `RetainMsgManager`：维护每个 topic 最后一条消息；订阅时补推；新发布清空（retain=false 时 `cleanTopic` 等逻辑）；
- `WillMessageManager`：连接时登记遗嘱，异常断开/超时时 `pop` 发布。

### 6.3 转发实现要点

`ClientManager.pubTopic` 会先过滤不在线的 clientId（基于最后 ping 时间），避免向僵尸连接发消息；`ResponseManager.publishClients` 实际写回 Channel。

## 7. 消息计数与统计

Redis key：

```text
message:send:total / message:send:today
message:receive:total / message:receive:today
message:subscribe:total
message:retain:total
message:topic:total
```

前端 “Netty 管理 → MQTT 统计”读取这些计数。

## 8. 与业务层的两个接口

```text
IDeviceReportMessageService：设备上报解析入口（属性上报）
IDataHandler：数据落业务（reportData/reportEvent/reportDevice）
IMqttMessagePublish：平台下行发布接口
```

上行属性消息在 `MqttPublish.sendToMQ` 中先执行规则脚本，再由 `parseReportMsg` 解析，最终调 `dataHandler.reportData`。下行由 `MqttMessagePublishImpl` 调 `PubMqttClient.publish` 回到 Broker。

## 9. 单机边界与集群化改造点

仓库默认是单节点：

| 状态/能力 | 存储位置 | 集群化影响 |
| --- | --- | --- |
| 订阅关系 topicMap/clientTopicMap | JVM 内存 | 节点间不共享，消息可能只在某节点投递 |
| Session | Redis（ISessionStore） | 需设计会话归属路由 |
| Retain/Will | JVM 内存 | 重启即丢，需持久化 |
| QoS2 状态 | JVM 内存 | 同上 |
| Redis pub/sub（redischannel） | Redis | 可用但无确认/持久化 |

仓库已为“消息总线”预留 `cluster.type=redis/rocketmq` 等设计位置，但代码中 RedisChannel 消费类体基本为空，说明官方并未在本开源版完整实现集群。学习价值在于理解“先跑通单机、再抽象总线”的演进路径。

## 10. 代码走查清单

想快速验证 Broker 行为，按顺序打断点：

1. `MqttMessageAdapter.channelRead0`
2. `MqttMessageDelegate.process`
3. `MqttConnect.handler`
4. `MqttPublish.handler`
5. `ClientManager.pubTopic`
6. `MqttPublish.sendToMQ`

下一章：[设备接入与消息上行链路](5-设备接入与消息上行链路.md)
