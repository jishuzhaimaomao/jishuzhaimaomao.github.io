---
title: "接口、性能与安全测试"
published: 2026-09-08
description: "仓库中 Controller 共 37 个，集中在 fastbee-open-api："
tags: [FastBee,软件测试]
category: FastBee
draft: false
slug: fastbee-test-5
---
# 接口、性能与安全测试

## 1. REST 接口范围

仓库中 Controller 共 37 个，集中在 `fastbee-open-api`：

```text
常规业务：device/product/thingsModel/category/group/deviceUser/log/eventLog/functionLog/...
运行时：   runtime/deviceRuntime（runState/funcLog/service/invoke）
数据中心： datacenter/DataCenter
视频：     media/Player/Ptz/SipDeviceChannel/MediaServer/ZmlHook
工具/认证： ToolController（register/mqtt auth/webhook/upload/genSdk/...）
系统：     AuthResource/OauthClientDetails/SocialLogin/WeChat/AppPreferences
```

## 2. 接口自动化测试设计

### 2.1 通用校验

每个接口建议检查：

1. 未登录 → 401；
2. 登录无权限 → 403（演示模式下部分接口行为不同）；
3. 参数缺失/类型错误 → 400/业务错误码；
4. 不存在的资源 → 404/空数据而非崩溃；
5. 分页参数（pageNum/pageSize）边界；
6. 导出 Excel 字段与表头正确。

### 2.2 关键接口用例表

| 接口 | 测试重点 |
| --- | --- |
| POST /iot/runtime/service/invoke | 参数校验、影子开关、messageId 返回、功能日志 |
| GET /iot/runtime/runState | serialNumber/type/productId 组合、非法类型 |
| POST /iot/device | serialNumber 唯一、租户归属、is_authorize 联动 |
| PUT /iot/device | 状态/位置更新是否触发状态消息 |
| GET /iot/device/statistic | 多库统计正确性 |
| POST /iot/tool/mqtt/auth | S/E/坏 clientId/授权码/超时密码 |
| POST /iot/tool/upload | 文件大小/类型限制、路径穿越 |
| GET /iot/tool/genSdk | 下载内容与设备连接参数一致 |
| 数据中心接口 | 时间范围、跨租户隔离 |
| 规则脚本 CRUD | enable 切换、脚本语法验证（当前是否校验） |

### 2.3 工具建议

- 接口文档：Swagger（启动后 `/swagger-ui/index.html`）；
- 集合管理：Postman/Apifox，把环境变量（baseUrl、token、产品/设备 ID）参数化；
- 自动化：JMeter/HttpRunner 可执行批量断言；
- 数据库断言：查 `iot_device_log`、Redis Hash、FunctionLog。

## 3. 性能测试

### 3.1 分层性能目标

| 层 | 指标示例（以学习环境为准） |
| --- | --- |
| REST | P95 < 300ms（简单列表）；登录限流生效 |
| MQTT Broker | 数千连接、数千 msg/s 时 CPU/内存可接受 |
| 上报入库 | Redis 不积压、日志无大幅延迟增长 |
| 规则脚本 | 脚本执行耗时计入消息链路 |
| 大屏/历史查询 | 大数据量曲线接口 SQL 可接受 |

### 3.2 方法

1. JMeter 压 REST（登录后带 token）；
2. 自写 Go/Python MQTT 压测脚本或使用 `mqtt-bench`/`emqtt-bench`；
3. 监控：Druid 慢 SQL、Redis INFO、JVM（VisualVM/Arthas）、Netty 线程；
4. 压测前用 `sql/clear` 或独立库，避免污染演示数据。

### 3.3 关注点

- `DataHandlerImpl.reportData` 逐条落库（TDengine 用 1ms 间隔）在**大批量**上报时是否成为瓶颈；
- `reportDeviceThingsModelValue` 每个 item 都查 Redis/DB 物模型，注意缓存命中；
- `ClientManager.pubTopic` 遍历匹配的复杂度；
- 定时任务线程池与业务线程池是否被长任务占满。

## 4. 安全测试

### 4.1 代码层已知修复

近期提交显示安全是活跃议题：

- `284b714c fix(漏洞): 新闻、通知加xss过滤`：新增 `RichTextSanitizer` + 单元测试；
- `a66e9e14 fix(系统): 解决xss前端问题`：前端新增 `utils/security.js` 与 SafeHtml 组件。

回归测试应包含富文本/新闻/通知的 script、事件、javascript URL、iframe、svg 等向量。

### 4.2 测试用例

| 类别 | 用例 |
| --- | --- |
| 认证 | 弱密码、验证码重用/暴力、JWT 过期/伪造、JWT 密钥泄露影响 |
| 授权 | 垂直越权（普通用户调用管理接口）、水平越权（跨用户设备）、MQTT 越权订阅 |
| 注入 | 搜索/列表 SQL 注入、orderBy 注入、XSS、富文本存储型 XSS |
| 上传 | 文件类型绕过、超大文件、路径穿越、恶意内容 |
| MQTT | 匿名连接、爆破账号、超长 clientId/topic、topic 通配滥用、遗嘱滥用 |
| 敏感信息 | 产品密钥返回、日志打印凭据、Redis/MySQL 默认口令 |
| 配置 | Swagger/Druid 生产是否关闭、上传目录权限、SSL |

### 4.3 工具

- OWASP ZAP/Burp：Web 扫描；
- `nuclei`/自写脚本：探测默认口令、未授权接口；
- `semgrep`/SpotBugs：代码扫描；
- `git log`：持续跟踪安全修复 commit，建立回归清单。

## 5. 稳定性/容错测试

| 场景 | 方法 | 预期 |
| --- | --- | --- |
| 重启恢复 | kill Java 后拉起 | Redis 连接恢复、Broker 重启、设备重新连后状态正确 |
| 依赖故障 | 停 MySQL/Redis | 报错友好、进程不雪崩（当前实现可能抛错，记录并优化） |
| 重复消息 | 同 payload 重复发 | 幂等性按业务定义验证 |
| 时间跳变 | NTP 同步/系统时间前移 | 日志时间、TDengine 主键冲突处理 |
| 大数据量 | 万级设备日志 | 历史查询/导出不 OOM |
| 数据库切换 | 启用 TDengine/MySQL 对比 | 同一套接口结果一致 |

## 6. 质量门禁建议

为该项目建议的 CI 门禁（按投入从小到大）：

1. `mvn test`（含已有单测）；
2. 编译 + 打包；
3. 新增核心模块单测覆盖率（如工具/协议层）；
4. 冒烟集成测试（认证→上报→落库）；
5. 前端 lint；
6. 依赖漏洞扫描。

这部分做完后，建议阅读[系列学习文章](/posts/fastbee-series-00/)巩固体系化认知。
