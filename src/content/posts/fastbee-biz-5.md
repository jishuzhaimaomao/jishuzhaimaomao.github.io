---
title: "[FastBee·业务篇] 典型业务流程与端到端场景"
published: 2026-09-08
description: "1. 管理员创建产品分类；"
tags: [FastBee,FastBee·业务篇,学习路线,IoT 业务]
category: FastBee·业务篇
draft: false
slug: fastbee-biz-5
---
# 典型业务流程与端到端场景

## 场景一：新建设备并完成 MQTT 接入

### 1.1 平台侧

1. 管理员创建产品分类；
2. 创建产品：选择协议 `JSON`、传输 `MQTT`、设备类型（直连/网关/监控），填写认证信息（简单认证的 mqtt 账号密码 / 加密认证密钥），**发布产品**（status=2）；
3. 在产品下添加物模型：温度（属性）、开关（功能）、异常（事件）等；
4. 可先配置“通用物模型模板”，新建产品时批量导入；
5. 添加设备：生成 `serialNumber`；或使用授权码流程预生成。

### 1.2 设备侧（模拟客户端也可）

设备 MQTT 连接参数：

```text
Broker：  mqtt://host:1883   （或 ws://host:8083/mqtt）
clientId：S&<serialNumber>&<productId>&<userId>
          E&<serialNumber>&<productId>&<userId>  （加密认证）
username：产品 mqtt_account
password：简单认证=产品 mqtt_password；加密认证=用产品密钥加密后的动态密码
```

### 1.3 预期效果

- Broker 返回 CONNACK；
- 设备状态在 Web 端从“未激活/离线”变为“在线”；
- Redis 在线列表写入 serialNumber；
- 重复使用相同 clientId 时，旧会话被移除（平台侧做会话收敛）。

## 场景二：设备周期上报属性

设备向主题发布 JSON 数组：

```text
topic:    /{productId}/{serialNumber}/property/post
payload:  [{"id":"temperature","value":"23.5"},{"id":"humidity","value":"60.1"}]
```

平台内发生：

```text
Netty Broker 收到 PUBLISH
  → 保留消息/ACK/转发订阅者
  → 规则引擎数据流脚本（可选改写 topic/payload）
  → 解析物模型值（decode）
  → DataHandler.reportData
  → DeviceServiceImpl.reportDeviceThingsModelValue
       ├── 校验物模型是否存在
       ├── 更新 Redis 缓存中的 value/shadow/ts
       └── 按 is_history 写时序/MySQL 日志
  → 通过 /ws/service 等 MQTT 主题推送给前端实时视图
```

用户在 Web 端可以立即看到：

- 设备详情实时数据（ECharts）；
- 历史记录/数据中心出现新数据；
- 事件上报进入事件日志；
- 功能回执进入功能日志（若设备应答）。

## 场景三：平台下发指令

典型链路（Web 点击“打开开关”）：

```text
前端 POST /iot/runtime/service/invoke
  body: { serialNumber, productId, identifier:"switch",
          remoteCommand: { switch:1 } }
  → IFunctionInvoke.invokeNoReply
  → MQSendMessageBo + snowflake messageId
  → IMqttMessagePublish.funcSend
      ├── 若设备开启影子：先写影子值并视为上报，不再直接下发
      ├── 查产品 protocolCode/transport
      ├── 组装 DeviceDownMessage
      ├── 规则脚本（平台下发事件 event=2，可改写）
      ├── jsonProtocolService.encode → [{"id":"switch","value":"1"}]
      └── PubMqttClient.publish 到 /{productId}/{serialNumber}/function/get
  → 记录 FunctionLog（初始结果：无应答）
```

若设备订阅了 `function/get` 并执行成功，可向 `/service/reply` 或 `/function/post` 回执；需注意当前开源版“回执 → 更新功能日志”的闭环尚未完成（`DeviceRuntimeController.reply` 等方法标注 TODO），详见技术篇指令下发章节。

## 场景四：异常掉线与心跳

1. 设备正常 DISCONNECT：`MqttDisConnect` → 移除订阅/会话 → 状态离线；
2. 网络闪断：Broker 靠 IdleStateHandler 与客户端最近 PING 时间判断超时，`SessionManger.removeClient` 触发离线；
3. 配置了遗嘱（Will）时，异常断开会代为发布遗嘱消息；
4. 平台向 `/status/post` 发布在线/离线消息（前端订阅实时感知）。

## 场景五：网关 + 子设备（MODBUS 风格采集）

FastBee 把“采集”抽象成网关下的子设备/从机：

- 网关上创建子设备：子设备记录 `gw_dev_code=网关编号`；
- 物模型可携带 `temp_slave_id`、寄存器地址、公式与解析类型；
- 上报数据中可用 `slaveId` 区分，内部编号 `serialNumber_slaveId`，物模型标识 `identifier#slaveId`；
- 下发时选择功能码（如 05 写线圈、06 写寄存器）并携带从机地址。

该链路与前端 `device/product-list`（网关产品选型）和“采集点模板”功能配套。

## 场景六：规则/场景让数据产生动作

官方 SQL 中内置了一个“消息转发规则”脚本示例：设备原始 JSON 是对象格式，脚本把它转换为平台 JSON 数组，再交回主链路。这说明了 FastBee 对**异构设备协议**的通用策略：

```text
设备私有格式
  → 产品协议脚本（LiteFlow/Groovy）
  → 统一 topic + 统一 JSON Array
  → 物模型校验与入库
```

同理，平台下发的统一指令也可以先经过“下发脚本”转成设备私有格式。

## 场景七：视频监控设备接入（GB28181）

1. 配置 SIP 域/ID/密码与媒体服务器（ZLMediaKit）；
2. 摄像头向 SIP 端口注册；
3. SIP 模块处理 Register、Catalog 查询，形成“设备-通道”树；
4. Web 播放：请求拉流 → SIP INVITE → ZLMediaKit 收流 → 前端播放 HLS/WebRTC/FLV 等；
5. 云台、录像等通过 PTZ/Record 指令下发。

## 综合练习建议

本地部署后（见技术篇部署章节），用仓库内置演示数据做一次闭环：

1. 打开“产品管理”，观察预置的 `★智能开关产品` 的物模型 JSON；
2. 添加一个测试设备；
3. 用 MQTTX 连接并发布属性数据；
4. 在设备详情里查看实时值与历史；
5. 触发一次“功能下发”，观察设备收到的 topic/payload 与功能日志。

这条练习做完，业务篇的核心就掌握了。接下来进入[技术篇](/posts/fastbee-tech-1/)。

