---
title: "数据库设计"
published: 2026-09-08
description: "springboot/sql/fastbee.sql 是 MySQL 全量脚本（约 70 张表 + 演示数据）；另有："
tags: [FastBee,时序数据]
category: FastBee
draft: false
slug: fastbee-tech-11
---
# 数据库设计

## 1. 数据库文件

`springboot/sql/fastbee.sql` 是 MySQL 全量脚本（约 70 张表 + 演示数据）；另有：

```text
sql/postgres/fastbee-v2.1.sql
sql/sqlserver/fastbee-v2.1.sql
sql/oracle/fastbee-v2.1.sql
sql/dameng/fastbee-v2.1.sql   # 达梦
sql/iotdb/iotdb.sql
sql/clear/clear-data.sql
```

## 2. 表分组

### 2.1 若依系统表

`sys_user/sys_role/sys_menu/sys_dept/sys_post/sys_dict/sys_config/sys_notice/sys_user_role/sys_role_menu/...`

扩展了：`sys_auth_user`（微信/OAuth 绑定）、多语言翻译表（`sys_menu_translate`、`sys_dict_*_translate`）。

### 2.2 IoT 产品/物模型

| 表 | 作用 |
| --- | --- |
| `iot_category` | 产品分类 |
| `iot_product` | 产品（含协议、认证、物模型 JSON） |
| `iot_protocol` | 协议注册（内置 JSON） |
| `iot_product_authorize` | 设备授权码 |
| `iot_things_model` | 物模型逐条 |
| `iot_things_model_template` | 通用模板 |
| `iot_device_template` | 产品-模板关联 |
| `iot_things_model_translate` | 物模型多语言 |

### 2.3 设备与权限

| 表 | 作用 |
| --- | --- |
| `iot_device` | 设备主表（状态/位置/影子开关/摘要） |
| `iot_device_group`、`iot_group` | 分组与设备分组关系 |
| `iot_device_user` | 设备用户/分享关系 |

### 2.4 数据日志与时序

| 表 | 作用 |
| --- | --- |
| `iot_device_log` | 属性/事件/上下线等日志（MySQL 模式） |
| `iot_event_log` | 事件日志（MySQL 模式） |
| `iot_function_log` | 指令下发/回执日志 |
| TDengine `fastbee_log` | 时序库模式下的目标库（代码动态建库） |

### 2.5 规则与任务

| 表 | 作用 |
| --- | --- |
| `iot_scene` | 场景/规则链（LiteFlow chain） |
| `iot_scene_script` | 场景触发器与动作明细 |
| `iot_scene_device` | 场景关联设备 |
| `iot_script` | 规则脚本（LiteFlow script 节点） |
| `iot_device_job` | 设备定时任务（控制/告警/场景定时） |
| `qrtz_*` | Quartz 调度表 |
| `sys_job`/`sys_job_log` | 系统任务 |

### 2.6 视频

| 表 | 作用 |
| --- | --- |
| `sip_config` | SIP 域/服务器配置 |
| `sip_device` | GB28181 设备 |
| `sip_device_channel` | 设备通道 |
| `media_server` | ZLMediaKit 流媒体服务配置 |

### 2.7 其他

- `news`/`news_category`：资讯；
- `social_platform`/`social_user`：三方登录；
- `oauth_client_details` 等 OAuth2 表；
- `app_language`/`app_preferences`：App 偏好；
- `gen_table*`：代码生成器。

## 3. 核心表字段速览

### iot_device（设备）

关键列：`device_id/product_id/serial_number/gw_dev_code/status/is_shadow/network_ip/longitude/latitude/active_time/things_model_value/summary/is_simulate/slave_id`。

索引策略：serial_number 唯一；product_id、tenant_id、user_id、create_time 各建普通索引。

### iot_product（产品）

关键列：`product_id/protocol_code/category_id/is_sys/is_authorize/mqtt_account/mqtt_password/mqtt_secret/status/device_type/network_method/vertificate_method/things_models_json/transport`。

### iot_things_model（物模型）

关键列：`model_id/product_id/identifier/type/datatype/specs/is_chart/is_monitor/is_history/is_readonly/is_share_perm/model_order/temp_slave_id/formula/reg_addr/quantity/code/parse_type`。

### iot_device_log（日志）

关键列：`log_id/identify/model_name/log_type/log_value/device_id/device_name/serial_number/is_monitor/mode/user_id/tenant_id/create_time`。

复合索引（serial_number, create_time）、（serial_number, is_monitor, create_time）服务历史查询。

## 4. JSON 字段的用法

MySQL 中多处使用 JSON 类型，如：

- `iot_product.things_models_json`
- `iot_device.things_model_value`
- `iot_device.summary`
- `iot_device_job.actions` / `alert_trigger`
- `iot_scene.el_data`

它们分别服务“产品详情渲染、设备状态冗余、设备摘要、定时动作、规则链”。换库（PostgreSQL/SQL Server/Oracle/达梦）时，JSON 列由各方言脚本适配。

## 5. 演示数据

`fastbee.sql` 自带：

- 管理员 `admin/admin123`；
- 产品示例：`★智能开关产品`（JSON/MQTT/直连）、`★网关产品`（子设备/数组物模型）、`￥视频监控产品`（GB28181）；
- 设备示例与物模型数据；
- 一条内置数据流脚本（消息转发规则）。

演示数据对学习有巨大价值：直接查看 `things_models_json` 能理解数组、对象、enum 等 specs 的真实写法。

## 6. 设计经验总结

1. **冗余导向查询**：device_name/product_name/user/tenant 冗余到子表，减少关联；
2. **状态与历史分离**：最近值放 Redis/设备行，历史放日志/时序库；
3. **平台通用性优先**：所有 SQL 提供多方言，便于替换已有 IT 系统的数据库；
4. **脚本/规则即数据**：LiteFlow 的 chain/script 都存业务表，配置可热管理；
5. **注意一致性**：JSON 冗余 + 分表实现会给“改产品名/删设备”带来级联更新风险，写测试时优先覆盖。

下一章：[部署、运维与二次开发指引](12-部署运维与二次开发指引.md)
