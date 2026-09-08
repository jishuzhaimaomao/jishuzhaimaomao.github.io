---
title: "视频接入：GB28181 与 SIP"
published: 2026-09-08
description: "FastBee 采用标准视频接入范式："
tags: [FastBee,视频接入]
category: FastBee
draft: false
slug: fastbee-tech-9
---
# 视频接入：GB28181 与 SIP

## 1. 视频能力在 FastBee 中怎么工作

FastBee 采用标准视频接入范式：

```text
GB28181 摄像头/NVR ──SIP(UDP 5061)──▶ FastBee SIP Server
                                              │
                    目录/状态/云台等 信令处理  │
                                              ▼
                                    ZLMediaKit（流媒体服务器）
                                              │
                                    HLS/FLV/WebRTC 等拉流
                                              ▼
                                      前端播放器页面
```

其中：

- **SIP 信令**由仓库内 `fastbee-server/sip-server` 实现；
- **媒体转发/播放**由独立 ZLMediaKit 容器承担（不在本仓库代码内）；
- **业务表**：`sip_config`、`sip_device`、`sip_device_channel`、`media_server`。

## 2. 模块结构

```text
com.fastbee.sip
├── conf/            # SIP 系统配置（SysSipConfig）、线程池
├── domain/          # SipDevice/SipDeviceChannel/MediaServer/SipConfig
├── enums/           # ChannelType/PTZCmd/AlarmMethod/...
├── handler/
│   ├── IReqHandler/IResHandler     # SIP 请求/响应接口
│   ├── req/                         # Register/Invite/Bye/Cancel/Ack
│   ├── req/message/                 # MESSAGE 报文处理
│   │   ├── notify/                  # Keepalive/MediaStatus/Alarm/移动位置
│   │   └── response/                # Catalog/DeviceInfo/Config/Control/Status
│   └── res/                         # 对设备请求的响应处理
├── model/           # 请求消息/流/通道模型
├── server/          # SipLayer/MessageInvoker/VideoSessionManager
├── service/         # 设备/通道/流媒体服务
└── util/
```

约 115 个 Java 文件，是仓库内复杂度仅次于 IoT 业务的模块。

## 3. 关键信令流程

### 3.1 设备注册

`RegisterReqHandler`：

1. 校验 SIP 域与设备 ID；
2. 设备在线状态更新；
3. 触发目录查询（Catalog）以拉取通道；
4. 建立/更新设备会话。

### 3.2 通道管理

`CatalogHandler` 解析设备返回的 XML（DeviceID/名称/状态/厂商），落 `sip_device_channel`，在 Web“视频中心 → 通道管理”展示。

### 3.3 实时预览

前端播放请求大致流程：

```text
PlayerController（选流/播放地址）
  → 创建 StreamURL
  → INVITE 信令给摄像头（携带 ZLMediaKit 收流端口/SSRC）
  → 摄像头 RTP 推流给 ZLMediaKit
  → ZLMediaKit 转封装
  → 前端用播放器拉流
```

`MediaServerController` 负责 ZLMediaKit 配置/探测/自动同步，`ZmlHookController` 接收 ZLMediaKit 回调。

### 3.4 云台与录像

- `PtzController`：云台方向/缩放；
- `PlayerController`：录像查询/回放；
- 通过 SIP MESSAGE（`<Control>`、`<Query>` 等）下发到设备。

## 4. 配置

### 4.1 application-dev/prod.yml

```yaml
sip:
  enabled: false/true
  ip: 192.168.x.x       # 绑定的 SIP 地址
  port: 5061
  domain: 3402000000
  id: 34020000002000000001
  password: 12345678
```

### 4.2 数据库配置页

- 媒体服务器（ZLMediaKit IP/端口/密钥/自动配置）；
- SIP 设备与通道；
- SIP 系统配置。

## 5. 与 MQTT 业务的桥接

`sip-server` 中出现 `VideoMqttService`：把视频设备状态同步为 `iot_device` 数据（firmwareVersion、状态等），说明 FastBee 把“监控设备”也纳入统一设备体系（device_type=3）。

## 6. 学习与测试注意

- 本地跑通需要真实摄像头/NVR 或 GB28181 模拟器（如 wvp-GB28181-pro、LiveGBS 模拟源）；
- 需要配置 ZLMediaKit 服务并保证 SIP/RTSP 端口可达；
- 前端播放组件在 `vue/src/views/components/player` 与相关 SIP 页面；
- 媒体服务器（`media_server`）与 `sip_config` 表是运行核心，先在 Web 页面配好再联调。

下一章：[前端 Vue 架构与实时通信](/posts/fastbee-tech-10/)
