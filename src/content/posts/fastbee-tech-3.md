---
title: "认证与权限体系（Web + MQTT）"
published: 2026-09-08
description: "GET  /captchaImage         → 返回 uuid + 图形/数学验证码"
tags: [FastBee,MQTT,权限与认证]
category: FastBee
draft: false
slug: fastbee-tech-3
---
# 认证与权限体系（Web + MQTT）

## 1. Web/移动端登录

### 1.1 登录流程（若依标准）

```text
GET  /captchaImage         → 返回 uuid + 图形/数学验证码
POST /login                → 校验验证码 + 用户名密码
                           → 登录成功记录 logininfor
                           → 返回 token（JWT）
后续请求头： Authorization: Bearer <token>
```

密码使用 `SecurityUtils.encryptPassword`（BCrypt）。登录接口还支持注册、找回密码等工具接口（`ToolServiceImpl`）。

### 1.2 JWT 过滤

`JwtAuthenticationTokenFilter` 从 Header 取 token → 解析用户 → 放 SecurityContext。Redis 缓存登录用户（登录在线列表 `monitor/online` 可强退）。

### 1.3 前端动态权限

登录后调用 `getInfo` 拿用户角色/权限，调用 `getRouters` 拿菜单，动态注册路由；按钮用 `v-hasPermi` 控制。

## 2. REST 接口授权

Controller 常用注解：

```java
@PreAuthorize("@ss.hasPermi('iot:device:list')")
```

权限码与菜单 `sys_menu.perms` 一一对应；未配置权限码的接口在开源版中可能仅依赖登录态，需要测试关注。

## 3. MQTT 三端认证

`AuthService`（Broker 内部）按 clientId 前缀分流：

```java
if (clientId.startsWith("server")) { ... 平台内部账号 ... }
else if (clientId.startsWith("web") || clientId.startsWith("phone")) { ... JWT ... }
else { ... 设备认证 ... }
```

### 3.1 设备认证：clientId 编码规则

设备 clientId 是**用 `&` 分隔的四个段**：

```text
<认证类型>&<serialNumber>&<productId>&<userId>
认证类型：S = 简单认证；E = 加密认证
```

后端 `ToolServiceImpl.clientAuth`：

1. 解析 clientId，校验四段非空；
2. 查 `iot_device`+`iot_product`（`selectProductAuthenticate`）；
3. 产品必须是已发布（status=2）；
4. S：比对产品 `mqtt_account/mqtt_password`；
5. E：用产品 `mqtt_secret` 解密动态密码（`AESUtils`）、校验有效期；
6. 若启用授权码 `is_authorize=1`，校验 `iot_product_authorize` 并绑定；
7. 设备已存在：检查未禁用；不存在：自动创建设备（`insertDeviceAuto`）。

### 3.2 内部客户端/Web/App

- `server*`：用户名密码与 `MqttClientConfig` 比较；
- `web*`/`phone*`：password 字段传 `Bearer JWT`，由同一 `token.secret` 解析。

前端实时通道使用的 clientId 以 `web` 开头，并在 MQTT.js 连接时把 JWT 作为 password，这是前端能实时收发设备数据的认证基础。

## 4. 三方与开放生态登录

| 能力 | 位置 | 说明 |
| --- | --- | --- |
| JustAuth 社交登录 | `/auth/render|callback|login` + `iot_social_platform` | Gitee/Gitee 等平台配置管理 |
| 微信 App/小程序 | `/wechat/*` | mobileLogin/miniLogin/绑定 |
| OAuth2 Client（云云对接） | `/iot/clientDetails` + `oauth_client_details` | 管理第三方应用凭据 |
| 工具认证（兼容 EMQX） | `/iot/tool/mqtt/auth`、`/mqtt/authv5` | 给外部 Broker 做 HTTP Auth |
| EMQX WebHook | `/iot/tool/mqtt/webhook`、`webhookv5` | 外部 Broker 上下线事件转业务 |

## 5. 数据范围（DataScope）

若依 `@DataScope` 切面会解析用户角色与部门数据权限，自动拼接到 SQL（通过 MyBatis 参数）。IoT 业务叠加了租户与用户过滤。

业务代码中的常见模式：

```java
// 仅当角色是 tenant/general 时限制为自己的 userId
if (roleKey.equals("tenant") || roleKey.equals("general")) {
  entity.setUserId(loginUser.getUserId());
}
```

这意味着同样一份“设备列表”接口，admin 看到全部，租户/普通用户只看到自己的数据。

## 6. 认证链路中的薄弱点（也是测试重点）

1. **设备端 clientId 可预测**：serialNumber/productId/userId 均为明文字段，必须依赖 username/password 真正鉴权；
2. **产品密钥** `mqtt_secret` 明文存在 `iot_product`，生产需数据库加密/脱敏；
3. **Web MQTT 通道**：JWT 放在 MQTT password 中，TLS 未启用时有被嗅探风险；
4. **垂直越权**：只校验“登录者”不校验“资源归属”的接口需要逐一排查；
5. **默认口令**：admin/admin123、容器 MySQL/Redis 口令为仓库默认值，公网部署必须修改。

相关文件索引：

```text
AuthService                  springboot/fastbee-server/mqtt-broker/.../auth/AuthService.java
ToolController               springboot/fastbee-open-api/.../controller/ToolController.java
ToolServiceImpl              springboot/fastbee-service/fastbee-iot-service/.../impl/ToolServiceImpl.java
JWT Filter                   springboot/fastbee-framework/.../security/filter/JwtAuthenticationTokenFilter.java
```

下一章：[自研 MQTT Broker 实现解析](/posts/fastbee-tech-4/)
