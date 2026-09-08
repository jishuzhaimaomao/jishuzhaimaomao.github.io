---
title: "[FastBee·系列] 10 网关与子设备：采集那点事"
published: 2026-09-08
description: "智能开关、温湿度计可能自带 WiFi；但工厂里的电表、PLC、串口传感器往往没有 IP，它们挂在网关上："
tags: [FastBee,FastBee·系列文章,设备与产品,网关与子设备]
category: FastBee·系列文章
draft: false
slug: fastbee-series-10
---
# 10 网关与子设备：采集那点事

## 不是所有设备都自带网络

智能开关、温湿度计可能自带 WiFi；但工厂里的电表、PLC、串口传感器往往没有 IP，它们挂在网关上：

```text
子设备（从机） ──RS485/Modbus──▶ 网关 ──MQTT/TCP──▶ 平台
```

网关负责把多种子设备数据汇聚后上报。平台侧因此多了一层“父子关系”要管理。

## FastBee 的网关模型

`iot_device` 里：

```text
device_type：1 直连、2 网关、3 监控
gw_dev_code：子设备记录所挂网关的编号
slave_id：   从机地址
```

子设备在平台上也有一条完整设备记录，但多了 `gw_dev_code`。

## 数据如何表达“哪台子设备”

网关把子设备数据通过一条 MQTT 消息上报时，必须能区分从机。FastBee 用两种拼接：

```text
设备侧编号：serialNumber_slaveId（日志/详情维度）
物模型侧：  identifier#slaveId（查模型维度）
```

在 `reportDeviceThingsModelValue` 中：

```java
String serialNumber = slaveId == null
        ? input.getDeviceNumber()
        : input.getDeviceNumber() + "_" + slaveId;

String identity = identity + (slaveId != null ? "#" + slaveId : "");
```

## 物模型如何建模子设备

看演示“网关产品”的物模型：

```json
{
  "id": "device",
  "type": "array",
  "datatype": {
    "type": "array",
    "arrayType": "object",
    "arrayCount": 5,
    "params": [
      {"id":"device_co2", "name":"二氧化碳", ...},
      {"id":"device_switch", "name":"设备开关", ...}
    ]
  }
}
```

一个 `device` 模型里的 `params` 描述了每路子设备的数据点，网关设备详情页可据此渲染子设备卡片。

## MODBUS 方向的能力

`ThingsModel` 上有一组工业采集字段：

```text
temp_slave_id   从机号
reg_addr        寄存器地址
formula         上报换算（%s*10）
reverse_formula 下发反算
quantity        读取寄存器数
code            Modbus 功能码（如 03 读保持寄存器、06 写单寄存器）
parse_type      ushort/int/float 等
```

种子数据中就有 `★MODBUS协议产品`，其物模型 identifier 就是寄存器地址（`0`、`1`、`11`），value_type/quantity/code 字段记录了采集参数。

## 控制子设备

平台下发到子设备时，`MqttMessagePublishImpl` 中：

```java
if (bo.getSlaveId() != null) {
    PropertyDto thingModels = thingsModelService.getSingleThingModels(
            productId, identifier + "#" + slaveId);
    bo.setCode(...);   // 自动补 Modbus 功能码
}
```

`DeviceDownMessage` 的字段同时包含：

```text
subFlag：是否发往网关的子设备
subSerialNumber：子设备编号
slaveId：从机地址
code/address/count：Modbus 指令参数
```

这说明下行模型是“MQTT + Modbus”的混合抽象：如果目标设备本身是 Modbus 网关，平台就能把控制指令组装成带功能码/地址的下发结构。

## 诚实提醒

本仓库能看到**模型、字段、解析入口**，但完整“网关协议栈”（轮询采集、子设备拓扑自动发现、Modbus 主站轮询示例）主要在外部 SDK 仓库（README 中列出的 `fastbee-sdk`、Arduino/ESP 项目）和官方商业版中。RoadMap 也把“网关/子网关：上线、绑定、拓扑、透传/轮询”列为优化方向。

## 实践任务

1. 建一个网关产品（device_type=2）并发布；
2. 建一个挂在该网关下的子设备；
3. 模拟网关上报子设备温度（带 slaveId）；
4. 在设备列表/详情观察子设备数据与模型匹配；
5. 测试非法 slaveId、未定义从机地址时的行为。

下一站：[从若依到 FastBee：二次开发心法](/posts/fastbee-series-11/)

