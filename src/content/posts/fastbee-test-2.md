---
title: "测试策略与测试分层"
published: 2026-09-08
description: "FastBee 属于“协议网关 + 业务平台 + 管理后台”复合系统，测试策略不能只做“增删改查”。建议围绕四条价值流设计："
tags: [FastBee,软件测试]
category: FastBee
draft: false
slug: fastbee-test-2
---
# 测试策略与测试分层

## 1. 目标

FastBee 属于“协议网关 + 业务平台 + 管理后台”复合系统，测试策略不能只做“增删改查”。建议围绕四条价值流设计：

1. **设备数据能进得来**（协议/编解码/入库）
2. **平台能把指令发得出去**（下发/回执/影子）
3. **用户能管得住**（权限/数据隔离/分享）
4. **系统跑得稳**（并发/重连/存储/安全）

## 2. 测试金字塔（适配本项目）

```text
                / E2E：前端点击→设备收到 → 少量关键链路
             / 系统级：Spring Boot 集成测试（内存库/容器）
          / 组件级：Service+MyBatis 事务/数据权限
       / 单元级：工具类、物模型解析、协议编解码、纯逻辑
    / 静态与代码评审：规范、依赖、安全扫描
```

由于项目现状自动化接近零，建议采用“**先补安全与纯逻辑单测，再补核心服务集成测试，最后做少量 E2E**”的增量路线。

## 3. 测试分层明细

### 3.1 静态/白盒层

推荐工具与检查项：

| 关注 | 工具 | 内容 |
| --- | --- | --- |
| Java 规范 | Checkstyle/SpotBugs | NPE、空集合、资源关闭 |
| 安全 | Semgrep/SpotBugs 安全规则、OWASP Dependency Check | XSS/硬编码密钥/漏洞依赖 |
| 前端 | ESLint（已有脚本）、vue-tsc（无 TS 时可省） | 未使用变量、危险 v-html |
| SQL | 方言对比脚本 | fastbee.sql 与其他 SQL 结构一致性 |

### 3.2 单元测试层

最适合本仓库的第一批单测对象：

1. `TopicsUtils`：topic 构建、通配符校验与匹配（`+`/`#` 边界）；
2. `JsonProtocolService`：JSON 数组 encode/decode、非法 JSON、空数组；
3. `ThingsModel` specs 解析与 `reportDeviceThingsModelValue` 分支（可用 Mockito mock Mapper/Redis）；
4. `RichTextSanitizer`（已有）与 `XssValidator`；
5. `ToolServiceImpl` 认证分支：S/E 认证、授权码绑定、自动建设备（mock DAO）；
6. `CronUtils`、CRC16/CRC8 工具；
7. `RedisKeyBuilder` 与 ValueItem 序列化。

注意根 pom 用 JUnit（常见若依默认 JUnit4）；若引入 JUnit5/Mockito，需统一版本。

### 3.3 组件/集成测试层

建议使用 Spring Boot Test + H2/Testcontainers：

| 测试目标 | 做法 |
| --- | --- |
| Controller 权限 | `@SpringBootTest` + MockMvc + `@WithMockUser` 式安全上下文 |
| 设备上报落库 | 注入 IDeviceService，Mock/内存 Redis + H2 MySQL 方言 |
| 认证接口 | 打真实 `/iot/tool/mqtt/auth`（本地 MySQL+Redis 环境） |
| 定时任务 | 直接调用 JobInvokeUtil，校验 actions 解析后是否调 publish 接口 |
| TSDB 切换 | 用 Testcontainers TDengine 验证 `saveDeviceLog` |

真实环境最简单的方式仍是“本地全套跑起来 + 手动/脚本 SQL 断言”，先把集成测试框架搭好再逐模块迁移。

### 3.4 E2E 层

用 Playwright/UI 自动化覆盖两条核心用户路径：

1. 登录 → 新建产品/物模型 → 添加设备；
2. 设备模拟上报 → 前端设备详情看到实时值。

设备侧模拟可使用 MQTT.js 脚本或 MQTTX 自动化。

## 4. 测试环境与数据

### 4.1 环境矩阵

| 环境 | 用途 | 数据 |
| --- | --- | --- |
| DEV | 日常开发自测 | 演示数据 + 开发者私有数据 |
| TEST | 集成/回归 | 全量演示数据 + 构造边界数据 |
| STAGE | 验收/性能 | 接近生产的配置与数据量 |
| PROD | 生产 | 不跑破坏性测试 |

### 4.2 数据工厂建议

由于表关系复杂（product→things→device→log），建议提供：

- 基础数据：1 个分类、1 个 JSON/MQTT 产品（已发布）、3 种物模型（属性/功能/事件）、直连设备、网关+子设备；
- 边界产品：禁用状态、未发布、GB28181；
- 授权码、分享用户、多租户用户。

## 5. 风险分级与优先级

| 优先级 | 模块 | 理由 |
| --- | --- | --- |
| P0 | 设备认证、属性上报、落库、状态机 | 一切价值的基础，故障影响最大 |
| P0 | Web 登录/RBAC/数据隔离 | 多租户平台的安全底线 |
| P1 | 指令下发、影子、定时任务 | 控制类业务，事故会造成设备误动作 |
| P1 | MQTT Broker（QoS/Retain/Will/并发连接） | 自研组件，正确性依赖专项测试 |
| P2 | 规则引擎脚本、场景 | 业务增益高但复杂度大，先数据流脚本后场景 |
| P2 | 视频接入 | 依赖硬件/模拟器，做协议级半自动测试 |
| P2 | 系统管理与运维页面 | 若依成熟度高，可降低优先级 |

## 6. 缺陷管理建议

- 建立用例-需求-缺陷关联矩阵；
- 协议问题与业务问题分开跟踪（topic/payload 层 vs 服务层）；
- 记录“真实硬件 + 网络环境”专项结果，避免只依赖本机模拟。

下一章：[核心功能测试用例设计](3-核心功能测试用例设计.md)
