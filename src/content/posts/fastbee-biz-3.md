---
title: "[FastBee·业务篇] 用户、租户、权限与设备分享"
published: 2026-09-08
description: "FastBee 的账号体系继承自若依（RuoYi-Vue），但为了设备所有权、多租户与分享做了二次演进。"
tags: [FastBee,FastBee·业务篇,设备与产品,权限与认证]
category: FastBee·业务篇
draft: false
slug: fastbee-biz-3
---
# 用户、租户、权限与设备分享

FastBee 的账号体系继承自若依（RuoYi-Vue），但为了设备所有权、多租户与分享做了二次演进。

## 1. 账号角色体系

初始化数据中定义了 4 类角色（`sys_role` + `sys_user_role`）：

| 角色 | 角色键 | 定位 |
| --- | --- | --- |
| 超级管理员 | `admin` | 平台级管理，能看全部资源 |
| 设备租户 | `tenant` | 拥有自己“租户”范围内的产品/设备/数据 |
| 普通用户 | `general` | 一般注册用户，默认绑定在个人名下 |
| 游客 | 未注册访问 | 只能访问白名单页面 |

用户注册默认分配普通用户角色（`ToolServiceImpl.register` 中 roleIds = {3L}）。系统在多张业务表上冗余了 `user_id/user_name` 与 `tenant_id/tenant_name`，目的就是让列表查询无需大量 JOIN 也能按所有者过滤。

## 2. 权限控制如何落到页面与接口

### 2.1 菜单-权限码

若依的 RBAC 由 `sys_menu` 表达目录(M)/菜单(C)/按钮权限(F)三层。菜单里保存：

- `path`/`component`：前端路由与视图文件
- `perms`：权限标识，例如 `iot:device:list`、`iot:service:invoke`

后端接口上使用 `@PreAuthorize("@ss.hasPermi('iot:device:list')")` 校验；前端用 `v-hasPermi` 指令控制按钮显隐。权限不是“代码写死”的，而是 DB 驱动（动态路由在 `vue/src/store/modules/permission.js` 中由后端菜单生成）。

### 2.2 数据范围

列表查询普遍带 `@DataScope` 注解，配合 `sys_role` 的 data_scope 与 `sys_dept` 过滤可见范围；IoT 侧再叠加 userId/tenantId 条件。

### 2.3 需要留意的越权风险点

README 中官方自己提示过“垂直越权漏洞修复参考”。在代码中可见：

- 设备新增时检查产品归属、用户/租户是否匹配（`DeviceServiceImpl.relateUser` 等）；
- 但不少 Controller 的权限点仍以“功能权限”为主，**数据级越权需要测试重点覆盖**，例如 A 用户用 B 用户的 serialNumber 直接调用状态查询/下发接口。

## 3. 租户模型怎么落地

与“独立租户隔离 Schema”的强多租户不同，FastBee 是**行级字段多租户**：几乎所有 IoT 表都有 `tenant_id/tenant_name`，业务代码在增删改查时写入或过滤这些字段，并通过 `sys_user.tenant_id` 判断归属。

这种设计的优点是单库单表简单、迁移友好；缺点是过滤必须靠代码自觉，容易漏，属于测试与安全加固的重点。

## 4. 设备所有权与分享

设备有两个层面的人：

- **所有者**：创建设备或认领设备的用户（`device_relate_user` 接口把设备挂到用户下）；
- **被分享者**：`iot_device_user` 记录分享关系。

分享授权时还会引用物模型的 `is_share_perm`：只有允许分享的物模型（属性/功能）才会展示给被分享者，避免把只读点开放成可控点。

前端入口：

- 设备列表 → 设备详情“设备用户/分享”；
- `vue/src/views/iot/device/device-user.vue`。

## 5. 三类“登录”要分清

FastBee 实际上有**三种完全不同的身份通道**，文档后续会反复使用：

| 通道 | 用户类型 | 认证材料 | 使用位置 |
| --- | --- | --- | --- |
| Web 登录 | 管理员/租户/普通用户 | 用户名密码 + 验证码 → JWT | `vue` + `/login` REST |
| MQTT Web/Phone 客户端 | Web 页面/App | clientId 以 `web`/`phone` 开头，password 携带 JWT | 前端实时数据、App |
| MQTT 设备客户端 | 设备 | clientId 形如 `S&编号&产品ID&用户ID`，用户名密码为产品凭据 | 设备接入 |

三者共同点：都会走 JWT/产品凭据校验，但认证逻辑在 `/iot/tool/mqtt/auth` 与 Broker 内置 `AuthService` 中分别实现（后者只是把校验委托给前者对应的 service 层）。

## 6. 小结

看 FastBee 权限要抓住三条线：

1. **功能权限**：菜单按钮（前端）+ `@PreAuthorize`（后端）；
2. **数据范围**：@DataScope + userId/tenantId 行级过滤；
3. **设备分享**：owner 与 guest 双轨 + 物模型级 sharePerm。

相关测试要点见[测试篇：核心功能用例设计](/posts/fastbee-test-3/)。

