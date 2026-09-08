---
title: "[FastBee·系列] 09 视频监控设备如何接入"
published: 2026-09-08
description: "传感器数据几 KB，走 MQTT 就够；摄像头是连续媒体流，需要："
tags: [FastBee,FastBee·系列文章,设备与产品,视频接入]
category: FastBee·系列文章
draft: false
slug: fastbee-series-09
---
# 09 视频监控设备如何接入

## 视频为什么是 IoT 平台的老大难

传感器数据几 KB，走 MQTT 就够；摄像头是**连续媒体流**，需要：

- 信令：告诉摄像头“我要看哪一路、流发到哪”；
- 媒体：RTP/RTSP 收流、转封装；
- 播放：浏览器不能直接播 RTSP，要转 HLS/FLV/WebRTC；
- 控制：云台、录像、报警联动。

中国安防设备普遍支持国家标准 **GB/T 28181**，所以 FastBee 的视频能力 = SIP 信令（自研）+ ZLMediaKit（流媒体）。

## GB28181 的最小流程

### 1. 注册

摄像头以 SIP 用户身份注册到平台：

```text
设备 → SIP REGISTER（UDP 5061）
平台 → 校验后回复 200 OK，标记在线
```

### 2. 目录

平台发 MESSAGE 请求设备目录：

```text
平台 → 设备：Catalog 查询
设备 → 平台：XML 返回通道列表
```

`CatalogHandler` 解析 DeviceID、名称、厂商、状态，写入 `sip_device_channel`，形成“设备-通道树”。

### 3. 预览

前端点击播放：

```text
平台 → 设备：INVITE（携带 ZLMediaKit 收流地址 + SSRC）
设备 → ZLMediaKit：RTP 推流
ZLMediaKit → 前端：转 HLS/FLV/WebRTC
```

### 4. 云台控制

```text
平台 → 设备：MESSAGE <Control>（上/下/左/右/缩放）
```

`PTZCmd` 枚举与 `PtzController` 对应。

## 代码是怎么组织的

`sip-server` 模块：

```text
handler/req/RegisterReqHandler    # 注册
handler/req/InviteReqHandler      # 邀请拉流
handler/req/ByeReqHandler         # 停止
handler/req/message/notify/       # 设备主动通知
  ├── KeepaliveHandler            # 心跳
  ├── MediaStatusHandler          # 媒体状态
  ├── AlarmHandler                # 报警
  └── MobilePositionHandler       # 移动位置
handler/req/message/response/     # 设备对查询的应答
  ├── CatalogHandler              # 目录
  ├── DeviceInfoHandler           # 设备信息
  ├── DeviceControlHandler        # 控制应答
  └── ...
server/                           # SIP 层
  ├── SipLayer
  ├── MessageInvoker
  └── VideoSessionManager         # 会话/流管理
```

设计上它模仿了经典 Java GB28181 实现（如 wvp 项目）的分层：请求处理、消息处理、命令类型分发三层。

## 表与配置

| 表/配置 | 作用 |
| --- | --- |
| `sip_config` | SIP 域、ID、密码等 |
| `sip_device` | 接入的 NVR/摄像头 |
| `sip_device_channel` | 每个通道（一路视频） |
| `media_server` | ZLMediaKit 地址/密钥/端口 |
| `application.yml sip.*` | 服务开关与 SIP 身份 |

## 视频与 MQTT 设备体系的融合

`VideoMqttService` 会把视频设备同步成 `iot_device`（device_type=3），固件版本、状态等也能从 SIP 数据更新。这让“视频监控”不再是孤立子系统，而是统一设备目录的一部分。

## 没有真摄像头怎么学

方案：

1. 使用 GB28181 模拟器/开源实现（如 wvp-GB28181-pro 的模拟设备、LiveGBS）；
2. 用 ZLMediaKit 的推流工具模拟；
3. 至少把 REGISTER、Catalog 信令跑通，看到设备表/通道表变化；
4. 再验证 INVITE 与播放。

## 二次开发提示

最容易扩展的点：

- 新增报警联动：`AlarmHandler` → 通知/事件系统；
- 新增录像回放：`RecordInput/RecordList` 相关模型 + PlayerController；
- 对接更多流媒体：把 `MediaServer` 抽象成接口，替换 ZLMediaKit 实现。

下一站：[网关与子设备：采集那点事](/posts/fastbee-series-10/)

