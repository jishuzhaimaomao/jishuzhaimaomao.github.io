---
title: "规则引擎与定时任务"
published: 2026-09-08
description: "FastBee 的“规则引擎”分两个层次："
tags: [FastBee,规则引擎]
category: FastBee
draft: false
slug: fastbee-tech-8
---
# 规则引擎与定时任务

## 1. 总体定位

FastBee 的“规则引擎”分两个层次：

1. **数据流脚本**：设备上下行时对消息做改写/过滤（`iot_script`，purpose=1）；
2. **场景联动（触发器 + 动作）**：`iot_scene` 驱动，包含设备触发、定时触发、动作执行（当前 UI/代码完整度有限）。

底层是 **LiteFlow**（规则链框架）+ 数据库表动态构建脚本节点 + Groovy 等脚本语言。

## 2. 表结构

### 2.1 iot_script（脚本）

关键字段：

| 字段 | 含义 |
| --- | --- |
| `script_event` | 1 设备上报、2 平台下发、3 设备上线、4 设备离线 |
| `script_purpose` | 1 数据流、2 触发器、3 执行动作 |
| `script_action` | 1 消息重发、2 消息通知、3 HTTP 推送、4 MQTT 桥接、5 数据库存储 |
| `script_type` | script / if_script / for_script / while_script / switch_script 等 |
| `script_language` | groovy / qlexpress / js / python / lua / aviator / java |
| `script_data` | 脚本源码 |
| `enable` | 是否生效 |

### 2.2 iot_scene（规则链）

`el_data` 存 LiteFlow EL（如 `IF(OR(T1,T2),THEN(A1,A2))`），`chain_name` 是链名。应用名统一为 `fastbee`，与 LiteFlow SQL 数据源配置一致。

### 2.3 iot_scene_script / iot_scene_device

场景与其“触发器/动作”的明细关联：每个场景脚本一行，包含 source（1 设备触发、2 定时、3 产品触发）、触发条件、物模型信息、设备清单。

## 3. 数据流脚本如何执行

入口是 `RuleProcess.processRuleScript(serialNumber, event, topic, payload)`：

1. 从 Redis/DB 查产品（protocolCode）；
2. 按 `productId + scriptEvent + scriptPurpose=1` 查脚本 ID 列表；
3. 每个脚本构建 `dataChain_{scriptId}` 链并执行；
4. 脚本上下文 `MsgContext` 含 `serialNumber/productId/protocolCode/payload/topic`；
5. 脚本可修改 `msgContext.payload` / `topic`，执行后返回，调用方决定是否用新值。

内置脚本示例（SQL 种子）做“对象 JSON → 数组 JSON”转换：

```groovy
sysTopic = "/" + productId + "/" + serialNumber + "/property/post"
// 将 {"temp":1} 转成 [{"id":"temp","value":"1"}]
msgContext.setTopic(sysTopic);
msgContext.setPayload(sysPayload);
```

调用位置：

- 上行：`MqttPublish.sendToMQ`（event=1）；
- 下行：`MqttMessagePublishImpl.funcSend`（event=2）；
- EMQX 回调：`subscribeCallback.messageArrived`。

## 4. 场景联动

`SceneServiceImpl`：

- 新增/修改场景时：
  - 生成 `chainName = C{snowflake}`；
  - 根据触发器/动作构建脚本源码（模板 `sceneContext.process(json)`）；
  - `buildElData` 拼 EL：条件 AND/OR/NOT + 执行方式 THEN/WHEN；
  - 动态 `LiteFlowNodeBuilder.createScriptBooleanNode/createScriptNode` 注册到内存；
  - 定时触发器写入 `iot_device_job`（job_type=3）交给 Quartz；
- 场景数据落库后，规则链通过 LiteFlow SQL 数据源加载。

执行上下文类：

```text
fastbee-gateway/fastbee-mq/.../ruleEngine/SceneContext.java
fastbee-plugs/fastbee-ruleEngine/.../context/MsgContext.java
```

`SceneContext` 内置了设备触发匹配、静默周期、值比较等逻辑（部分告警/通知逻辑被注释），是规则脚本运行时可直接调用的上下文。

> 注意：源码中场景联动处于“框架已通、细节待完善”的状态。学习时应区分：
> - 已经完整可用的：数据流脚本、定时设备任务、动态建链；
> - 待验证/待完善的：告警推送闭环、场景执行完整链路。

## 5. 定时任务（Quartz）

### 5.1 两类任务

| 任务 | 表 | 作用 |
| --- | --- | --- |
| 系统定时任务 | `sys_job` | 若依标准：调用 Bean 方法（白名单校验） |
| 设备任务 | `iot_device_job` | 设备定时控制、告警定时、场景定时触发 |

启动 `DeviceJobServiceImpl@PostConstruct`：

```java
scheduler.clear();
jobMapper.selectJobAll()  → ScheduleUtils.createScheduleJob
sysJobMapper.selectJobAll() → 系统任务
```

### 5.2 设备任务执行

`fastbee-open-api/data/quartz/JobInvokeUtil.invokeMethod` 读取 `iot_device_job.actions` JSON：

```java
List<Action> actions = JSON.parseArray(deviceJob.getActions(), Action.class);
if (action.getType() == 1) {
    messagePublish.publishProperty(productId, serialNumber, propertys, 0);
} else if (action.getType() == 2) {
    messagePublish.publishFunction(productId, serialNumber, functions, 0);
}
```

### 5.3 cron

`cron_expression` 复用若依 CronUtils；支持立即执行、暂停、恢复、删除等调度生命周期。

## 6. 线程与日志

`fastbee-ruleEngine` 提供：

- `MainExecutorBuilder`：主链线程池（10~30，队列 1000）；
- `WhenExecutorBuilder`：WHEN 并发线程池；
- `FlowLogExecutor`：打印上下文/执行步骤，区分 `script`/`scene` 两类日志；
- `RequestIdBuilder`：请求跟踪。

## 7. 二次开发建议

给某产品加私有协议：

1. 在 `iot_script` 插入数据流脚本（device 上报/下发事件）；
2. 脚本中按 `protocolCode` 分派、转换 payload/topic；
3. 开启 `enable`；
4. 联调时看 `script` 日志文件与 `MsgContext` 输出。

下一章：[视频接入：GB28181 与 SIP](9-视频接入-GB28181与SIP.md)
