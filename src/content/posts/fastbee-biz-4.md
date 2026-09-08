---
title: "[FastBee·业务篇] 功能地图：从界面菜单到代码模块"
published: 2026-09-08
description: "本文件把 Web 端看到的菜单和底层代码模块对应起来，方便“先点界面、再找源码”的学习路径。"
tags: [FastBee,FastBee·业务篇,系统架构]
category: FastBee·业务篇
draft: false
slug: fastbee-biz-4
---
# 功能地图：从界面菜单到代码模块

本文件把 Web 端看到的菜单和底层代码模块对应起来，方便“先点界面、再找源码”的学习路径。

## 1. 一级菜单全景

根据 `fastbee.sql` 的 `sys_menu` 种子数据，顶层菜单有：

| 一级菜单 | 业务归属 | 说明 |
| --- | --- | --- |
| 设备管理 (`iot`) | IoT 核心 | 产品分类/产品/设备/分组/事件日志等 |
| Netty 管理 (`netty`) | 开发调试 | 客户端在线列表、MQTT 统计 |
| 规则引擎 (`ruleengine`) | 自动化 | 规则脚本等 |
| 数据中心 (`dataCenter`) | 数据 | 历史记录、数据分析 |
| 视频中心 (`video`) | 视频 | SIP 设备/通道/流媒体/播放 |
| 系统管理 (`system`) | 基座 | 若依标准管理 |
| 系统监控 (`monitor`) | 运维 | 在线/定时任务/Druid/服务/缓存 |
| 系统工具 (`tool`) | 开发 | 表单构建/代码生成/系统接口 |

> 菜单与路由是动态加载的：前端只内置静态路由，其余根据登录用户角色从后端 `getRouters` 拉取，再 `router.addRoutes`。因此“菜单对不上代码”时先查 `sys_menu` 的 component/path。

## 2. IoT 功能与界面

### 2.1 产品域

| 界面 | 视图 | 后端 Controller |
| --- | --- | --- |
| 通用物模型模板 | `views/iot/template` | `ThingsModelTemplateController` |
| 产品分类 | `views/iot/category` | `CategoryController` |
| 产品管理/物模型 | `views/iot/product`（含 product-edit、product-things-model） | `ProductController`、`ThingsModelController` |
| 产品授权码 | `views/iot/product/product-authorize` | `ProductAuthorizeController` |
| 采集点模板 | `views/iot/template/parameter` | `ThingsModelTemplateController` |

### 2.2 设备域

| 界面 | 视图 | 后端 Controller |
| --- | --- | --- |
| 设备列表/编辑 | `views/iot/device` | `DeviceController` |
| 设备详情-监控 | `device-monitor.vue` | `DeviceRuntimeController` |
| 运行状态 | `running-status.vue` | `DeviceRuntimeController#runState` |
| 服务下发/回执 | `clientDetails`、`DeviceRuntimeController#service/invoke` | `FunctionInvokeImpl` |
| 设备定时 | `device-timer.vue` | `DeviceJobController` + Quartz |
| 设备分享/用户 | `device-user.vue`、`user-list.vue` | `DeviceUserController` |
| 设备分组 | `views/iot/group` | `GroupController` |
| 事件日志 | `views/iot/log` | `EventLogController` |
| 数据监控历史 | `views/dataCenter/history` | `DataCenterController` |

### 2.3 调试与工具

| 界面 | 视图 | 能力 |
| --- | --- | --- |
| Netty 客户端 | `iot/netty/clients.vue` | 查看 Broker 在线客户端、强制下线 |
| MQTT 统计 | `iot/netty/mqtt.vue` | 收发/订阅/保留消息计数 |
| 模拟/调试 | 工具接口 `/iot/tool/...` | 注册、认证测试、主题列表、协议编解码、SDK 生成 |

### 2.4 规则与场景

- `views/iot/scene/script.vue`：规则脚本（数据流脚本）维护；
- `views/iot/scene` 其他页面：场景联动设计入口（源码能力与 UI 的完整度有差异，务必按“技术篇-规则引擎”核验）。

## 3. 视频中心

| 功能 | 视图 | Controller/模块 |
| --- | --- | --- |
| 媒体服务器配置 | `iot/sip/mediaServer*` | `MediaServerController` |
| SIP 配置/设备 | `iot/sip/sipconfig`、`index` | `SipConfigController`、`SipDeviceController` |
| 通道/播放/云台 | `iot/sip/channel`、`PlayerController`、`PtzController` | sip-server 模块 |

## 4. 系统管理/监控/工具

这些基本是若依原生能力：

- 系统管理：用户/角色/菜单/部门/岗位/字典/参数/通知公告/新闻资讯；
- 系统监控：在线用户/定时任务/数据监控(Druid)/服务监控(oshi)/缓存监控(Redis)；
- 系统工具：表单构建、代码生成（`fastbee-generator`）、系统接口（Swagger）。

## 5. “移动端”在仓库里的体现

仓库没有 uniapp 工程，但预留了这些接口/表：

- `wechat`：微信 App/小程序登录与绑定；
- `oauth_client_details` + `OauthClientDetailsController`：OAuth 云云对接配置；
- `social_user/platform`：JustAuth 三方登录；
- `app_language`、`app_preferences`：App 语言与偏好；
- `translate/*`：菜单、字典、物模型等多语言翻译表。

移动端真正的 UI 仓库是官方 `fastbee-app`，与这些 REST/表结构配套。

## 6. 从菜单到源码的“三步定位法”

1. 打开 `sys_menu` 找到菜单的 `component`，例如 `iot/device/index`；
2. 在 `vue/src/views/iot/device/index.vue` 找到按钮调用的 `src/api/iot/device.js`；
3. API 文件中的 URL 指向后端 Controller，例如 `/iot/device` → `DeviceController` → `IDeviceService` → `DeviceServiceImpl`。

这个方法适合任何若依系项目，可以立刻建立“前端按钮 → REST → Service → Mapper/SQL”的调用链地图。

## 7. 小结

功能地图告诉我们：FastBee 的“业务”不是神秘的新概念，而是**标准 IoT 建模 + 若依管理后台**的组合。掌握菜单后，下一步应转入[典型业务流程](/posts/fastbee-biz-5/)，把散落的菜单串成一条端到端主流程。

