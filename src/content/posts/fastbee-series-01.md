---
title: "01 为什么需要物联网平台"
published: 2026-09-08
description: "假设你自己做了一台带 WiFi 的温湿度计，代码里连上 MQTT，服务器收到 temperature=25.6。这算不算“物联网平台”？"
tags: [FastBee,学习笔记]
category: FastBee
draft: false
slug: fastbee-series-01
---
# 01 为什么需要物联网平台

## 一个朴素的问题

假设你自己做了一台带 WiFi 的温湿度计，代码里连上 MQTT，服务器收到 `temperature=25.6`。这算不算“物联网平台”？

其实不算。你很快会遇到这些重复劳动：

1. **设备管理**：每台设备一个编号、激活、禁用、在线判断；
2. **数据语义**：温度 25.6 到底是摄氏、华氏、原始 ADC 值还是寄存器值？
3. **上行接入**：不同厂商上报格式不同，有的发 JSON、有的发 Modbus 字节流；
4. **下行控制**：网页点“打开开关”，怎么安全地发到那台设备并知道结果；
5. **历史存储**：1 万个设备每天 24 小时上报，MySQL 一张表扛不住怎么办；
6. **用户与安全**：谁能看这台设备、谁能控制它；
7. **告警与自动化**：温度超限要通知、定时开关要下发。

把这些重复能力做成一个“中间层”，就是物联网平台。设备厂商只写固件，业务方只写应用，平台负责“连接、管理、语义、存储、安全”。

## FastBee 的取舍

大厂 IoT 平台通常很强，但重：私有协议难接、本地部署贵、学习曲线陡。FastBee 的选择是：

- **单机可部署**：一个 Java 进程 + MySQL + Redis 就能跑；
- **MQTT 自建**：省去 EMQX 的运维，也让开发者能读到 Broker 每一行源码；
- **概念向主流平台看齐**：产品/设备/物模型，用户从阿里云等平台迁移过来几乎无认知成本；
- **规则脚本化**：私有协议不一定要改 Java，Groovy 脚本可改写消息；
- **若依底座**：管理后台、RBAC、代码生成直接复用成熟框架。

## 它的“平台”价值怎么体现

以仓库中的演示设备 `★智能开关产品`（productId=41）为例：

```text
产品定义：
  properties: temperature / humidity / co2 / brightness
  functions: switch / gear / irc / light_color / message / reset
  events: exception / height_temperature
设备：serialNumber=D1ELV3A5TOJS
```

如果不用平台，你要自己写：

- 产品物模型 JSON 的存取；
- 上报数据与物模型的校验映射；
- 设备状态缓存与历史日志；
- 前端曲线与下发按钮；
- 用户角色与设备分享。

用了平台，核心只剩：

1. 设备按约定 Topic 上报 JSON；
2. 页面配置物模型；
3. 平台完成剩下一切。

这就是“平台”的价值：把重复的 IoT 横切能力做成标准服务。

## FastBee 不是万能的

读完源码后更要诚实看待边界：

- 它首先是一套**可学习、可二次开发**的中小规模平台；
- OTA、完整场景告警、集群 Broker、全协议支持并未全部闭环；
- 硬件 SDK、移动端 UI 在别的仓库；
- 生产化需要自己补安全、监控、部署加固。

## 从哪开始读

建议先读代码里最能表达“平台抽象”的两个地方：

- `springboot/sql/fastbee.sql` 中 `iot_product`、`iot_device`、`iot_things_model` 的建表语句与演示数据；
- `springboot/fastbee-common/.../enums/TopicType.java` 的 Topic 规范。

下一站：[物模型：平台的语义中枢](/posts/fastbee-series-02/)
