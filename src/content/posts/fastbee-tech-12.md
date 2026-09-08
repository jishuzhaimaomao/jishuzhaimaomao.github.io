---
title: "部署、运维与二次开发指引"
published: 2026-09-08
description: "README 推荐的正式路径："
tags: [FastBee,部署运维]
category: FastBee
draft: false
slug: fastbee-tech-12
---
# 部署、运维与二次开发指引

## 1. 官方推荐部署（Docker）

### 1.1 一键安装

README 推荐的正式路径：

```bash
sudo wget -c https://hub.fastbee.cn/resource/install.sh && bash ./install.sh
```

### 1.2 仓库自带 Compose 方式

仓库 `docker/data` 可直接上传到服务器 `/var/data`：

```bash
cp -r docker/data /var/data   # 目录：mysql/redis/java/nginx/zlmedia
cd /var/data && docker-compose up -d
```

服务编排（docker-compose.yml）：

| 容器 | 固定内网 IP | 对外端口 | 作用 |
| --- | --- | --- | --- |
| redis | 177.7.0.10 | 6379 | 缓存/会话 |
| mysql | 177.7.0.11 | 3306 | 主库（initdb 自动执行 fastbee.sql） |
| java | 177.7.0.12 | 8080/1883/8083/5061(udp) | FastBee 单体 |
| nginx | 177.7.0.13 | 80/443 | 前端静态页/API 反代/WSS |
| zlmedia | 177.7.0.15 | 8082/8443/554/1935/30000-30100 | GB28181 流媒体 |

nginx 反代：

```text
/            → vue 静态页
/h5          → 移动端 h5（自行放入）
/prod-api/   → http://java:8080/
/mqtt        → WebSocket http://java:8083/mqtt
```

注意：仓库自带的 `docker-compose.yml` 中 MySQL/Redis 口令为部署默认值，Java 生产配置（`application-prod.yml`）与其匹配；上传前应修改所有默认口令。

## 2. 本地开发部署

### 2.1 中间件

```text
MySQL 5.7+：建库 fastbee，导入 springboot/sql/fastbee.sql
Redis：默认密码 fastbee（dev 配置）
```

### 2.2 后端

```bash
cd springboot
mvn clean package -DskipTests
cd fastbee-admin/target
java -jar fastbee-admin.jar --spring.profiles.active=dev
```

浏览器 Swagger：`http://localhost:8080/swagger-ui/index.html`（默认 `/dev-api` 前缀）。

### 2.3 前端

```bash
cd vue
npm install --registry=https://registry.npmmirror.com
npm run dev
```

访问 `http://localhost`，后端代理到 `http://localhost:8080`。

## 3. 生产环境检查清单

1. 修改 admin 默认密码；
2. 修改 MySQL/Redis 口令与 `token.secret`；
3. 按需开启/关闭 SIP、Swagger、Druid 监控页面；
4. 配置 HTTPS（nginx ssl + WSS）；
5. 开启 XSS 过滤并跟踪漏洞公告；
6. 数据备份：MySQL dump + Redis AOF；
7. 上传目录单独挂载（`/uploadPath`），限制文件类型；
8. 日志接入采集（logback 落 `/logs`）。

## 4. 运维监控能力

| 能力 | 入口 | 说明 |
| --- | --- | --- |
| 系统监控 | `/system/monitor/server` | CPU/内存/磁盘（oshi） |
| Druid | `/druid` | SQL/连接池/慢 SQL |
| Redis 缓存 | `/monitor/cache` | 内存/键统计 |
| 在线用户 | `/monitor/online` | JWT 会话强退 |
| 定时任务 | `/monitor/job` | Quartz 管理 |
| MQTT 统计 | Netty 管理 | Broker 消息计数 |

## 5. 二次开发指引

### 5.1 新增一张业务表与增删改查

标准若依流程：

1. SQL 增加表与菜单；
2. `fastbee-iot-service` 建 domain/mapper，`resources/mapper/iot` 加 XML；
3. `fastbee-service`（接口）或 iot-service 建 Service；
4. `fastbee-open-api` 建 Controller；
5. 前端 `api` + `views` 页面 + 菜单授权；
6. 用代码生成器（`tool/gen`）可加速 2~5。

### 5.2 新增传输协议

阅读顺序：

1. `ServerType`（枚举）
2. `TopicType`（主题）
3. `jsonProtocolService`（编解码参考）
4. `IMessagePublish.funcSend` 增加分支
5. 若新增 TCP/UDP/CoAP Server：参考 `base-server` + `boot-strap` 的 Server 抽象

### 5.3 新增设备数据点

1. 产品物模型页新增属性/功能/事件；
2. 设备固件按 Topic 与 JSON Array 格式上报；
3. 平台自动识别并入库。

### 5.4 定制私有协议

优先使用数据流脚本（Groovy）做编解码，不改 Java；复杂场景再扩展 `SysProtocol` 注解体系。

### 5.5 消息链扩展点

| 需求 | 改动点 |
| --- | --- |
| 上下行过滤/改写 | 数据流脚本 |
| 消息落其他系统 | `IDataHandler` 实现/规则 action |
| 对接第三方 | `fastbee-http` + Forest 客户端 |
| 集群 Broker | 重构 `ClientManager/MessageStore/SessionManger` 为 Redis/消息总线实现 |
| OTA 完成 | 实现 `upGradeOTA` + 回执处理 |

## 6. RoadMap 对照

官方 `RoadMap.md` 规划与代码现状对应关系：

| 规划 | 现状 |
| --- | --- |
| 消息总线、协议插件化 | 已有 `IMessageStore`、RedisChannel、SysProtocol 雏形，未完整闭环 |
| 网关/子设备、透传、轮询 | 有子设备模型与 MODBUS 扩展字段，SDK 在外部仓库 |
| 规则引擎、场景联动 | 框架已实现，业务细节待打磨 |
| coap/tcp/udp/sip/snmp/tr069 | ServerType/枚举已预留，SIP 已实现，其余未完整落地 |
| 测试/CI/自动化 | 仓库内自动化测试非常少（详见测试篇） |

下一部分进入[测试篇](/posts/fastbee-test-1/)。
