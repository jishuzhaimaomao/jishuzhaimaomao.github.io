---
title: "06 设备影子与在线状态管理"
published: 2026-09-08
description: "网络设备不可控。用户晚上 11 点对一台已下线的设备说“明早 6 点开灯”，平台怎么办？"
tags: [FastBee,设备与产品]
category: FastBee
draft: false
slug: fastbee-series-06
---
# 06 设备影子与在线状态管理

## 为什么需要影子

网络设备不可控。用户晚上 11 点对一台已下线的设备说“明早 6 点开灯”，平台怎么办？

两个选择：

1. 拒绝下发（设备不在线）；
2. 把指令存下来，等设备上线再给。

方案 2 就是“影子”：平台保存一份设备的**期望状态**，它不一定等于设备当前实际状态。

## FastBee 影子模型

Redis 中每个物模型字段的 `ValueItem`：

```json
{
  "value":  "20.5",   // 设备最近上报的实际值
  "shadow": "20.5",   // 平台保存的期望值（通常也作为最近值）
  "ts": "2026-09-08 10:00:00"
}
```

看 `reportDeviceThingsModelValue` 的代码分支：

```java
if (isShadow) {
    valueItem.setShadow(value);            // 影子上报：只改 shadow
} else {
    valueItem.setValue(value);
    valueItem.setShadow(value);            // 普通上报：value=shadow
    valueItem.setTs(DateUtils.getNowDate());
}
```

而指令下发在影子模式下不再真正 publish：

```java
if (bo.getIsShadow()) {
    dataHandler.reportData(...);  // 当作一次影子上报
    return;
}
```

## 设备上线时如何同步影子

设计意图是：

1. 设备 CONNECT；
2. 平台发布状态 `/status/post`，消息里带 `isShadow`；
3. 若影子开启，平台把保存的属性/功能值（Retain 或主动下发）发给设备；
4. 设备订阅后同步本地状态。

代码参考：

- `DeviceServiceImpl.getDeviceShadowThingsModel`：把 Redis 缓存转成属性和功能列表；
- `MqttRemoteManager.pushDeviceStatus`：上线即发状态；
- `ToolController.webHookProcess`（EMQX 模式）中 `client_connected` 分支：影子设备上线后发布属性和功能。

> 内置 Broker 与 EMQX 两套逻辑并存，联调时先确认当前模式，再验证“上线同步”预期。

## 在线状态：不是“断开才算离线”

设备离线判断：

```text
CONNECT 成功          → ONLINE
正常 DISCONNECT       → OFFLINE
异常断网              → 靠心跳超时判定
```

`ClientManager` 记录每个客户端最近活跃时间：

```java
public static void updatePing(String clientId) {
    pingMap.put(clientId, DateUtils.getTimestamp());
}
```

判定：

```java
if (currTime - timestamp > DEVICE_PING_EXPIRED) {
    return false;   // 视为不可投递
}
```

`SessionManger.removeClient` 里发布 OFFLINE 并调用：

```java
deviceCache.updateDeviceStatusCache(statusBo);
remoteManager.pushDeviceStatus(...);
```

`DeviceCacheImpl` 同步 Redis 在线 ZSet 与 MySQL 状态。有意思的细节：ONLINE 更新 MySQL 前 `Thread.sleep(500)`，注释是“延时解决状态同步问题”。

## 状态相关的数据位置

| 数据 | 位置 | 目的 |
| --- | --- | --- |
| 在线列表 ZSet | Redis | 快速判断在线、统计 |
| 最近值/影子 Hash | Redis | 低延迟读取 |
| 设备主状态 | MySQL iot_device.status | 业务持久化 |
| 上线/下线历史 | 日志表（log_type 5/6） | 审计回溯 |
| 状态推送 | MQTT `/status/post` | 实时通知前端 |

## 容易踩的坑

1. **重启即丢**：在线 ZSet 是 Redis 的，能持久化；但 Broker 的会话/订阅映射是 JVM 的，重启后所有设备看起来都要重连；
2. **影子与离线补报**：代码中还有 `property-offline/function-offline` 等主题分支，表示设计上允许设备离线期间把记录补上来，但完整闭环需要验证；
3. **设备编号大小写**：代码里 `input.setDeviceNumber(serialNumber.toUpperCase())`，缓存 key 必须和上报编号保持一致。

## 实践任务

1. 开启设备的 `is_shadow`；
2. 设备离线，Web 下发一个开关值；
3. 查看 Redis Hash 的 shadow 是否变化、MySQL 状态是否仍离线；
4. 设备上线，观察它是否收到同步数据；
5. 拔网线 2 分钟，确认离线判定时间与 `keep-alive`/超时阈值的关系。

下一站：[时序数据：存得下、查得快](/posts/fastbee-series-07/)
