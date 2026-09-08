---
title: "物模型与数据存储设计"
published: 2026-09-08
description: "FastBee 在数据库里保存两份物模型："
tags: [FastBee,物模型,时序数据]
category: FastBee
draft: false
slug: fastbee-tech-7
---
# 物模型与数据存储设计

## 1. 物模型的两种载体

FastBee 在数据库里保存两份物模型：

1. **产品物模型 JSON**（`iot_product.things_models_json`）：整体快照，方便产品详情页一次展示；
2. **逐条物模型**（`iot_things_model`）：每个属性/功能/事件一行，是数据校验、缓存、设备详情渲染的主要依据。

另有一份**模板库**（`iot_things_model_template`）与产品通过 `iot_device_template`（产品-模板关联）提供批量复制。

## 2. 物模型结构解析

### 2.1 主字段

| 字段 | 说明 |
| --- | --- |
| `model_name` | 展示名（温湿度/开关/异常） |
| `identifier` | 数据标识符，产品内唯一 |
| `type` | 1=属性，2=功能，3=事件 |
| `datatype` | 上层数据类型标识（integer/decimal/string/bool/enum/array/object） |
| `specs` | 具体定义 JSON |
| `is_history / is_monitor / is_chart / is_readonly / is_share_perm` | 存储与 UI 开关 |
| `model_order` | 排序 |
| MODBUS 字段 | `temp_slave_id/reg_addr/formula/reverse_formula/quantity/code/parse_type` |

### 2.2 specs 常见形态（SQL 种子数据实例）

数值：

```json
{"type":"decimal","max":120,"min":-20,"step":0.1,"unit":"℃"}
```

枚举按钮：

```json
{"type":"enum","showWay":"button",
 "enumList":[{"text":"重启","value":"restart"}]}
```

数组（RGB 灯色）：

```json
{"type":"array","arrayType":"integer","arrayCount":"3"}
```

复杂采集（子设备/功能分组）：

```json
{"type":"array","arrayType":"object","arrayCount":5,
 "params":[{"id":"device_co2","name":"二氧化碳",...}]}
```

## 3. 设备侧物模型值缓存

### 3.1 Redis Key

`RedisKeyBuilder.buildTSLVCacheKey(productId, serialNumber)`（设备物模型值命名空间）：

```text
TSLV:{productId}_{serialNumber}           # Redis Hash
field = identifier（数组等场景为 id，含 slaveId 时加 #slaveId）
value = ValueItem JSON
```

另外 `RedisKeyBuilder.buildTSLCacheKey(productId)` 是**物模型定义缓存**：`TSL:{productId}` Hash 下缓存该产品各 identifier 对应的物模型定义，供上报时快速查询。

`ValueItem` 至少包含：

```json
{
  "id": "temperature",
  "value": "23.5",      // 最近实际值
  "shadow": "23.5",     // 影子值
  "ts": "2026-09-08 10:00:00"
}
```

### 3.2 上报写入流程

详见[消息上行链路](5-设备接入与消息上行链路.md)第 4 节。核心顺序是：**物模型存在性校验 → 缓存更新 → 日志落库**。

## 4. 历史数据存储抽象

### 4.1 ILogService 接口

`com.fastbee.iot.tsdb.service.ILogService` 统一了设备日志读写：

```java
createSTable(database)
saveDeviceLog(deviceLog)
saveBatch(TdLogDto dto)
deleteDeviceLogByDeviceNumber(deviceNumber)
selectCategoryLogCount(device)
selectMonitorList(deviceLog)
selectDeviceLogList(deviceLog)
listHistory(deviceLog)
countThingsModelInvoke(dataCenterParam)
```

### 4.2 四种实现

| 实现类 | 启用条件 | 说明 |
| --- | --- | --- |
| `MySqlLogServiceImpl` | 未启用时序库时 | 写 `iot_device_log`/`iot_event_log` |
| `TdengineLogServiceImpl` | `taos.enabled=true`，`@Primary` + `@DS("taos")` | 写 TDengine 超级表 |
| `InfluxLogService` | Influx 配置启用 | 通过 Influx Java API |
| `IotDbLogService` | IoTDB 配置启用 | 通过 IoTDB JDBC |

启动时 `ApplicationStarted` 会检查并发开启数量：

```java
enabledCount = TDengine + Influx + IoTDB
if (enabledCount > 1) { log.error("只能启用一个时序数据库"); return; }
```

> 当前开源配置默认全部时序库关闭 → 数据落 MySQL。

### 4.3 TDengine 细节

- `@ConditionalOnProperty(name="spring.datasource.dynamic.datasource.taos.enabled", havingValue="true")`
- JDBC URL 支持 `jdbc:TAOS://`、`jdbc:TAOS-WS://`、`jdbc:TAOS-RS://`
- 启动自动建库 `fastbee_log` 与超级表；
- `saveDeviceLog` 用 Snowflake 生成 logId，时间由调用方保证递增（1ms 间隔技巧）。

## 5. 日志分类

设备日志类型 `log_type`：

| 值 | 含义 | 去向 |
| --- | --- | --- |
| 1 | 属性上报 | device_log |
| 2 | 调用功能/指令 | function_log |
| 3 | 事件上报 | device_log/event_log |
| 4 | 设备升级 | 预留 |
| 5 | 设备上线 | device_log |
| 6 | 设备离线 | device_log |
| 8 | 其他事件 | event_log |

`mode`：1=影子模式、2=在线模式、3=其他。

## 6. 实时监测与历史查询

设备详情页运行时数据来自三类查询：

- `selectMonitorList`：monitor 数据点实时列表；
- `selectDeviceLogList`：按 identifier/时间分页；
- `listHistory`：ECharts 曲线（TDengine 中会去掉毫秒）。

数据中心接口：

- `DataCenterController`/`DataCenterServiceImpl`
- 分类统计 `selectCategoryLogCount`：属性/事件/监测总数
- 物模型调用统计 `countThingsModelInvoke`：按产品/时间聚合

## 7. 冗余设计与一致性

平台为查询性能做了不少“写时冗余”：

- `iot_device.things_model_value`（JSON）存设备全部物模型值；
- Redis Hash 存最近值/影子；
- 日志表冗余 device_name/product_name/user/tenant。

代价是状态一致需要业务保证：删除设备时清理 Redis 与日志；改物模型时可能需要同步各设备缓存。这是测试与二次开发需要重点回归的部分。

下一章：[规则引擎与定时任务](8-规则引擎与定时任务.md)
