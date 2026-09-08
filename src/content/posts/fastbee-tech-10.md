---
title: "前端 Vue 架构与实时通信"
published: 2026-09-08
description: "前端是基于 RuoYi-Vue 演进的管理后台，版本与依赖："
tags: [FastBee,前端开发,系统架构]
category: FastBee
draft: false
slug: fastbee-tech-10
---
# 前端 Vue 架构与实时通信

## 1. 技术栈

前端是基于 RuoYi-Vue 演进的管理后台，版本与依赖：

- Vue `2.6.12` + Vue Router `3.x` + Vuex `3.x`
- Element-UI `2.15.x`
- Axios `0.24`
- ECharts `5.4`
- MQTT.js `4.3.3`（实时通道）
- CodeMirror（脚本/JSON 编辑）、vue-qr、easywasmplayer（视频）等

构建工具：Vue CLI 4，路由懒加载，gzip 压缩。

## 2. 目录结构

```text
vue/src/
├── api/            # 按模块封装 REST（iot/system/monitor/tool）
├── assets/         # 样式/图标/大屏素材
├── components/     # 通用组件（Pagination/Editor/FileUpload/...）
├── directive/      # v-hasPermi 权限指令等
├── lang/           # zh-CN / en-US
├── layout/         # 后台布局（Sidebar/TagsView/Navbar）
├── lib/            # 公共库
├── plugins/        # cache/等
├── router/         # 静态路由
├── store/          # Vuex（permission/app/settings/tagsView/...）
├── utils/          # request/auth/ruoyi/dict/...
└── views/          # 页面
    ├── iot/        # 产品/设备/分组/日志/SIP/规则
    ├── dataCenter/ # 历史/分析
    ├── bigScreen/  # 可视化大屏
    ├── system/     # 用户/角色/菜单...
    ├── monitor/    # 运维监控
    └── tool/       # 表单/代码生成/Swagger
```

## 3. 请求层设计

`utils/request.js` 封装 Axios：

- baseURL = `process.env.VUE_APP_BASE_API`（dev `/dev-api`、prod `/prod-api`）；
- 请求头自动携带 `Authorization: Bearer <token>`；
- 带语言头（中英切换）；
- 1 秒内重复 POST/PUT 拦截；
- 响应统一处理 401 重新登录、500/601 提示；
- blob/arraybuffer（导出）原样返回。

环境文件：

```text
vue/.env.development → VUE_APP_SERVER_API_URL=http://localhost:8080/
vue/.env.production  → VUE_APP_MQTT_SERVER_URL 为空（自动取当前域名）
```

## 4. 路由与权限

`router/index.js` 只有基础路由；登录后：

```text
store.dispatch('GetInfo')      // 用户信息/权限
 → store.dispatch('GenerateRoutes')  // 由后端菜单构建
 → router.addRoutes(accessRoutes)    // 动态注册
```

菜单/权限来自后端 `sys_menu`，因此**前端本身不硬编码业务菜单树**。按钮权限 `v-hasPermi="['iot:device:add']"`。

## 5. MQTT 实时通道

前端引入 `mqtt` 依赖，与设备相关页面通过 `ws://host:8083/mqtt` 或 wss 建立连接：

- clientId 以 `web` 开头，密码携带 Web JWT；
- 订阅业务主题，如 `/ws/service` 及设备状态主题；
- 设备数据上报后，后端发布消息，前端实时刷新曲线/状态。

相关入口代码在 `views/iot/device/*`、`lib` 中的 MQTT 工具里；生产环境通过 Nginx `/mqtt` 反代到 `java:8083/mqtt`（WebSocket Upgrade）。

## 6. IoT 页面亮点

### 6.1 产品物模型编辑

`iot/product/product-things-model.vue`：

- 三类模型 Tab（属性/功能/事件）；
- 数据定义可切换类型并编辑 specs；
- 支持数组/对象嵌套、单位/范围/步长/枚举；
- 保存时生成 `things_models_json` 与逐条物模型。

### 6.2 设备监控

`device-monitor.vue`/`running-status.vue`：

- 调 `/iot/runtime/runState` 轮询或 MQTT 推送；
- ECharts 展示监测点曲线；
- 服务调用面板把“功能”渲染成按钮/下拉。

### 6.3 规则脚本

`iot/scene/script.vue` 集成 CodeMirror，支持 Groovy 等脚本编辑。

### 6.4 数据中心与大屏

- `dataCenter/history`：历史查询表格与曲线；
- `bigScreen/home.vue`：可视化大屏（@jiaminghi/data-view + ECharts）。

## 7. 构建与发布

```bash
npm install
npm run dev        # 开发：http://localhost:80
npm run build:prod # 生产：dist/
```

`vue.config.js` 已配置 devServer 代理与 gzip。发布产物直接放入 Nginx 静态目录（Docker 中 `/usr/share/nginx/html`）。

## 8. 阅读建议

按“先请求层、再权限、再一个页面闭环”的顺序读：

1. `main.js` / `App.vue`
2. `utils/request.js` + `utils/auth.js`
3. `permission.js`
4. `store/modules/permission.js`
5. `views/iot/device/index.vue` → `api/iot/device.js` → 后端 DeviceController

下一章：[数据库设计](/posts/fastbee-tech-11/)
