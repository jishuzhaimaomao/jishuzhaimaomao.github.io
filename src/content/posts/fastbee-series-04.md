---
title: "[FastBee·系列] 04 一条上报数据的旅程"
published: 2026-09-08
description: "假设一台温湿度计每分钟发布一次数据。跟着 temperature=25.6 从网线出发，看看它在 FastBee 里走了多远。"
tags: [FastBee,FastBee·系列文章,学习笔记]
category: FastBee·系列文章
draft: false
slug: fastbee-series-04
---
# 04 一条上报数据的旅程

假设一台温湿度计每分钟发布一次数据。跟着 `temperature=25.6` 从网线出发，看看它在 FastBee 里走了多远。

## 第 0 站：设备连接

设备 MQTT CONNECT：

```text
clientId: S&D1ELV3A5TOJS&41&1
```

Broker 校验通过后，`SessionManger.buildSession` 把设备置为在线，并把状态同步到 Redis 和 MySQL。设备随后订阅：

```text
/41/D1ELV3A5TOJS/function/get     ← 准备接收平台指令
```

## 第 1 站：PUBLISH 进入 Broker

设备发布：

```text
topic:   /41/D1ELV3A5TOJS/property/post
payload: [{"id":"temperature","value":"25.6"}]
```

Netty 解码出 `MqttPublishMessage`，`MqttMessageAdapter` 交给 `MqttMessageDelegate`，命中 `MqttPublish`。

Broker 先完成 MQTT 语义：

- QoS1 回 PUBACK；
- 更新保留消息；
- 把消息转发给所有匹配订阅者（此刻没有其他设备订阅这个 topic，但浏览器订阅的 `/ws/service` 稍后会收到平台推送）。

然后 `sendToMQ` 判断这是“上报”（topic 以 post 结尾），开始平台处理。

## 第 2 站：规则引擎插一脚

`RuleProcess.processRuleScript(serialNumber, 1, topic, payload)` 按产品查找数据流脚本。

如果该产品有 Groovy 脚本，它可能改写 payload：例如把设备私有格式转成平台 JSON 数组。脚本有机会：

- 改 topic（例如把子设备消息转成真实子设备编号）；
- 改 payload；
- 什么都不改直接放行。

## 第 3 站：解析成物模型值

消息进入 `DeviceReportMessageServiceImpl.parseReportMsg`：

1. `buildReport` 从 DB 查设备，取 productId；
2. `JsonProtocolService.decode` 把 JSON 数组解成：

```java
ThingsModelSimpleItem(id="temperature", value="25.6")
```

## 第 4 站：业务层校验与缓存

`DataHandlerImpl.reportData` 调 `DeviceServiceImpl.reportDeviceThingsModelValue`。

这一站决定了数据的生死：

```java
PropertyDto dto = thingsModelService.getSingleThingModels(productId, identity);
if (null == dto) continue;   // 产品没定义这个物模型 → 丢弃
```

通过后写入 Redis Hash：

```text
key:   TSLV:41:D1ELV3A5TOJS
field: temperature
value: {"id":"temperature","value":"25.6","shadow":"25.6","ts":"..."}
```

同时检查该物模型 `is_history`，决定是否进入下一步。

## 第 5 站：历史落库

`logService.saveDeviceLog(deviceLog)`：

- 默认实现 `MySqlLogServiceImpl` → 插入 `iot_device_log`；
- 启用 TDengine 时 → `TdengineLogServiceImpl` → 写时序库超级表。

代码里有个细节：连续多条日志的 ts 会“每条间隔 1ms”，注释写的是“避免 TDengine 时间冲突”——时序库主键通常包含时间，同毫秒会撞主键。

## 第 6 站：实时推给前端

落库成功后：

```java
PushMessageBo messageBo = new PushMessageBo();
messageBo.setTopic(topicsUtils.buildTopic(productId, serialNumber, TopicType.WS_SERVICE_INVOKE));
messageBo.setMessage(JSON.toJSONString(result));
remoteManager.pushCommon(messageBo);
```

浏览器 Web 端订阅 `/41/D1ELV3A5TOJS/ws/service`，收到后更新 ECharts 曲线。

## 旅程地图

```text
设备 → MqttPublish
     → RuleProcess（可选改写）
     → DeviceReportMessageServiceImpl（decode）
     → DataHandlerImpl.reportData
     → DeviceServiceImpl.reportDeviceThingsModelValue
         ├─ 物模型校验
         ├─ Redis 当前值/影子
         └─ ILogService 历史存储
     → MqttRemoteManager 推 /ws/service
     → 前端曲线
```

## 你可以打断点验证的地方

建议按顺序打断点：

1. `MqttPublish.sendToMQ`
2. `RuleProcess.processRuleScript`
3. `DeviceReportMessageServiceImpl.handlerReportMessage`
4. `DataHandlerImpl.reportData`
5. `DeviceServiceImpl.reportDeviceThingsModelValue`
6. `MySqlLogServiceImpl.saveDeviceLog`

下一条跟着“控制指令”走：[平台如何控制设备](/posts/fastbee-series-05/)

