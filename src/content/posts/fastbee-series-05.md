---
title: "[FastBee·系列] 05 平台如何控制设备"
published: 2026-09-08
description: "设备上报是“单向广播”，就算丢一条损失也不大。控制指令是“反向单播”，要面对这些问题："
tags: [FastBee,FastBee·系列文章,设备与产品]
category: FastBee·系列文章
draft: false
slug: fastbee-series-05
---
# 05 平台如何控制设备

## 控制比上报更难

设备上报是“单向广播”，就算丢一条损失也不大。控制指令是“反向单播”，要面对这些问题：

1. 设备在不在线？
2. 指令发到哪个 topic、什么格式？
3. 要不要等设备确认？
4. 设备离线怎么办？
5. 谁有权限控制这台设备？

## 一次“打开开关”的调用链

前端点按钮 → `POST /iot/runtime/service/invoke`：

```json
{
  "serialNumber": "D1ELV3A5TOJS",
  "productId": 41,
  "identifier": "switch",
  "remoteCommand": {"switch": 1}
}
```

`DeviceRuntimeController` 组装成 `InvokeReqDto`，交给 `IFunctionInvoke.invokeNoReply`：

- 生成 messageId（雪花算法）；
- 转成 `MQSendMessageBo`；
- 调 `mqttMessagePublish.funcSend(bo)`。

注意接口名是 **NoReply**：这个版本先实现“发出去即返回”，不等待设备回执。

## funcSend 的分岔路口

`MqttMessagePublishImpl.funcSend` 先看设备影子：

```java
if (bo.getIsShadow()) {
    // 影子模式：不直接下发，把值当上报写入
    dataHandler.reportData(dataBo);
    return;
}
```

非影子模式继续：

```java
DeviceDownMessage downMessage = DeviceDownMessage.builder()
        .messageId(...)
        .body(bo.getValue())
        .serialNumber(...)
        .serverType(serverType)
        .build();

switch (serverType) {
    case MQTT:
        InstructionsMessage instruction = buildMessage(downMessage, TopicType.FUNCTION_GET);
        // 规则引擎有机会改写
        publish(instruction.getTopicName(), instruction.getMessage(), funcLog);
        break;
}
```

## 编码与主题

`buildMessage`：

1. 查产品 protocolCode/transport；
2. 拼 topic：

```text
/41/D1ELV3A5TOJS/function/get
```

3. 交给协议编码器：

```java
byte[] data = jsonProtocolService.encode(encodeData, null);
// 结果示例：[{"id":"switch","value":"1"}]
```

## 谁把消息发给设备

有意思的是，下发不是直接写设备 Channel，而是通过**平台自己的 MQTT 客户端** `PubMqttClient`：

```java
mqttClient.publish(pushMessage, topic, false, 0);
```

它连的还是本地 Broker（`tcp://127.0.0.1:1883`）。这个设计把“平台下发”和“设备上报”统一到 MQTT 语义中：设备订阅了 `/function/get`，Broker 把平台发布的消息路由给它。

## 功能日志

每次下发都会记录 `FunctionLog`：

```java
log.setResultMsg(FunctionReplyStatus.NORELY.getMessage());
functionLogService.insertFunctionLog(log);
```

前端“功能日志”可以看到：什么时间、给谁、下了什么指令、是否得到应答。

## 当前版本的诚实提醒

开源版控制链的完成度：

| 环节 | 状态 |
| --- | --- |
| 在线设备下发 MQTT | 已实现 |
| 离线影子写入 | 已实现 |
| 功能日志“无应答”记录 | 已实现 |
| 设备 `/service/reply` 更新回执 | **未实现（TODO）** |
| 属性读取 `/property/get` | 主题已定义，配合 SDK 验证 |
| OTA `/upgrade/set` | **主方法为空，未闭环** |

对测试/二开来说，这是明确的机会点：补齐“回执匹配 messageId → 更新 FunctionLog”是很有价值的增量功能。

## 怎么验证一次下发

用 MQTTX 开两个连接：

- 设备连接（订阅 `/41/D1ELV3A5TOJS/function/get`）；
- 浏览器/Web 连接（调用接口触发）。

观察设备连接是否收到：

```text
topic:   /41/D1ELV3A5TOJS/function/get
payload: [{"id":"switch","value":"1"}]
```

下一站：[设备影子与在线状态管理](/posts/fastbee-series-06/)

