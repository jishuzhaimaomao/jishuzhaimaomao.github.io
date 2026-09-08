---
title: "03 自研 MQTT Broker：从“能用”到“懂它”"
published: 2026-09-08
description: "很多 IoT 平台用 EMQX/Mosquitto。FastBee 选择 Netty 手写 MQTT Broker，对产品的好处是“少装一个组件”，对学习者的好处是——你能看到 MQTT Broker 的全部实现。"
tags: [FastBee,MQTT]
category: FastBee
draft: false
slug: fastbee-series-03
---
# 03 自研 MQTT Broker：从“能用”到“懂它”

## 为什么自研 Broker

很多 IoT 平台用 EMQX/Mosquitto。FastBee 选择 Netty 手写 MQTT Broker，对产品的好处是“少装一个组件”，对学习者的好处是——**你能看到 MQTT Broker 的全部实现**。

## Broker 的最小工作清单

一个可用 Broker 至少要处理：

1. 接入：Netty TCP/WebSocket；
2. 协议解析：MQTT 报文编解码；
3. 认证：CONNECT 时校验 clientId/账号密码；
4. 会话：记录谁在线、客户端属性；
5. 订阅：topic ↔ session 映射，支持 `+`/`#`；
6. 转发：收到 PUBLISH 推给匹配订阅者；
7. QoS：0/1/2 的确认时序；
8. 扩展：Retain、Will、心跳超时。

FastBee 用一个 `@Process` 注解优雅地实现了“报文类型到处理类”的分发：

```java
// MqttMessageDelegate
handlers.forEach(handler -> {
    Process annotation = handler.getClass().getAnnotation(Process.class);
    processor.put(annotation.type(), handler);
});
```

之后给每个 `MqttMessageType` 写一个类即可，新增报文类型时非常清爽。

## CONNECT：一台设备如何“合法上线”

`MqttConnect` 的流程里最值得学习的是**三端统一认证**：

```text
server*   → 平台内部账号密码
web*/phone* → password 里带 JWT
其他      → 设备认证 clientId = S|E&编号&产品ID&用户ID
```

设备认证最终回到 `ToolServiceImpl.clientAuth`，因为“认证”本质需要查产品表和设备表，而它们不在 Broker 模块而在 IoT 服务模块。这也解释了一个架构判断：**Broker 与业务同进程时，认证逻辑可以复用 service，不需要 HTTP 调用**。

## PUBLISH：一条消息的五个动作

`MqttPublish.handler` 依次做：

```text
1. pubRetain       保留消息更新
2. callBack        ACK 客户端（QoS 分级）
3. sendMessageToClients  转发给其他订阅者
4. sendToMQ        交给平台业务处理
5. Redis 计数      消息统计
```

注意第 3、4 步并存：Broker 不只是“收到消息转业务”，还要**先完成 MQTT 语义的分发**，再处理平台语义。设备上报时，浏览器可能已经订阅了 `+/+/ws/service` 等主题。

## QoS2 的完整状态

QoS2 需要四段握手：

```text
PUBLISH → PUBREC → PUBREL → PUBCOMP
```

`MessageStoreImpl` 用内存 Map/Set 保存消息 ID 状态，注释写着 “TODO 后续 Redis 处理”。读到这里你会真正理解：**开源版是单机内存设计**。

## 心跳与“离线”判定

MQTT 的 DISCONNECT 是礼貌告别，真实网络更多是突然消失。FastBee 的做法：

- Netty `IdleStateHandler` 超时关闭通道；
- `ClientManager.updatePing` 记录每次活跃时间；
- 发布前用 `validClient` 判断是否超时；
- `SessionManger.removeClient` 时更新设备状态为离线并推 `/status/post`。

## 从 Broker 学会的设计思想

1. **按报文类型注册处理器**：策略模式 + 注解，比巨型 switch 易扩展；
2. **协议与业务分层**：Broker 只解决 MQTT 语义，业务通过接口进入；
3. **先单机后集群**：明确把状态放在内存，承认边界，再逐步抽象到 Redis/消息总线；
4. **兼容外部 Broker**：保留 EMQX auth/webhook 实现，让系统可插拔。

## 深入阅读

代码路径：

```text
handler/adapter/MqttMessageAdapter.java
handler/adapter/MqttMessageDelegate.java
handler/MqttConnect.java
handler/MqttPublish.java
manager/ClientManager.java
manager/SessionManger.java
service/impl/MessageStoreImpl.java
```

下一站：[一条上报数据的旅程](04-一条上报数据的旅程.md)
