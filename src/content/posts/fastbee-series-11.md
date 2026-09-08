---
title: "11 从若依到 FastBee：二次开发心法"
published: 2026-09-08
description: "FastBee 的长相和若依（RuoYi-Vue）几乎一样：Vue2 + Element-UI 后台、Spring Boot + Security + JWT、动态菜单、代码生成器。"
tags: [FastBee,若依]
category: FastBee
draft: false
slug: fastbee-series-11
---
# 11 从若依到 FastBee：二次开发心法

## FastBee 的基因

FastBee 的长相和若依（RuoYi-Vue）几乎一样：Vue2 + Element-UI 后台、Spring Boot + Security + JWT、动态菜单、代码生成器。

这不是巧合：它的系统层直接复用若依 3.8.9。所以如果你会若依，你已经会 FastBee 的 60%。

## 熟悉与陌生的分界线

| 熟悉（若依） | 陌生（FastBee 新增） |
| --- | --- |
| 用户/角色/菜单/部门 | 产品/设备/物模型 |
| sys_* 表 | iot_* 表 |
| SysUser/SysRole | Device/Product/ThingsModel |
| /system/** 接口 | /iot/**、/iot/runtime/** |
| 操作日志 | 设备日志/事件日志/功能日志 |
| Spring MVC 增删改查 | 消息链路（Broker→Service→TSDB） |

二次开发时最重要的是别把“业务表新增”当全部——**物模型、Topic、缓存、时序存储**才是 IoT 的隐藏维度。

## 三个核心心法

### 心法 1：先找 deviceId/serialNumber 怎么流转

所有 IoT 功能最终都落到“某台设备”。跟踪时先问：

```text
serialNumber 从哪来？
→ 通常 URL/body 传入
→ deviceId 从哪查？
→ 谁校验了归属？
→ 数据写到哪张表/哪个 Redis key？
```

`DeviceServiceImpl.selectDeviceBySerialNumber` 和 `RedisKeyBuilder.buildTSLVCacheKey` 是两个万能坐标。

### 心法 2：REST 只是“半个入口”

传统 CRUD 只有 REST 一个入口。FastBee 的业务数据还有一个入口：**MQTT**。

举例：新增“上报电池电量”功能，不能只做前端页面，还要：

1. 产品物模型加属性 `battery`；
2. 设备按 Topic 上报；
3. 确认物模型 `is_history/is_monitor`；
4. 确认缓存与日志写入；
5. 前端图表。

### 心法 3：改代码前先看“双轨设计”

FastBee 常出现两个相似实现：

- 内置 Broker 直通 vs EMQX 队列；
- MySQL 日志 vs TDengine 日志；
- 页面服务层 vs MQ 消息服务层。

改功能前先确认当前跑的是哪条轨，避免改了 A 轨，实际跑 B 轨。

## 完整二次开发示例：新增“设备纬度管理”页面

### Step 1 数据

`iot_device` 已有 `longitude/latitude` 字段，业务侧不用加表。

### Step 2 后端

`Device` domain 已有字段 → 只需在 Service/Controller 增加接口或在现有 update 中校验。

### Step 3 前端

`views/iot/device` 中设备编辑弹窗加两个输入框，调已有 `device.js` 的更新接口。

### Step 4 权限

确保角色有 `iot:device:edit`；新增页面按钮时加菜单权限。

### Step 5 测试

分别验证 admin、租户、被分享用户对该字段的读写隔离。

## 进阶：新增“振动传感器”产品类型

如果设备数据格式特殊，最佳路径不是改 Java 协议类，而是：

1. 产品协议 code 设为自定义；
2. 给该产品写数据流 Groovy 脚本，把私有 payload 转成 JSON 数组；
3. 需要复杂逻辑再扩展 `JsonProtocolService` 类似的协议类（可参考 `@SysProtocol` 注解）；
4. 补充物模型定义。

这套模式让“协议适配”与“平台代码”解耦，是 FastBee 最值得借鉴的扩展点。

## 踩坑清单

| 坑 | 说明 |
| --- | --- |
| 改了物模型不清缓存 | Redis 仍存旧 ValueItem，需清 key 或兼容升级 |
| 新增上报字段没加物模型 | 数据被静默丢弃 |
| 只改 MySQL 实现 | 启用 TDengine 后功能不一致 |
| 直接改默认口令配置 | 容器/本地配置各有不同，确认环境 |
| 忽略双轨 | EMQX 模式与内置模式行为差异 |
| 权限只加按钮 | 后端接口没加 `@PreAuthorize`，等于没控制 |

## 推荐代码阅读顺序（二开版）

```text
DeviceController → DeviceServiceImpl → DeviceMapper.xml
DeviceRuntimeController → FunctionInvokeImpl → MqttMessagePublishImpl
ThingsModelController → ThingsModelServiceImpl → iot_things_model
DataCenterController → DataCenterServiceImpl → ILogService 实现
```

下一站：[把 FastBee 跑起来的部署实战](/posts/fastbee-series-12/)
