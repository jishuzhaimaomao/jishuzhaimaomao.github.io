---
title: "12 把 FastBee 跑起来的部署实战"
published: 2026-09-08
description: "FastBee 的 Docker 部署本质是五个角色："
tags: [FastBee,部署运维]
category: FastBee
draft: false
slug: fastbee-series-12
---
# 12 把 FastBee 跑起来的部署实战

## 部署的本质

FastBee 的 Docker 部署本质是五个角色：

```text
MySQL（业务库）
Redis（缓存/会话/消息计数）
Java（FastBee 单体：Web + Broker + SIP）
Nginx（前端 + 反代 + WSS）
ZLMediaKit（视频流媒体，可选）
```

## 路线 A：仓库 Compose（最快）

### Step 1 准备目录

```bash
mkdir -p /var/data
cp -r docker/data/* /var/data/
cd /var/data
```

### Step 2 检查并启动

```bash
docker-compose config   # 先校验
docker-compose up -d
docker-compose ps
```

MySQL 首次启动会自动执行 `initdb/fastbee.sql`。

### Step 3 访问

```text
http://服务器IP/
admin / admin123
```

### Step 4 验证

```bash
# 首页/静态资源是否 200
curl -s -o /dev/null -w '%{http_code}' http://127.0.0.1/
# 或直接看 Java 容器日志（启动成功会打印 FastBee 的 ASCII Logo）
docker logs -f java
```

## 路线 B：源码本地跑

### 后端

```bash
# 1. 建库导数据
mysql -uroot -p -e "create database fastbee default charset utf8;"
mysql -uroot -p fastbee < springboot/sql/fastbee.sql

# 2. Redis（本地无密码或密码 fastbee）

# 3. 编译
cd springboot
mvn clean package -DskipTests

# 4. 启动（用 dev 配置）
java -jar fastbee-admin/target/fastbee-admin.jar --spring.profiles.active=dev
```

### 前端

```bash
cd vue
npm install --registry=https://registry.npmmirror.com
npm run dev
```

访问 `http://localhost`。

## 部署后必做的自测清单

### 1. 登录与菜单

- admin 登录成功；
- 菜单完整（设备管理/数据中心/视频/系统管理）。

### 2. 产品/设备闭环

- 打开“智能开关产品”，确认已发布；
- 添加测试设备；
- 用 MQTTX 按 `S&编号&41&1` 连接；
- 发布 `property/post` 数据；
- 在设备详情看到实时值与历史。

### 3. Broker

```bash
nc -zv 127.0.0.1 1883
# WebSocket 端口
curl -i http://127.0.0.1:8083/mqtt   # 预期 400/426 类 WebSocket 握手响应
```

### 4. Redis/MySQL

```bash
redis-cli -a <口令> keys 'device:*' | head
mysql -e "select count(*) from iot_device_log;"
```

### 5. 视频（可选）

配置 `media_server` 与 `sip_config`，用模拟器验证 REGISTER。

## 生产加固清单（对照检查）

| 项 | 操作 |
| --- | --- |
| 账号 | 改 admin 密码；关闭演示模式 `demoEnabled=false` |
| 数据库口令 | 改 MySQL/Redis 口令并同步 yml |
| JWT | 换长随机 `token.secret` |
| HTTPS | nginx 配证书，/mqtt 用 WSS |
| 监控暴露 | 生产关闭 Druid 页面与 Swagger（如需则加认证） |
| XSS/漏洞 | 跟进官方安全提交，启用 XSS 过滤 |
| 备份 | MySQL 定时 dump + Redis AOF |
| 日志 | 保留 `/logs`，接入采集 |
| 端口 | 1883/8083/5061 按需用防火墙收敛 |

## 常见问题速查

| 现象 | 原因/排查 |
| --- | --- |
| 容器启动后 Java 反复重启 | 连不上 MySQL/Redis；看 `docker logs java` |
| 设备连接 1883 失败 | 检查 Broker 是否启动、clientId 格式、产品已发布 |
| 数据不上报表 | 物模型未定义 / TSDB 配置冲突 / topic 拼错 |
| 前端接口 404 | nginx `/prod-api` 反代、baseURL、后端 profile 不一致 |
| 规则脚本不执行 | `iot_script.enable`、事件编号、applicationName |
| 视频看不到 | SIP 配置、ZLMediaKit 端口/密钥、网络互通 |

## 最后的话

把 FastBee 跑起来只是开始。真正的“吃透”是：

1. 能独立画出它的消息链路；
2. 能指出每个状态的存储位置；
3. 能说出它“为什么这样设计”；
4. 能诚实指出哪些能力还没闭合；
5. 能动手补一个 TODO 闭环。

建议完成部署后，回到[业务篇](../01-业务篇/1-项目全景-定位与版本.md)或[技术篇](../02-技术篇/1-总体架构与模块清单.md)按章节精读，再用[测试篇](../03-测试篇/3-核心功能测试用例设计.md)的用例给自己出卷。祝你学习顺利！
