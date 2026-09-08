---
title: "[FastBee·技术篇] 设备接入与消息上行链路"
published: 2026-09-08
description: "TopicType 枚举是 FastBee 协议层字典，核心规则："
tags: [FastBee,FastBee·技术篇,设备与产品,系统架构]
category: FastBee·技术篇
draft: false
slug: fastbee-tech-5
---
# 设备接入与消息上行链路

## 1. Topic 规范

`TopicType` 枚举是 FastBee 协议层字典，核心规则：

```text
/<productId>/<serialNumber>/<dataType>/<action>
```

| 方向 | dataType | action | 示例 |
| --- | --- | --- | --- |
| 设备上报（平台订阅） | property | post | `/41/D1ELV3A5TOJS/property/post` |
| 设备上报事件 | event | post | `/41/D1ELV3A5TOJS/event/post` |
| 设备上报功能结果 | function | post | `/41/D1ELV3A5TOJS/function/post` |
| 设备上报信息 | info | post | `/.../info/post` |
| 时间同步 | ntp | post | `/.../ntp/post` |
| 功能回执 | service | reply | `/.../service/reply` |
| 平台下发功能 | function | get | `/.../function/get` |
| 平台下发属性读取 | property | get | `/.../property/get` |
| 状态发布 | status | post | `/.../status/post` |
| 平台推送前端 | ws | service | `/.../ws/service` |
| 模拟设备 | property | get/set/simulate | `/.../property/get/simulate` 等 |

订阅侧使用通配：订阅端关注 `+/+/property/post`；发布侧是具体 `/{productId}/{serialNumber}/...`。

### Topic 解析工具

`TopicsUtils` 提供 `buildTopic`、`parseSerialNumber`、`parseProductId`、`parseTopicName`、`getThingsModel`。设备编号从 topic 段中“按长度 >9”识别——这是 FastBee 的兼容性技巧，也意味着编号长度要遵守约定。

## 2. 上行数据处理的两个入口

### 2.1 内置 Broker 直通（默认 enabled=true）

```text
设备 PUBLISH
  → MqttPublish.sendToMQ
  → 仅当 topic 以 /post 结尾
  → ruleProcess.processRuleScript(serialNumber, 1, topic, payload)
  → 规则脚本可能改写 topic/payload
  → 若含 "property"：parseReportMsg
```

`DeviceReportMessageServiceImpl.parseReportMsg`：

1. `buildReport`：查设备、取 productId、识别物模型类型；
2. `jsonProtocolService.decode`：把 payload 解成 `ThingsModelSimpleItem` 列表；
3. `handlerReportMessage`：校验 topic 后缀为 `/property/post` 或 `/property/simulate/post`；
4. 组装 `ReportDataBo` → `dataHandler.reportData`。

### 2.2 外部 EMQX 兼容（enabled=false）

```text
EMQX webhook → /iot/tool/mqtt/webhook(v5)
内部 PubMqttClient 订阅 +subscribeCallback
  → DeviceOtherQueue
  → DeviceOtherMsgHandler.messageHandler（info/event/function/...）
```

> 结论：开源版“完全验证且直连”的上行主链路是 **property/post**。event/function/info 等在 EMQX 模式下由队列处理器实现；内置 Broker 模式下这几类消息的消费链路并不完整，二次开发时建议补齐到 `IDeviceReportMessageService` 或队列入口。

## 3. 内置 JSON 协议编解码

`JsonProtocolService`（fastbee-protocol-collect）：

decode（设备→平台）：

```java
String data = new String(deviceData.getData(), UTF_8);
List<ThingsModelSimpleItem> values = JSON.parseArray(data, ThingsModelSimpleItem.class);
```

即默认设备上报的是 **JSON 数组**：

```json
[{"id":"temperature","value":"23.5"},{"id":"humidity","value":"60"}]
```

encode（平台→设备）：

```json
[{"id":"switch","value":"1"}]
```

官方 SQL 中的“消息转发规则”脚本演示了如何把**对象格式**（`{"temperature":23.5}`）转成数组格式再交给平台。

## 4. 属性数据入库核心逻辑

`DeviceServiceImpl.reportDeviceThingsModelValue`（约 223~380 行）逐项处理：

```text
for item in 上报项:
  identity = item.id
  若 array_ 前缀：截断取真实 id
  若 slaveId 非空：identity += "#"+slaveId

  查物模型（单点），不存在 → 跳过

  Redis 缓存 ValueItem（key: TSLV:{productId}_{SN}）
  - 数组：按下标更新 value/shadow
  - 普通：更新 value/shadow/ts

  按模型类型分支：
    PROP(属性): 若 is_history=1 → DeviceLog(logType=1)
    SERVICE(功能): 若 is_history=1 → FunctionLog(funType 等)
    EVENT(事件): → DeviceLog(logType=3) 或 EventLog

  最终：
    logService.saveDeviceLog(...)（每 1ms 间隔，规避 TDengine 主键时间冲突）
```

Redis 缓存 Hash 的 value 是 JSON 字符串（`ValueItem`），包括当前值、影子值、时间戳、备注等。

## 5. 实时推送到前端

`DataHandlerImpl.reportData` 落库后：

```java
PushMessageBo messageBo = new PushMessageBo();
messageBo.setTopic(topicsUtils.buildTopic(productId, serialNumber, TopicType.WS_SERVICE_INVOKE));
messageBo.setMessage(JSON.toJSONString(result));
remoteManager.pushCommon(messageBo);
```

也就是把最新结果发布到 `/ws/service`，前端浏览器通过 MQTT over WebSocket 订阅后实时刷新。

## 6. 事件/信息上报处理

EMQX 队列模式中 `DeviceOtherMsgHandler`：

- `info` → `reportDevice`：设备上报位置/RSSI/摘要并刷新状态；
- `ntp` → 回复时间同步；
- `function` → 当功能上报回执处理；
- `event` → 落事件日志；
- `property-offline/function-offline` → 影子数据（离线补报）。

## 7. 常见联调样例

假设产品 41、设备编号 `D1ELV3A5TOJS`：

### 属性上报

```text
topic: /41/D1ELV3A5TOJS/property/post
qos: 1
payload:
[{"id":"temperature","value":"26.3","ts":"2026-09-08 10:00:00"},
 {"id":"humidity","value":"58.2"}]
```

预期：

- 设备详情出现实时曲线；
- Redis `TSLV:41:D1ELV3A5TOJS` 更新（Hash 的 field 为物模型标识）；
- `iot_device_log`/TDengine 新增 temperature/humidity 记录；
- WebSocket 收到 `/ws/service` 消息。

### 事件上报

```text
topic: /41/D1ELV3A5TOJS/event/post
payload: [{"id":"exception","value":"设备温度过高","remark":"alarm"}]
```

预期：事件日志（logType=3）新增记录；若物模型 exception 未定义会被跳过。

## 8. 调试验证工具

- 前端“Netty 管理 → 客户端”：看设备是否在线；
- `MQTTX`/`mosquitto_pub`：模拟发布；
- `/iot/tool/getTopics`：查看全部平台订阅/下发 Topic；
- `/iot/tool/decode`：测试 CRC/报文（MODBUS 工具）。

下一章：[指令下发与设备影子](/posts/fastbee-tech-6/)

