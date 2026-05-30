---
categories: ["nasa-cFS学习"]
tags: ["Linux", "nasa-cFS" , "OS"]
title: "GroundSystem功能详解"
date: 2026-05-26
draft: True
---
---

# cFS GroundSystem 功能测试报告 (DeepSeek v4 Pro辅助)

---

## 〇、前置操作：TO_LAB 订阅表修复

> ⚠️ **重要**：cFS 开源 Bundle 默认配置下，TO_LAB（遥测输出应用）的订阅表 to_lab_sub.c 中所有订阅条目均被注释。这导致即使发送 Enable Tlm 指令，地面站也无法收到任何遥测数据。

**修复方法**：编辑 `apps/to_lab/fsw/tables/to_lab_sub.c`，取消 `/* cFE Core subscriptions (examples) */` 下方所有条目的 `/*` `*/` 注释符，使 TO_LAB 订阅以下遥测频道：

| 订阅的遥测包 | 消息 ID | 说明 |
|-------------|---------|------|
| CFE_ES_HK_TLM | 0x800 | 执行服务健康状态 |
| CFE_EVS_HK_TLM | 0x801 | 事件服务健康状态 |
| CFE_SB_HK_TLM | 0x803 | 软件总线健康状态 |
| CFE_TBL_HK_TLM | 0x804 | 表管理健康状态 |
| CFE_TIME_HK_TLM | 0x805 | 时间服务健康状态 |
| CFE_TIME_DIAG_TLM | 0x806 | 时间服务诊断 |
| CFE_SB_STATS_TLM | 0x80A | 软件总线统计 |
| CFE_TBL_REG_TLM | 0x80C | 表注册信息 |
| CFE_EVS_LONG_EVENT_MSG | — | 长事件消息 |
| CFE_EVS_SHORT_EVENT_MSG | — | 短事件消息 |
| CFE_ES_APP_TLM | 0x80B | 应用注册信息 |
| CFE_ES_MEMSTATS_TLM | 0x810 | 内存统计 |

然后 `make native_std.install` 重新编译，重启 `./core-cpu1`。

---

## 一、系统功能总览

### 1.1 GroundSystem 是什么？

GroundSystem 是 cFS 的**地面站模拟软件**，用于在开发/测试阶段模拟地面指控中心的功能。它通过 UDP/IP 与 cFS 核心通信。

### 1.2 与实际航天系统的对应关系

| 实际航天元素 | GroundSystem 中的对应 | 说明 |
|-------------|----------------------|------|
| **地面指控中心** | GroundSystem.py 主窗口 | 控制上行指令、监控下行遥测 |
| **上行链路 (Uplink)** | cmdUtil → UDP 1234 → CI_LAB | 指令从地面发往卫星 |
| **下行链路 (Downlink)** | TO_LAB → UDP 2234 → RoutingService | 遥测从卫星发往地面 |
| **指令编码器** | cmdUtil.c (C 程序) | 将指令参数组装为 CCSDS 标准包 |
| **遥测解码器** | GenericTelemetry.py | 按字段定义解析 CCSDS 遥测包 |
| **卫星上的指令分发** | SB 软件总线 | 发布/订阅模式路由消息 |
| **卫星上的遥测收集** | SB + TO_LAB | 各应用发布遥测 → TO_LAB 汇总下发 |
| **航天器标识** | Spacecraft1 (自动检测 IP) | 多星场景下区分不同航天器 |

### 1.3 数据流架构

```
┌─────────────────────────────────────────────────────────────┐
│                    GroundSystem (地面站)                     │
│                                                             │
│  ┌──────────┐  指令  ┌──────────┐  UDP:1234  ┌──────────┐  │
│  │ 指令 GUI │ ─────→ │ cmdUtil  │ ────────→  │ CI_LAB   │  │
│  │(PyQt5)   │        │  (C程序)  │           │ (指令注入) │  │
│  └──────────┘        └──────────┘           └────┬─────┘  │
│                                                   │        │
│  ┌──────────┐  遥测  ┌────────────┐  UDP:2234     │ SB     │
│  │ 遥测 GUI │ ←───── │RoutingSvc  │ ←────────── │ 软件总线│  │
│  │(PyQt5)   │  ZMQ   │ (Python)   │              │        │  │
│  └──────────┘        └────────────┘           ┌──┴──────┐ │
│                                                │ TO_LAB  │ │
│                                                │(遥测输出)│ │
│                                                └─────────┘ │
└─────────────────────────────────────────────────────────────┘
```

---

## 二、窗口逐一说明

### 2.1 Main Window（主窗口）

**功能**：地面站控制中枢，启动/停止各子系统。

| 控件 | 功能 | 说明 |
|------|------|------|
| **Start Command System** | 打开指令系统窗口 | 启动上行链路 GUI |
| **Start Telemetry System** | 打开遥测系统窗口 | 启动下行链路 GUI |
| **Selected IP Address** | 航天器 IP 下拉框 | 自动检测活跃航天器 |
| **TLM Header Version** | 遥测包头版本选择 | CCSDS V1/V2/Custom |
| **CMD Header Version** | 指令包头版本选择 | CCSDS V1/V2/Custom |

### 2.2 Command System Main Page（指令系统主页）

**功能**：cFS 各子系统的指令入口。

**表格列说明**：

| 列名 | 含义 | 示例值 |
|------|------|--------|
| Subsystem | 子系统名称 | Executive Services |
| Packet ID | CCSDS 消息 ID | 0x1806 |
| Destination | 目标 IP 地址 | 127.0.0.1 |

**按钮说明**：

| 按钮 | 功能 |
|------|------|
| **Display Page** | 打开该子系统完整指令列表 |
| **Enable Tlm** | 快捷按钮——开启遥测下行（仅 TO_LAB 行） |
| **各 No-Op 按钮** | 快捷按钮——发送空操作测试指令 |

**子系统列表与对应模块**：

| 子系统 | Packet ID | cFS 源码模块 | 真实功能 |
|--------|-----------|------------|---------|
| Executive Services | 0x1806 | `cfe/modules/es/` | 应用管理、系统监控 |
| Software Bus | 0x1803 | `cfe/modules/sb/` | 消息发布/订阅中枢 |
| Table Services | 0x1804 | `cfe/modules/tbl/` | 配置表管理 |
| Time Services | 0x1805 | `cfe/modules/time/` | 系统时钟同步 |
| Event Services | 0x1801 | `cfe/modules/evs/` | 事件日志 |
| Command Ingest | 0x1884 | `apps/ci_lab/` | 指令注入（Lab） |
| Telemetry Output | 0x1880 | `apps/to_lab/` | 遥测输出（Lab） |
| Sample App | 0x1882 | `apps/sample_app/` | 示例应用 |

> 📝 **验证点**：逐行点击 Display Page，记录每个子系统有哪些可用指令。

### 2.3 Display Page（指令详情页）

**功能**：显示某个子系统的所有可用指令。

每个子系统指令页包含若干可发送的指令，以 ES 为例：

| 指令 | 功能码 | 参数 | 说明 |
|------|--------|------|------|
| Noop | 0x00 | 无 | 空操作，测试连通性 |
| Reset Counters | 0x01 | 无 | 重置计数器 |
| Restart App | — | AppName | 重启指定应用 |
| Reload App | — | AppName | 重新加载应用 |
| Delete App | — | AppName | 删除/停止应用 |
| ... | | | |

> 📝 **验证点**：打开每个子系统的 Display Page，截图记录所有指令及其参数。

### 2.4 Parameter Dialog（参数对话框）

**功能**：为需要参数的命令输入参数值。

**示例——Enable Tlm 参数**：

| 字段 | 含义 | 输入示例 |
|------|------|---------|
| dest_IP | 遥测目标 IP 地址 | `127.0.0.1` |

> 📝 **验证点**：找出哪些指令需要参数，记录参数名和数据类型。

### 2.5 Telemetry System Main Page（遥测系统主页）

**功能**：遥测包分发中心，显示各类遥测包的接收计数。

**表格列说明**：

| 列名 | 含义 |
|------|------|
| Page | 遥测页面描述 |
| Packet Count | 该类遥测包接收计数 |

**可用遥测页面一览**：

| 页面 | 对应数据 | 消息 ID |
|------|---------|---------|
| Event Messages | 事件消息 | 0x808 |
| ES HK Tlm | ES 健康状态 | 0x800 |
| EVS HK Tlm | EVS 健康状态 | 0x801 |
| SB HK Tlm | SB 健康状态 | 0x803 |
| TBL HK Tlm | TBL 健康状态 | 0x804 |
| TIME HK Tlm | TIME 健康状态 | 0x805 |
| TIME DIAG Tlm | TIME 诊断 | 0x806 |
| SB STATs Tlm | SB 统计 | 0x80A |
| ES APP Tlm | ES 应用注册表 | 0x80B |
| TBL REG Tlm | TBL 注册表 | 0x80C |
| SB ALLSUBs Tlm | SB 全部订阅 | 0x80D |
| SB OneSub Tlm | SB 单个订阅 | 0x80E |
| ES Shell Tlm | ES Shell 输出 | 0x80F |
| ES MEMSTATS Tlm | ES 内存统计 | 0x810 |
| Sample App HK Tlm | Sample App 健康状态 | 0x883 |

### 2.6 Generic Telemetry Page（遥测数据页）

**功能**：以表格形式实时显示某个遥测包的解析结果。

**表格列说明**：

| 列 | 含义 |
|----|------|
| Telemetry Point Label | 数据字段名称 |
| Current Value | 当前值 |

---

## 三、关键遥测数据详解

### 3.1 ES HK Tlm（Executive Services 健康状态）

| 字段 | 含义 | 安全意义 |
|------|------|---------|
| Command Counter | ES 收到的指令数 | 异常高 → 可能遭受指令洪泛 |
| Error Counter | ES 错误次数 | >0 → 系统异常 |
| Registered Core Apps | 已注册核心应用 | 异常减少 → 关键服务崩溃 |
| Registered CFS Apps | 已注册任务应用 | 异常增加 → 恶意应用注入 |
| Registered Tasks | 已注册后台任务 | 监控系统负载 |
| Registered Libs | 已注册共享库 | 异常增加 → 恶意库注入 |
| Reset Type | 复位类型 | 监控复位事件 |
| Processor Resets | 处理器复位次数 | 高频 → 硬件或攻击 |
| Heap Bytes Free | 堆空闲字节 | 下降 → 内存泄漏 |
| cFE/OSAL/PSP Version | 各组件版本号 | 版本审计 |
| Syslog Entries | 系统日志条数 | 监控日志使用 |

### 3.2 SB HK Tlm（Software Bus 健康状态）

| 字段 | 安全意义 |
|------|---------|
| NoSubscribersCounter | 消息发送但无订阅者 → 可能的消息注入攻击 |
| MsgSendErrorCounter | 消息发送失败 → 总线过载或拒绝服务 |
| MsgReceiveErrorCounter | 消息接收失败 |
| PipeOverflowErrorCounter | 管道溢出 → 订阅者处理不及，可能 DoS |
| MsgLimitErrorCounter | 消息大小超限 → 可能缓冲区溢出尝试 |
| CreatePipeErrorCounter | 管道创建失败 |
| DuplicateSubscriptionsCounter | 重复订阅 → 可能的重放攻击 |

---

## 四、功能测试项清单

> 📝 逐项测试，记录实际结果

| 编号 | 测试项 | 操作 | 预期结果 | 实测结果 |
|------|--------|------|---------|---------|
| T01 | 订阅表修复后遥测显示 | Enable Tlm → Start Telemetry | Packet Count > 0 | ✅ 通过 |
| T02 | 航天器自动检测 | Enable Tlm 后观察 Main Window | 下拉框出现 127.0.0.1 | |
| T03 | ES NOOP 指令 | 指令系统 → ES → NOOP | cFS 终端显示 No-op 日志 | |
| T04 | ES NOOP 后计数器变化 | 发 NOOP → 看 ES HK Tlm | Command Counter +1 | |
| T05 | SB NOOP 指令 | 指令系统 → SB → NOOP | cFS 终端显示 SB No-op | |
| T06 | EVS NOOP 指令 | 指令系统 → EVS → NOOP | cFS 终端显示 EVS No-op | |
| T07 | TBL NOOP 指令 | 指令系统 → TBL → NOOP | cFS 终端显示 TBL No-op | |
| T08 | TIME NOOP 指令 | 指令系统 → TIME → NOOP | cFS 终端显示 Time No-op | |
| T09 | CI_LAB NOOP 指令 | 指令系统 → CI → NOOP | cFS 终端显示 CI No-op | |
| T10 | TO_LAB NOOP 指令 | 指令系统 → TO → NOOP | cFS 终端显示 TO No-op | |
| T11 | Sample App NOOP | 指令系统 → Sample App → NOOP | cFS 终端显示 Sample No-op | |
| T12 | ES HK Tlm 数据验证 | 双击 ES HK Tlm 页面 | 各项有实际数值 | |
| T13 | EVS HK Tlm 数据验证 | 双击 EVS HK Tlm 页面 | 各项有实际数值 | |
| T14 | SB HK Tlm 数据验证 | 双击 SB HK Tlm 页面 | 各项有实际数值 | |
| T15 | TBL HK Tlm 数据验证 | 双击 TBL HK Tlm 页面 | 各项有实际数值 | |
| T16 | TIME HK Tlm 数据验证 | 双击 TIME HK Tlm 页面 | 各项有实际数值 | |
| T17 | ES APP Tlm 数据验证 | 双击 ES APP Tlm 页面 | 列出已注册应用 | |
| T18 | ES MEMSTATS Tlm | 双击 ES MEMSTATS Tlm | 内存统计有数据 | |
| T19 | SB STATS Tlm | 双击 SB STATS Tlm | SB 统计数据 | |
| T20 | Reset Counters 指令 | 指令系统 → ES → Reset Counters | ES 计数器归零 | |
| T21 | 遥测包持续更新 | 观察任意遥测页 30 秒 | Sequence Count 递增 | |

---

## 五、数据解析原理

GroundSystem 如何解析遥测包？以 ES HK Tlm 为例：

```
遥测定义文件: tlmGUI/cfe-es-hk-tlm.txt
格式: 描述, 字节偏移, 字节长度, Python类型, 显示方式, 枚举值...

示例行:
Command Counter,  12,  1,  B, Dec, ...
                 ↑    ↑   ↑   ↑
                 │    │   │   └─ 十进制显示
                 │    │   └─── unsigned byte (1字节)
                 │    └─────── 从包的第12字节开始
                 └────────── 字段名
```

GroundSystem 收到 CCSDS 遥测包后，按这个定义表逐个字段用 Python `struct.unpack` 解析并显示。

---

你先按这份报告框架逐项测试，把 **T01-T21** 的实测结果填上，遇到任何不明白的字段或行为，随时问我。测试完后我帮你进一步细化报告内容。