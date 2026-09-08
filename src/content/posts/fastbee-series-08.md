---
title: "[FastBee·系列] 08 规则引擎让数据流动起来"
published: 2026-09-08
description: "前面讲的是“一条数据安全入库”。但物联网真正有价值的是数据驱动业务："
tags: [FastBee,FastBee·系列文章,规则引擎]
category: FastBee·系列文章
draft: false
slug: fastbee-series-08
---
# 08 规则引擎让数据流动起来

## 数据到了平台之后

前面讲的是“一条数据安全入库”。但物联网真正有价值的是**数据驱动业务**：

- 温湿度超过阈值 → 开风机；
- 设备上报私有格式 → 自动转换成平台格式；
- 特定事件 → 通知/入库/推送到第三方系统。

这正是规则引擎的工作。

## FastBee 的规则引擎选型

FastBee 没有自研规则引擎，而是采用 **LiteFlow**：

- LiteFlow 用 EL 表达式定义“规则链”；
- 每个节点可以是普通组件、布尔组件、脚本组件；
- 规则与脚本可以存放在**数据库**，由 LiteFlow 加载。

把规则存数据库是产品化很关键的一步：管理员可以在网页改规则，不用重新发版。

## 三类脚本概念

`iot_script.script_purpose`：

```text
1 = 数据流脚本：设备数据进业务前改写
2 = 触发器：判断条件是否满足
3 = 执行动作：下发、通知等
```

以 SQL 里内置的“消息转发规则”为例（purpose=1）：

```groovy
// 设备原始 payload 是 {"temperature":25}
JSONObject jsonObject = JSONUtil.parseObj(payload);
jsonObject.keySet().forEach(key -> {
    newObject.put("id", key);
    newObject.put("value", jsonObject.getStr(key));
    newArray.add(newObject);
});
sysPayload = newArray.toString();

msgContext.setTopic(sysTopic);
msgContext.setPayload(sysPayload);
```

设备发 `{"temperature":25}`，经过脚本变成平台认识的 `[{"id":"temperature","value":"25"}]`。

这就是“协议脚本化”：**新增一种私有协议时，不一定要写 Java，可以写 Groovy**。

## 脚本在消息链中的位置

上行（设备→平台）：

```java
// MqttPublish.sendToMQ
MsgContext context = ruleProcess.processRuleScript(
        reportBo.getSerialNumber(), 1, topicName, new String(source));
if (context 改写成功) {
    reportBo.setTopicName(context.getTopic());
    reportBo.setData(context.getPayload());
}
```

下行（平台→设备）：

```java
// MqttMessagePublishImpl.funcSend
MsgContext context = ruleProcess.processRuleScript(
        bo.getSerialNumber(), 2, topicName, payload);
```

所以规则脚本是“网关模式”的过滤器：上下行都能改写。

## RuleProcess 做了什么

```java
public MsgContext processRuleScript(String serialNumber, int event,
                                    String topic, String payload) {
    // 1. 查产品信息（带 Redis 缓存）
    // 2. 查该产品该事件的启用数据流脚本
    // 3. 为每个脚本动态建 LiteFlow 链
    // 4. 执行，返回 MsgContext
}
```

没有脚本时返回空 `MsgContext`，调用方继续走原逻辑，**规则引擎不阻断默认行为**。

## 场景联动（更高一层）

场景 = 触发器 + 动作：

```text
IF(OR(T1,T2), THEN(A1,A2))
```

`SceneServiceImpl` 把网页配置转换成：

1. 触发器脚本（source=设备/定时/产品）；
2. 动作脚本（source=设备下发等）；
3. `iot_scene.el_data` 存 EL；
4. 动态注册到 LiteFlow。

定时触发器通过 `iot_device_job` 交给 Quartz，到点触发动作。

## 当前状态的客观评价

读完代码后的评价：

✅ **数据流脚本链路是完整可用的**：上下行都调用，脚本表 + LiteFlow + SQL 种子示例齐全；
⚠️ **场景联动框架在但细节待完善**：部分告警/通知代码被注释，执行细节需要自测；
⚠️ **脚本调试靠日志**：`FlowLogExecutor` 打印 script/scene 日志，缺少断点式可视化。

## 实践任务

1. 在产品 41 下建一个“温度×2”的数据流脚本；
2. 用 MQTTX 发 `[{"id":"temperature","value":"10"}]`；
3. 查日志/数据库，观察温度是否变成 20；
4. 再建一个语法错误脚本，验证单条脚本失败不影响后续消息。

下一站：[视频监控设备如何接入](/posts/fastbee-series-09/)

