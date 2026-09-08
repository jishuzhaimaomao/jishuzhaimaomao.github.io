---
title: "07 时序数据：存得下、查得快"
published: 2026-09-08
description: "100 万个传感器、每分钟 1 条、保留一年："
tags: [FastBee,时序数据]
category: FastBee
draft: false
slug: fastbee-series-07
---
# 07 时序数据：存得下、查得快

## 一个传感器的数据不可怕

100 万个传感器、每分钟 1 条、保留一年：

```text
1000000 × 60 × 24 × 365 ≈ 5256 亿条
```

传统 MySQL 单表无法支撑这种写入与聚合查询。IoT 平台通常引入**时序数据库（TSDB）**：写入快、压缩高、时间范围查询与聚合友好。

## FastBee 的数据分层

FastBee 并没有把所有数据都塞时序库，而是分层：

| 数据 | 存储 | 特点 |
| --- | --- | --- |
| 产品/设备/物模型 | MySQL | 关系型、频繁修改 |
| 最新值/影子 | Redis | 实时读取 |
| 属性历史/事件/功能 | MySQL 或 TSDB | 时间序列 |

所以它面临一个经典问题：**业务表在 MySQL，时序数据可能在 TDengine，怎么让上层无感？**

## 接口抽象是关键

`ILogService`：

```java
int saveDeviceLog(DeviceLog deviceLog);
List<MonitorModel> selectMonitorList(DeviceLog deviceLog);
List<DeviceLog> selectDeviceLogList(DeviceLog deviceLog);
List<HistoryModel> listHistory(DeviceLog deviceLog);
...
```

上层只依赖接口，具体实现在启动时确定。

## 四种实现怎么共存

```text
MySqlLogServiceImpl     默认（无时序库）
TdengineLogServiceImpl  @ConditionalOnProperty(taos.enabled=true)
InfluxLogService        Influx 配置启用
IotDbLogService         IoTDB 配置启用
```

`ApplicationStarted` 在启动时检查：

```java
if (enabledCount > 1) {
    log.error("只能启用一个时序数据库...");
    return;
}
```

避免两个时序库同时生效导致实现冲突。

## 数据模型怎么设计

以 MySQL 模式为例，`iot_device_log` 的关键列：

```text
log_id / identify / model_name / log_type / log_value
device_id / device_name / serial_number
is_monitor / mode
user_id / tenant_id / create_time
```

冗余 device_name、tenant_name 等字段，是为了在时序查询时不 JOIN 业务表——典型的“写放大换读简单”。

索引直接服务查询习惯：

```sql
INDEX (serial_number, create_time)
INDEX (serial_number, is_monitor, create_time)
```

TDengine 模式下由 Mapper 按数据库名拼接超级表 SQL，同样字段语义。

## 写时序库的隐蔽坑

代码里有个极易被忽略的细节：

```java
long baseTs = System.currentTimeMillis();
for (int i = 0; i < deviceLogList.size(); i++) {
    deviceLogList.get(i).setTs(new Date(baseTs + i));  // 1ms 间隔
    logService.saveDeviceLog(deviceLogList.get(i));
}
```

为什么 +i？因为 TDengine/InfluxDB 的写入时间戳若完全相同，会与主键/唯一约束冲突或覆盖。批量写入时**时间戳必须单调递增**。

## 查询层

数据中心与设备详情的查询都经过 ILogService：

- 实时监测：`selectMonitorList`
- 历史分页：`selectDeviceLogList`
- 曲线：`listHistory`
- 统计：`selectCategoryLogCount` / `countThingsModelInvoke`

## 怎么选型

| 规模 | 建议 |
| --- | --- |
| 学习/小型演示 | 默认 MySQL 足够 |
| 每日百万级点 | 开 TDengine（Docker Compose 中已预留注释配置） |
| 已有 InfluxDB/IoTDB | 改配置切换实现 |

## 实践任务

1. 默认 MySQL 上报一条数据，查 `iot_device_log`；
2. 在 `application-dev.yml` 打开 `taos.enabled=true` 并启动 TDengine；
3. 再次上报，观察 TDengine 超级表写入；
4. 对比设备详情历史查询在两个存储下的 SQL 与耗时。

下一站：[规则引擎让数据流动起来](08-规则引擎让数据流动起来.md)
