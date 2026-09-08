---
title: "[FastBee·测试篇] MQTT 接入专项测试手册"
published: 2026-09-08
description: "推荐："
tags: [FastBee,FastBee·测试篇,MQTT,软件测试]
category: FastBee·测试篇
draft: false
slug: fastbee-test-4
---
# MQTT 接入专项测试手册

## 1. 测试对象

- 端口：TCP 1883、WebSocket 8083/mqtt
- 实现：`mqtt-broker`（Netty）+ `TopicsUtils` + `AuthService`
- 设备接入角色：设备端（S/E 认证）、Web/Phone、server 内部客户端

## 2. 工具准备

推荐：

- MQTTX（桌面，支持 MQTT/WebSocket）
- mosquitto_pub / mosquitto_sub
- 浏览器 WebSocket（mqtt.js 页面）
- Wireshark 或 tcpdump（抓 MQTT/网络包）

## 3. 连接参数模板

### 3.1 设备简单认证（S）

```text
host: localhost
port: 1883
clientId: S&D1ELV3A5TOJS&41&1
username: FastBee
password: P47T6OD5IPFWHUM6   # 来自演示产品 mqtt_password
cleanSession: true/false 按用例
```

> 演示产品 41 的密码可从 `fastbee.sql` 的 `iot_product` 查得，仅测试用。

### 3.2 Web 端

```text
host: ws://localhost:8083/mqtt
clientId: web{time}
password: Bearer <登录JWT>
```

### 3.3 加密认证（E）

```text
clientId: E&D1ELV3A5TOJS&41&1
password: AES(mqtt_password + 时间戳) 使用产品 mqtt_secret 加密
```

可先用 `ToolServiceImpl` 的实现（`AESUtils`）写测试工具脚本。

## 4. 协议符合性用例

| 编号 | 用例 | 操作 | 预期 |
| --- | --- | --- | --- |
| M-01 | 正常 CONNECT | S 认证参数正确 | CONNACK Accepted |
| M-02 | 错误密码 | 任意改密码 | CONNACK 拒绝并断连 |
| M-03 | 空 clientId/username | 缺参数 | 拒绝 |
| M-04 | 非发布产品 | 用未发布产品 | 拒绝 |
| M-05 | clientId 段数错误 | `S&SN&41` | 拒绝（格式提示） |
| M-06 | 禁用设备 | 平台禁用后再连 | 拒绝 |
| M-07 | server/web 认证 | server 前缀+错误账号；web 前缀+坏 JWT | 分别拒绝 |
| M-08 | 未 CONNECT 直接 PUBLISH | 客户端不发送 CONNECT | 拒绝/断开 |
| M-09 | 心跳超时 | keepalive 很小且不 ping | 服务端断开，设备变离线 |
| M-10 | cleanSession=true 重连 | 重连后订阅 | 旧订阅清除，需重新订阅 |
| M-11 | cleanSession=false | 重连后查询 | 会话行为按实现记录（当前实现待验证） |
| M-12 | 重复 clientId | 两台同时连 | 旧连接被移除，新连接可用 |
| M-13 | 非法订阅 | `#`开头/无`/`/多个`#` | SUBACK 不应成功 |
| M-14 | 通配订阅 | 订阅 `+/+/property/post` | 能收到该前缀下属性上报 |
| M-15 | QoS0/1/2 发布 | 分别发消息 | ACK 序列符合（QoS2 PUBREC→PUBREL→PUBCOMP） |
| M-16 | Retain | retain=true 发布→新订阅者 | 新订阅立即收到保留消息 |
| M-17 | 清除 Retain | retain=true 空消息 | 清除后不再补推 |
| M-18 | Will | CONNECT 带遗嘱，然后异常断开 | 遗嘱消息被发布 |
| M-19 | 正常 DISCONNECT | 客户端主动断开 | 不发布遗嘱，状态离线 |

## 5. 业务主题用例

### 5.1 属性上报

```bash
mosquitto_pub -h localhost -p 1883 -i 'S&D1ELV3A5TOJS&41&1' \
  -u FastBee -P '<产品密码>' \
  -t /41/D1ELV3A5TOJS/property/post -q 1 \
  -m '[{"id":"temperature","value":"25.6"},{"id":"humidity","value":"56"}]'
```

断言点：

1. 返回 PUBACK；
2. 订阅 `+/41/D1ELV3A5TOJS/ws/service` 的客户端收到推送；
3. MySQL/TDengine `iot_device_log` 出现温度/湿度；
4. Redis `TSLV:41:D1ELV3A5TOJS` Hash 的字段更新；
5. Web 设备详情出现曲线。

### 5.2 非法数据

| payload | 预期 |
| --- | --- |
| 非 JSON | decode 报错/忽略，服务不崩 |
| `[]` | 空数据，正常返回 |
| `[{"id":"not_exist","value":"1"}]` | 跳过未知模型 |
| 类型错误 `"abc"` | 按实现校验（记录日志/丢弃） |
| 超大 payload（>2MB 上限） | 连接按解码失败处理 |

### 5.3 模拟设备主题

`property/simulate/post`、`property/get/simulate` 等应进入模拟数据通道，可供前端模拟器测试。

### 5.4 订阅“功能下发”

设备需预先订阅 `function/get`：

```text
订阅: /41/D1ELV3A5TOJS/function/get
Web 触发: 打开开关
收到: [{"id":"switch","value":"1"}]
```

## 6. 状态机验证

辅助观察：

- `/41/D1ELV3A5TOJS/status/post` 的订阅者；
- Redis `device:online:list` ZSet；
- `iot_device.status`；
- Netty 管理页客户端列表。

场景：

1. 连接 → ONLINE 消息 + Redis + MySQL 在线；
2. 发 PINGREQ 保活 → 不掉线；
3. 停发心跳 → 判定离线；
4. 主动断开 → 立即离线；
5. 服务器重启 → 全部设备按离线处理（当前实现为内存状态，注意预期）。

## 7. 性能与稳定性专项

专项目标建议（小规模基准）：

| 场景 | 方法 |
| --- | --- |
| 连接风暴 | 脚本并发 500/1000 个连接，观察 CONNACK 成功率与内存 |
| 上报吞吐 | 单设备 100~1000 msg/s 观察丢包、落库延迟 |
| 多设备上报 | 100 设备每 5 秒上报，观察 Redis/DB 压力 |
| 大量订阅主题 | 观察 topicMap 与发布匹配耗时 |
| 重连稳定性 | 设备循环 连接-断开-心跳超时 |

## 8. 已知边界与缺陷提示

- 会话、Retain、Will、QoS 状态在 JVM 内存，**重启即失**；
- `MessageStoreImpl` 注释明确“-TODO 后续 Redis 处理”；
- 服务端内部发布与设备消息共用同一 Broker，压测时注意内部客户端的影响；
- 设备离线消息补推、订阅持久化在开源版实现不完整，测试时按实际代码行为记录，不按商业平台标准臆测。

下一章：[接口、性能与安全测试](/posts/fastbee-test-5/)

