---
title: "后端框架与启动装配"
published: 2026-09-08
description: "启动类："
tags: [FastBee,学习笔记]
category: FastBee
draft: false
slug: fastbee-tech-2
---
# 后端框架与启动装配

## 1. 启动入口与配置

启动类：

```text
springboot/fastbee-admin/src/main/java/com/fastbee/FastBeeApplication.java
```

`@SpringBootApplication` 默认扫描 `com.fastbee`，把 common/framework/service/server/mq/plugs 等包中的 `@Component` 全部装配进同一上下文。

### 1.1 配置文件

| 文件 | 作用 |
| --- | --- |
| `application.yml` | 公共配置：端口、Broker、数据源骨架、token、MyBatis-Plus、分页、XSS、上传路径 |
| `application-dev.yml` | 本地开发：MySQL/Redis localhost、SIP 关闭、Swagger `/dev-api` |
| `application-prod.yml` | 生产/容器：MySQL/Redis 指向 Docker 网络 IP（177.7.0.x）、SIP 开启、日志 error |
| `application-sql.yml` | 适配达梦等方言的备选配置 |

### 1.2 核心配置项

```yaml
server:
  port: 8080
  broker:              # 内置 Netty Broker
    broker-node: node1
    port: 1883
    websocket-port: 8083
    websocket-path: /mqtt
    keep-alive: 70
token:
  secret: ...           # JWT 密钥
  expireTime: 1440      # 分钟
```

本地开发版与容器版关键差异：

- dev 的 MySQL/Redis 密码是 `fastbee`；
- prod（容器）的 MySQL/Redis 密码是 `iot@admin123`，且数据库 IP 是 compose 子网固定地址。

## 2. Bean 装配结构

### 2.1 Web 与安全

框架模块 `fastbee-framework`：

- `security/filter/JwtAuthenticationTokenFilter`：解析 `Authorization: Bearer <jwt>`
- `security/handle/AuthenticationEntryPointImpl` 等：401/403 输出
- `SecurityConfig`（`SecurityConfig` 或类似）放行 `/login`、验证码、Swagger、工具类匿名接口
- `aspectj`：`@Log`、`@DataScope`、`@RepeatSubmit` 等切面
- `interceptor`：放行配置、防止重复提交

### 2.2 数据访问

- 数据源：`dynamic-datasource`，`master` 是默认主库；`taos` 是可选时序库；
- Mapper XML 位置：`classpath*:mapper/**/*Mapper.xml`；
- 类型别名：`com.fastbee.**.domain`；
- MyBatis-Plus 逻辑删除配置：`del_flag`（1 删除/0 正常），注意不同表逻辑值不完全一致；
- Druid：连接池、`stat,wall` 防火墙、慢 SQL 统计。

### 2.3 消息/服务线程

`application.yml` 中 Spring `task.execution.pool`（core 20 / max 200 / queue 3000）与 Netty 线程池分离：

- Netty boss/worker：`MQTTBootStrap` 通过 `NettyConfig.custom()` 配置；
- 业务线程：若开启 `businessCore`，独立 `ThreadPoolExecutor`；
- 异步任务：`@Async(FastBeeConstant.TASK.XXX)`。

## 3. 启动顺序（重要）

1. Spring 容器创建：数据源、Redis、MyBatis 等；
2. `MQTTBootStrap`（`@Order(10)`）：以 init/destroy 方式启动 `MqttServer`（1883）与 `WebSocketServer`（8083/mqtt）；
3. `DeviceJobServiceImpl@PostConstruct`：清空并重建 Quartz 任务（设备任务 + 系统任务）；
4. `ApplicationStarted@PostConstruct`：初始化时序数据库（TDengine 建库建超级表等）；
5. `StartBoot implements ApplicationRunner`（`@Order(2)`）：
   - 启动 `DeviceOtherListen`（常驻消费 `DeviceOtherQueue`）；
   - 初始化内部 `PubMqttClient`（默认连 `tcp://127.0.0.1:1883`）。

### 说明：内部 MQTT 客户端

`PubMqttClient` 的职责在不同模式不同：

- `enabled=false`（外接 EMQX）：连接外部 Broker，`connectComplete` 时订阅 `+/+/.../post` 全部上报主题，把消息交给 `subscribeCallback` → `DeviceOtherQueue` → `DeviceOtherMsgHandler`；
- `enabled=true`（内置 Broker）：该客户端主要用于**下行发布**（`/function/get`、`/status/post` 等），订阅逻辑关闭，设备上报由 Broker handler 直接进服务层。

这种“一套入口、两种模式”的双轨设计是阅读消息代码时最容易混淆的点，务必先弄清当前 `enabled` 值。

## 4. 约定与模式

### 4.1 返回模型

- `AjaxResult`：HTTP 业务统一返回 {code, msg, data}
- `TableDataInfo`：分页列表（`BaseController.startPage()` 配合 PageHelper）
- 异常体系：`ServiceException`、`DemoModeException` 等，由 `GlobalExceptionHandler` 统一转为 JSON

### 4.2 分层约定

虽然叫 IoT Service，但 `fastbee-iot-service` 中 Controller 在 open-api，Service 接口在 iot-service，Mapper 在 iot-service/mapper，形成：

```text
Controller → Service(interface) → ServiceImpl → Mapper(interface) → XML/SQL
```

### 4.3 数据权限约定

业务列表方法常见：

```java
SysUser user = SecurityUtils.getLoginUser().getUser();
// tenant/general 角色把自己的 userId 塞进查询对象
device.setUserId(user.getUserId());
```

## 5. 配置驱动能力清单

阅读配置即可了解平台能力开关：

- `server.broker.*`：Broker 端口/心跳
- `sip.enabled`：视频接入开关（含 SIP 域/ID/密码）
- `spring.datasource.dynamic.datasource.taos.enabled`：是否使用 TDengine
- `liteflow.rule-source-ext-data-map`：规则引擎表结构映射（chain=iot_scene、script=iot_script）
- `fastbee.demoEnabled`：演示模式限制
- `xss.enabled`：XSS 过滤
- `swagger.enabled`：接口文档

## 6. 常见启动故障排查

| 现象 | 排查点 |
| --- | --- |
| 启动后 1883 起不来 | `server.broker.port` 被占用；日志看 `MqttServer.initialize` |
| 规则脚本不生效 | 检查 LiteFlow 表名/字段与 `application.yml` 是否一致、`iot_script.enable` |
| 设备数据没入库 | 先确认 TSDB 开关：同时启用两个 TSDB 会被 `ApplicationStarted` 拒绝并回退 |
| 内部客户端循环重连 | Redis/`server.broker.port` 配置错误，看 `PubMqttClient.initialize` 日志 |

下一章：[认证与权限体系](/posts/fastbee-tech-3/)
