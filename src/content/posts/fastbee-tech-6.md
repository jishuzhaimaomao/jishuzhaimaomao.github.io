---
title: "指令下发与设备影子"
published: 2026-09-08
description: "Web / App"
tags: [FastBee,设备与产品,系统架构]
category: FastBee
draft: false
slug: fastbee-tech-6
---
# 指令下发与设备影子

## 1. 下发链路总览

```text
Web / App
  └─ POST /iot/runtime/service/invoke
       body: { serialNumber, productId, identifier,
               remoteCommand: { switch: 1 } }
  └─ DeviceRuntimeController
       └─ IFunctionInvoke.invokeNoReply(InvokeReqDto)
            ├─ 拷贝为 MQSendMessageBo
            ├─ snowflake 生成 messageId
            └─ mqttMessagePublish.funcSend(bo)
  └─ MqttMessagePublishImpl.funcSend
       ├─ 影子模式？ 写影子值并走 reportData（不上行下发）
       ├─ 查询产品 protocolCode/transport
       ├─ 记录 FunctionLog（funType=1，指令下发）
       ├─ 组装 DeviceDownMessage（messageId/body/identifier/slaveId...）
       ├─ 按 ServerType 分发
       │    MQTT → buildMessage → TopicType.FUNCTION_GET
       │           → 规则脚本(event=2，可改写 topic/payload)
       │           → PubMqttClient.publish
       └─ 更新 FunctionLog 状态（默认“无应答”）
```

关键类：

```text
DeviceRuntimeController      REST 入口
IFunctionInvoke/Impl         service 调用抽象（目前只有 no-reply）
IMqttMessagePublish/Impl     真正的发布实现
DeviceDownMessage            下发消息模型（MQTT/MODBUS 字段混合）
InstructionsMessage          最终 “topic + byte[]”
PubMqttClient                内部发布客户端
```

## 2. 产品传输协议的作用

`Product.transport` 决定下发分支：

- `MQTT`：完整可用；
- `GB28181`：监控设备，走 SIP 命令而非 MQTT；
- COAP/TCP/UDP：`ServerType` 枚举存在，但当前 `funcSend` 的 switch 只实现 MQTT 分支。

因此新增传输协议时，重点扩展 `MqttMessagePublishImpl.funcSend` 的分支与协议编码器。

## 3. 影子（Shadow）模式

### 3.1 什么是影子

平台保存两个值：

- `value`：设备最近上报的实际值；
- `shadow`：平台保存的“期望值/目标值”。

设备离线时下发指令，直接写入 shadow；设备重新上线（或平台主动同步）时再推送给设备。

### 3.2 代码中的体现

`reportDeviceThingsModelValue(input, type, isShadow)`：

```java
if (isShadow) {
    valueItem.setShadow(value);          // 只更新影子
} else {
    valueItem.setValue(value);
    valueItem.setShadow(value);          // 实际值同时当影子
    valueItem.setTs(new Date());
}
```

`funcSend` 开头：

```java
if (bo.getIsShadow()) {
    // 不真正下发，把值当影子上报入库
    dataHandler.reportData(...); return;
}
```

影子同步相关接口/类：

- `DeviceServiceImpl.getDeviceShadowThingsModel`
- `FirmwareCacheImpl`（固件缓存，非升级完整实现）
- 外部 EMQX webhook 的 `client_connected` 分支：影子模式设备上线后把属性和功能保留消息推给设备。

### 3.3 设备重新上线如何收到数据

正常流程设计是：

1. 设备 CONNECT；
2. 平台发布 `/status/post`（含 isShadow）；
3. 若启用影子，平台通过 Retain 主题/主动发布属性功能；
4. 设备订阅 `function/get`、`property/get` 收到最新期望值。

内置 Broker 下 `MqttRemoteManager.pushDeviceStatus` 已实现状态发布；属性/功能的下发依赖设备自身的订阅，这与 SDK 行为需要配合验证。

## 4. 功能回执与状态机

`FunctionLog` 记录下发：

- `fun_type`：1=平台下发功能、2=属性读取等；
- `result_code / result_msg`：回执状态。

`FunctionReplyStatus` 枚举包含 NO_REPLY、SUCCESS、FAIL 等。当前 `publish` 在无异常时直接记 `NO_REPLY`，并未实现等待设备 `/service/reply` 后更新——是开源版 TODO（Controller 的 `reply` 方法同样标记 TODO）。

## 5. 定时下发与场景动作

`DeviceJob`（设备定时）由 Quartz 调度，执行器 `fastbee-open-api/data/quartz/JobInvokeUtil` 解析 `actions` JSON：

```java
action.type == 1 → publishProperty(...)
action.type == 2 → publishFunction(...)
```

动作支持属性读取与功能下发，供“设备定时任务/定时控制”使用；场景联动的定时触发器也复用 `DeviceJob`（job_type=3）。

## 6. 属性读取（下发 get）

Topic 上平台可以发 `/property/get` 请求设备上报当前值；设备收到后回 `/property/post`，复用上报链路入库。读取指令在 `TopicType.PROPERTY_GET` 中定义，`pubMqttClient`/工具接口可发起。

## 7. 测试建议（对接代码）

验证一次下发：

1. 设备连接并订阅 `function/get`；
2. Web 端发送 `switch=1`；
3. 抓包/日志确认 topic=`/product/SN/function/get`，payload=`[{"id":"switch","value":"1"}]`；
4. 查看 `iot_function_log` 有“无应答”记录；
5. 设备执行后发布 `service/reply`，验证是否会更新回执（当前预期：暂不更新，需二次开发补齐）。

## 8. OTA（预留）

公共模型：

```text
OtaUpgradeBo/OtaReplyMessage/OtaUpgradeDelayTask
TopicType.FIRMWARE_SET(/upgrade/set)、FIRMWARE_UPGRADE_REPLY(/upgrade/reply)
IFirmwareCache 固件地址缓存
```

但 `MqttMessagePublishImpl.upGradeOTA` 方法体为空，`buildMessage(OtaUpgradeBo)` 虽然存在但未被调用。因此**OTA 主流程在开源版未闭合**，验收前应明确范围。

下一章：[物模型与数据存储设计](/posts/fastbee-tech-7/)
