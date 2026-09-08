---
title: "测试现状评估"
published: 2026-09-08
description: "以当前 master 分支为准，逐项核验后结论如下："
tags: [FastBee,软件测试]
category: FastBee
draft: false
slug: fastbee-test-1
---
# 测试现状评估

## 1. 仓库内自动化测试盘点

以当前 `master` 分支为准，逐项核验后结论如下：

| 类别 | 结果 | 说明 |
| --- | --- | --- |
| 后端单元测试 | **仅 1 个测试类** | `springboot/fastbee-common/src/test/java/.../html/RichTextSanitizerTest.java` |
| 后端集成测试 | 0 | 未发现 `@SpringBootTest` 等 |
| 前端测试 | 0 | 无 Jest/Vitest/Mocha 配置与用例 |
| E2E 测试 | 0 | 无 Playwright/Cypress |
| CI 流水线 | 0 | 无 `.github/workflows`、Jenkinsfile、GitLab CI |
| 覆盖率工具 | 0 | 无 JaCoCo 等配置 |
| 测试报告/测试文档 | 0 | 仓库无 test 目录文档 |

唯一的测试类是最近一次安全修复提交（`284b714c fix(漏洞): 新闻、通知加xss过滤`）新增的，验证 `RichTextSanitizer` 对 XSS 的净化行为，使用 JUnit 4 + Assert：

```java
@Test
public void shouldRemoveExecutableContent() {
    String sanitized = RichTextSanitizer.sanitize(input);
    Assert.assertFalse(sanitized.contains("<script"));
    ...
}
```

该测试覆盖了富文本净化的关键安全场景（script/事件属性/javascript URL/iframe/svg、白名单保留、幂等与空值），是仓库中质量示范价值最高的测试文件。

## 2. 仓库中的“Test”不等于自动化测试

源码中还有多个以 Test 命名的类，容易造成误判：

```text
fastbee-admin/.../tool/TestController.java、TestController2.java
fastbee-http/.../controller/TestUploadController.java、TestDownloadController.java、TestAsyncController.java
fastbee-http/.../client/TestInterceptorClient.java
```

它们是**功能示例/接口演示**（Forest HTTP 客户端示例、上传下载异步示例），不是测试用例。

## 3. 手工/半自动验证的现状线索

虽然没有测试代码，但项目提供了大量“便于手工联调”的能力：

- 演示数据：`fastbee.sql` 内置产品、设备、物模型、脚本；
- Netty 管理页：在线客户端查看、强制下线、MQTT 统计；
- `ToolController`：注册、认证、主题、编解码、SDK 生成等工具接口；
- 模拟设备主题与前端模拟入口（property/simulate）；
- Swagger 接口文档；
- 规则脚本日志（script/scene logger）。

可以推断：项目实际的“测试”主要集中在**手工功能验证 + 真实硬件联调**，自动化几乎空白。

## 4. 质量风险清单

基于代码分析，当前最大的测试空白对应以下风险：

1. **数据链路正确性**：属性上报 → 物模型校验 → Redis → MySQL/TDengine，全链路无自动化，回归靠人肉；
2. **MQTT 协议符合性**：QoS、Retain、Will、通配符、心跳、会话收敛是手写实现，没有协议测试；
3. **权限/越权**：多租户字段过滤散落在 service 代码，194+ 个 `@PreAuthorize` 只验证“有没有登录/权限点”，不验证数据级隔离；
4. **并发与重连**：设备重复 clientId、断线重连、掉线补报等场景无自动化；
5. **DB 方言**：5 套 SQL 是否与代码一致，无迁移/兼容测试；
6. **安全**：XSS/认证/密钥等依赖人工确认；
7. **前端**：物模型编辑 JSON 合法性、动态路由权限等无回归。

## 5. 当前可以立刻做的三件事

1. **让已有测试跑起来**：`mvn -pl fastbee-common test`（或 IDE 运行），确认基线是绿的；
2. **在本地用演示数据做冒烟清单**：见本目录第 3、4 篇，形成手工回归表；
3. **把冒烟清单逐步自动化**：从最核心的“设备属性上报 → 落库”接口测试开始。

## 6. 后续章节

- [测试策略与测试分层](2-测试策略与测试分层.md)：为该项目设计可落地的质量策略
- [核心功能测试用例设计](3-核心功能测试用例设计.md)：业务功能测试点
- [MQTT 接入专项测试手册](4-MQTT接入测试手册.md)：协议与设备侧
- [接口、性能与安全测试](5-接口性能与安全测试.md)：REST、性能、安全
