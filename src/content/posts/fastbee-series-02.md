---
title: "02 物模型：平台的语义中枢"
published: 2026-09-08
description: "设备上报一串字节，平台怎么知道它是什么？物联网行业给出的答案不是让每台设备写死解析代码，而是引入物模型（Thing Model）。"
tags: [FastBee,物模型]
category: FastBee
draft: false
slug: fastbee-series-02
---
# 02 物模型：平台的语义中枢

## 从“设备在说什么”开始

设备上报一串字节，平台怎么知道它是什么？物联网行业给出的答案不是让每台设备写死解析代码，而是引入**物模型（Thing Model）**。

物模型是“设备数据语义的契约”：

- 平台先定义：这个产品有温度（属性）、有开关（功能）、会报异常（事件）；
- 设备只要按契约上报/订阅；
- 前端就能自动渲染成曲线、开关、按钮。

## FastBee 的三个模型类别

| 类别 | 代码 type | 方向 | 直觉理解 |
| --- | --- | --- | --- |
| 属性 Property | 1 | 数据点 | “现在是多少”：温度 25.6℃ |
| 功能 Function | 2 | 控制点 | “去做一件事”：打开开关 |
| 事件 Event | 3 | 瞬时通知 | “刚才发生了什么”：设备异常 |

这种三分类几乎和主流公有云 IoT 平台一致，学习价值很高。

## specs：让抽象落到可执行

`iot_things_model.specs` 是物模型最精彩的部分。看两条真实种子数据：

温度：

```json
{"type":"decimal","max":120,"min":-20,"step":0.1,"unit":"℃"}
```

档位：

```json
{"type":"enum","showWay":"select",
 "enumList":[
   {"text":"低速","value":"0"},
   {"text":"高速","value":"3"}]}
```

这里不仅描述了“类型”，还描述了**UI 怎么展示、输入怎么校验**。一个模型定义可以同时驱动：

- 后端数据校验；
- 前端表单（下拉/按钮/滑块）；
- 图表展示（isChart）；
- 历史存储开关（isHistory）；
- 实时监测（isMonitor）。

## 更深一层：数组、对象与采集

简单模型只适合单个传感器。FastBee 的网关产品演示了复杂建模：

```text
product 55（网关产品）下的功能 device：
type=array, arrayType=object, arrayCount=5
params 里定义了每路子设备的温度、开关、挡位...
```

这意味着一个“子设备”功能可以表达 5 路子采集。配上物模型扩展字段：

- `temp_slave_id`：Modbus 从机号；
- `reg_addr`：寄存器地址；
- `formula`：上报换算 `%s*10`；
- `reverse_formula`：下发反算；
- `code/parse_type/quantity`：功能码与解析类型。

FastBee 把 Modbus 的“采集点表”也物模型化了。也就是说，**物模型不仅是 UI 契约，还是工业采集的寄存器映射表**。

## 物模型在运行时的角色

设备上报时：

```java
// DeviceServiceImpl.reportDeviceThingsModelValue 片段
PropertyDto dto = thingsModelService.getSingleThingModels(productId, identity);
if (null == dto) {
    continue;   // 没定义的字段，直接忽略
}
```

这说明物模型是**白名单**：不是设备发什么平台存什么，而是**只有产品物模型定义过的 identifier 才被接受**。这个设计能有效防止脏数据灌库。

## 学习建议

动手做一次：

1. 打开 SQL，复制产品 41 的 `things_models_json`；
2. 用 JSON 格式化工具观察 properties/functions/events；
3. 在前端“产品物模型”页面做一次可视化编辑；
4. 在 `reportDeviceThingsModelValue` 打断点，看上报时如何与物模型交互。

下一站：[自研 MQTT Broker](/posts/fastbee-series-03/)
