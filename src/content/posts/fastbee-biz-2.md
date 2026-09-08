---
title: "核心概念：产品、设备与物模型"
published: 2026-09-08
description: "FastBee 的核心建模链路是："
tags: [FastBee,物模型,设备与产品]
category: FastBee
draft: false
slug: fastbee-biz-2
---
# 核心概念：产品、设备与物模型

FastBee 的核心建模链路是：

```text
产品分类(Category)
   └── 产品(Product) ──可绑定── 通用物模型模板(ThingsModelTemplate)
          │  1:N
          ├── 物模型(ThingsModel)：属性/服务(功能)/事件
          ├── 授权码(ProductAuthorize)
          └── 设备(Device)：直连设备 / 网关 / 监控设备
                 ├── 子设备（通过 gw_dev_code 挂在网关上）
                 └── 设备用户/分享（device_user）
```

## 1. 产品分类与产品

### 1.1 分类 `iot_category`

产品必须归属于一个分类。分类表在初始化 SQL 中已有电工照明、家居安防等示例。分类主要是**组织与检索**用途，不参与设备通信。

### 1.2 产品 `iot_product`

产品描述“一类设备长什么样、怎么连、怎么认证”，核心字段：

| 字段 | 含义 |
| --- | --- |
| `protocol_code` | 协议编号。内置 `JSON`（系统协议表 `iot_protocol` 中注册） |
| `transport` | 传输协议：`MQTT`、`GB28181`、`COAP/TCP/UDP/WEBSOCKET`（后几种枚举存在但开源主链路是 MQTT） |
| `device_type` | 1 直连设备、2 网关设备、3 监控设备 |
| `status` | 1 未发布、2 已发布。**未发布产品的设备无法通过 MQTT 认证** |
| `mqtt_account/mqtt_password` | 设备连接 Broker 的简单认证账号/密码 |
| `mqtt_secret` | 加密认证使用的产品密钥（HMAC/对称加密材料） |
| `is_authorize` | 是否启用“设备授权码”二次校验 |
| `things_models_json` | 产品物模型 JSON（properties/functions/events） |
| `template_id`/关联表 | 可从“通用物模型模板”快速复制物模型 |

### 1.3 产品授权码 `iot_product_authorize`

授权码用于“先开码、后激活”的硬件销售/交付场景：平台生成授权码，用户烧录/绑定时提交授权码；`ToolServiceImpl.authCodeProcess` 校验授权码并把它与设备编号绑定。未绑定时设备会自动注册（`insertDeviceAuto`）。

## 2. 设备

### 2.1 设备编号与类型

设备表 `iot_device` 唯一键是 `serial_number`。设备编号由平台生成（`DeviceServiceImpl.generationDeviceNum`，以 `D` 开头的字母数字串），也可以由硬件自行指定并在认证时自动入库。

设备类型直接决定业务展示与网关逻辑：

- **直连设备**：独立连接 Broker；
- **网关设备**：可以挂载多个子设备；
- **监控设备**：GB28181 摄像头/NVR 等，由 SIP 模块管理。

子设备本身也是一条 `iot_device` 记录，通过 `gw_dev_code` 指向网关的 `serial_number`；上报/下发的数据标识使用 `子设备serialNumber_slaveId`、物模型 `identifier#slaveId` 表达从机维度（MODBUS 场景常见）。

### 2.2 设备状态机

状态字段 `status`（`DeviceStatus` 枚举）：

| 值 | 状态 | 触发 |
| --- | --- | --- |
| 1 | 未激活 | 新建设备、认证前 |
| 2 | 禁用 | 管理员停用（认证会被拒绝） |
| 3 | 在线 | MQTT CONNECT 成功/心跳正常 |
| 4 | 离线 | DISCONNECT、心跳超时、异常断开 |

在线/离线不是简单在收到 DISCONNECT 时翻转：Broker 端 `SessionManger` 通过“最近一次 PING/消息时间”判断心跳是否超时，超时才判定离线。状态同时写入 Redis（在线 ZSet）与 MySQL。

### 2.3 设备影子（Shadow）

设备可开启 `is_shadow`（0/1）。影子是平台侧保存的“最近一次期望/实际状态”，用于：

- 设备不在线时先保存目标值；
- 设备重新上线后，平台把最新期望值推送给设备（影子同步）。

影子与实时值共同存储在 Redis Hash 的 `ValueItem` 中（key：`TSLV:{productId}_{serialNumber}`，RedisKeyBuilder.buildTSLVCacheKey），value/shadow/ts 三个维度。详细机制见技术篇。

## 3. 物模型（Thing Model）

物模型是 IoT 平台最具辨识度的抽象，相当于设备数据的“Schema + API 语义”。

### 3.1 三类模型

| 类别 | type | 方向 | 类比 |
| --- | --- | --- | --- |
| 属性 Property | 1 | 设备→平台为主，平台也可读 | 传感器值、状态 |
| 功能/服务 Function | 2 | 平台→设备为主 | 开关、挡位、屏显等可执行动作 |
| 事件 Event | 3 | 设备→平台 | 异常、告警等瞬时信息 |

### 3.2 数据类型定义（specs）

`iot_things_model.specs` 是 JSON，支持：

- `integer` / `decimal`：min/max/step/unit
- `string`：maxLength
- `bool`：trueText/falseText
- `enum`：enumList[text/value]、showWay（select/button）
- `array`：arrayType/arrayCount/params（支持“数组内嵌对象”表达子设备与分组采集）
- `object`：params 表达结构体（例如“功能分组”）

### 3.3 模型上的业务标记

每个物模型还带有业务开关：

- `is_history`：是否写入历史日志（时序库/MySQL）
- `is_monitor`：是否作为“实时监测”数据点
- `is_chart`：是否出现在图表
- `is_readonly`：是否只读（禁止平台下发）
- `is_share_perm`：是否允许授权给被分享用户控制
- `model_order`：排序

### 3.4 MODBUS 扩展字段

当产品面向采集设备时，模型还扩展了寄存器语义：

`temp_slave_id`（从机号）、`reg_str/reg_addr`（寄存器地址）、`formula/reverse_formula`（上报缩放公式与下发反算公式）、`quantity`、`code`（功能码）、`parse_type`（如 ushort）等，说明该平台把“采集点表”也物模型化了。

### 3.5 物模型模板

`iot_things_model_template` 提供行业通用模板（温湿度、光照、PM2.5、开关等）。新建产品时可以从模板批量导入，模板与产品之间通过 `iot_device_template` 关联记录；产品物模型是模板的“实例化副本”，此后各自维护。

## 4. 设备、产品、物模型之间的代码对应

| 业务对象 | Domain 类 | 主要表 | 备注 |
| --- | --- | --- | --- |
| 产品分类 | `Category` | `iot_category` | |
| 产品 | `Product` | `iot_product` | 含 `things_models_json` |
| 物模型 | `ThingsModel` | `iot_things_model` | 产品下的独立行 |
| 模板 | `ThingsModelTemplate` | `iot_things_model_template` | |
| 设备 | `Device` | `iot_device` | `things_model_value` 冗余 JSON |
| 授权码 | `ProductAuthorize` | `iot_product_authorize` | |
| 分组 | `Group` / `DeviceGroup` | `iot_group` / `iot_device_group` | |
| 设备用户 | `DeviceUser` | `iot_device_user` | 所有权/分享 |

## 5. 小结

读懂“产品 + 设备 + 物模型”是理解 FastBee 一切业务的前提。之后无论看数据上报、规则脚本还是前端设备详情，你都会反复碰到三类标识符：

- `serialNumber`：谁是设备
- `identifier`：说的是哪个数据点
- `productId + protocolCode + transport`：用什么协议、如何编解码

继续阅读：[3-用户-租户-权限与分享.md](3-用户-租户-权限与分享.md) 或 [5-典型业务流程与端到端场景.md](5-典型业务流程与端到端场景.md)
