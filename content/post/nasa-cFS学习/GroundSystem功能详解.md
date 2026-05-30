---
categories: ["nasa-cFS学习"]
tags: ["Linux", "nasa-cFS", "OS"]
title: "GroundSystem功能详解"
date: 2026-05-30
draft: false
---

# cFS GroundSystem 功能测试报告

## 〇、前置操作：TO_LAB 订阅表修复

> ⚠️ **重要**：cFS 开源 Bundle 默认配置下，TO_LAB（遥测输出应用）的订阅表 `to_lab_sub.c` 中所有订阅条目均被注释。这导致即使发送 Enable Tlm 指令，地面站也无法收到任何遥测数据。此问题是本实验的第一个关键发现。

**修复方法**：编辑 `apps/to_lab/fsw/tables/to_lab_sub.c`，取消 `/* cFE Core subscriptions (examples) */` 下方所有条目的 `/*` `*/` 注释符，然后执行：

```bash
cd ~/File/cFS && make native_std.install
```

重启 `./core-cpu1` 后，GroundSystem 才可正常接收遥测数据。

---

## 一、系统功能总览

### 1.1 通信协议基础：CCSDS 空间包协议

GroundSystem 与 cFS 之间的所有通信均基于 **CCSDS（空间数据系统咨询委员会，Consultative Committee for Space Data Systems）** 制定的国际标准。cFS 本身是 CCSDS 协议族在飞行软件领域的一套开源参考实现。

#### CCSDS 的地位

| 对比维度 | 互联网领域 | 航天领域 |
|---------|-----------|---------|
| 标准制定组织 | IETF（互联网工程任务组） | **CCSDS** |
| 成员 | 全球网络工程师社群 | NASA、ESA、JAXA、CNSA（中国国家航天局）等 |
| 成立时间 | 1986 | **1982** |
| 典型协议 | TCP、UDP、IP、HTTP | CCSDS 空间包协议、遥控协议、遥测协议、CFDP |
| 法律地位 | 行业事实标准 | **ISO 国际标准** |

#### CCSDS 与 cFS 的对应关系

cFS 各组件是对 CCSDS 国际标准的具体实现：

| CCSDS 标准 | 蓝皮书编号 | cFS 中的实现 |
|-----------|----------|------------|
| 空间包协议（Space Packet Protocol） | CCSDS 133.0-B-2 | `cfe/modules/msg/` — 消息格式定义（Stream ID + Sequence + Length） |
| 遥控空间数据链路协议（TC） | CCSDS 232.0-B | `cmdUtil.c` — 上行指令包组装 |
| 遥测空间数据链路协议（TM） | CCSDS 132.0-B | TO_LAB — 下行遥测包打包 |
| CFDP 文件传输协议 | CCSDS 727.0-B | `apps/cf/` — 可靠文件传输 |
| 电子数据单（EDS） | CCSDS 876.0-B | `tools/eds/` — 接口 XML 定义与代码生成 |

#### CCSDS 空间包基本结构

cFS 中每条消息（指令或遥测）都封装为 CCSDS 空间包：

```
CCSDS Version 1 空间包（6 字节主头）:
┌────────────┬────────────┬────────────┬──────────────────┐
│ Stream ID  │  Sequence  │   Length   │   Packet Body    │
│  (2 字节)   │  (2 字节)   │  (2 字节)   │  (变长载荷数据)     │
└────────────┴────────────┴────────────┴──────────────────┘
  ├─ Ver(3b) ┤             │
  ├─ Type(1b)┤             │
  ├─ SecHdr(1b)┤            │
  └─ APID(11b)┘             │
                ├─SeqFlag(2b)┤
                └─SeqCnt(14b)┘

（位字段定义来自 CCSDS 133.0-B-2，在 cmdUtil.c 注释中有完整记录）
```

> 正因 cFS 完全遵循 CCSDS 国际标准，理论上任何符合 CCSDS 协议的地面站均可与 cFS 卫星互操作——不限于 NASA 自研的 GroundSystem。UDP/IP 在此仅作为底层传输承载，CCSDS 包被嵌入 UDP 数据报 payload 中传输。

### 1.2 GroundSystem 是什么？

GroundSystem 是 cFS 的**地面站模拟软件**（Python + PyQt5 编写），用于在开发/测试阶段模拟地面指控中心。通过 UDP/IP 协议与 cFS 核心通信：指令走 UDP 端口 1234，遥测走 UDP 端口 2234。

### 1.3 与实际航天系统的对应关系

| 实际航天元素 | GroundSystem 中的对应 | 说明 |
|-------------|----------------------|------|
| **通信协议标准** | CCSDS 空间包协议（133.0-B-2） | cFS 整个通信体系的基础协议 |
| **地面指控中心** | `GroundSystem.py` 主窗口 | 控制上行指令、监控下行遥测 |
| **上行链路 (Uplink)** | cmdUtil → UDP 1234 → CI_LAB | 指令从地面发往卫星 |
| **下行链路 (Downlink)** | TO_LAB → UDP 2234 → RoutingService | 遥测从卫星发往地面 |
| **指令编码器** | `cmdUtil.c`（C 程序） | 将指令参数组装为 CCSDS 标准包 |
| **遥测解码器** | `GenericTelemetry.py` | 按字段定义解析 CCSDS 遥测包 |
| **卫星上的指令分发** | SB 软件总线（`cfe/modules/sb/`） | 发布/订阅模式路由消息 |
| **卫星上的遥测收集** | SB + TO_LAB | 各应用发布遥测 → TO_LAB 订阅并转发 |
| **航天器标识方式** | GUI 下拉框显示源 IP 地址 | 自动检测活跃航天器的 IP，支持多星场景 |

> 📝 **注意**：RoutingService 在终端日志中会打印 "Detected Spacecraft1 at 127.0.0.1"，其中 "Spacecraft1" 是内部 ZMQ 路由名，仅用于构建 ZMQ 发布频道（如 `GroundSystem.Spacecraft1.TelemetryPackets.0x800`），**不会出现在 GUI 界面上**。用户看到的仅为 IP 地址。

### 1.4 数据流架构

```
┌──────────────────────────────────────────────────────────────┐
│                    GroundSystem (地面站)                      │
│                                                              │
│  ┌──────────┐  指令   ┌──────────┐  UDP:1234  ┌──────────┐  │
│  │ 指令 GUI │ ──────→ │ cmdUtil  │ ────────→  │ CI_LAB   │  │
│  │ (PyQt5)  │         │  (C程序)  │           │ (指令注入) │  │
│  └──────────┘         └──────────┘           └────┬─────┘  │
│                                                    │ SB     │
│  ┌──────────┐  遥测   ┌────────────┐  UDP:2234     │ 软件总线│
│  │ 遥测 GUI │ ←────── │RoutingSvc  │ ←─────────── │        │  │
│  │ (PyQt5)  │   ZMQ   │ (Python)   │              └──┬─────┘  │
│  └──────────┘         └────────────┘             ┌──┴──────┐ │
│                                                   │ TO_LAB  │ │
│                                                   │(遥测输出)│ │
│                                                   └────────┘ │
└──────────────────────────────────────────────────────────────┘
```

**关键通信端口**：

| 端口 | 方向 | 承载协议 | 应用层协议 | 负责模块 |
|------|------|---------|----------|---------|
| UDP 1234 | 地面 → 星上 | UDP/IP | **CCSDS 空间包** | cmdUtil → CI_LAB → SB |
| UDP 2234 | 星上 → 地面 | UDP/IP | **CCSDS 空间包** | TO_LAB → RoutingService |
| ZMQ IPC | 地面内部 | ZeroMQ PUB/SUB | 原始字节 | RoutingService → 遥测 GUI |

---

## 二、窗口逐一说明

### 2.1 Main Window（主窗口）

**功能**：地面站控制中枢，启动/停止各子系统，自动检测活跃航天器。

| 控件 | 功能 |
|------|------|
| **Start Command System** | 打开指令系统窗口（上行链路 GUI） |
| **Start Telemetry System** | 打开遥测系统窗口（下行链路 GUI） |
| **Selected IP Address** | 航天器 IP 下拉框——RoutingService 收到遥测包后自动填充 |
| **TLM Header Version** | 遥测包头版本：`1` / `2` / `Custom`（对应 CCSDS Version 1 和 Version 2 包格式，V2 比 V1 多了 4 字节扩展头） |
| **CMD Header Version** | 指令包头版本：`1` / `2` / `Custom`（同上，控制组包时在 CCSDS 包头后额外跳过的字节偏移量） |

> 📝 **实测现象**：发送 Enable Tlm 前下拉框只有 "All"，发送后自动出现 `127.0.0.1`。注：终端日志中虽然打印 "Detected Spacecraft1 at 127.0.0.1"，但 GUI 中仅显示 IP 地址，不显示内部路由名。

### 2.2 Command System Main Page（指令系统主页）

**功能**：cFS 各子系统的指令入口。表格列出所有可用的子系统及其消息 ID。

**表格列说明**：

| 列名 | 含义 | 示例 |
|------|------|------|
| Subsystem/Page | 子系统名称 | Executive Services |
| Packet ID | CCSDS 消息 ID（十六进制） | 0x1806 |
| Send To | 目标 IP 地址 | 127.0.0.1 |

**按钮说明**：

| 按钮 | 功能 |
|------|------|
| **Display Page** | 打开该子系统的完整指令列表 |
| **Enable Tlm**（TO_LAB 行） | 快捷按钮——开启遥测下行，需输入 dest_IP |
| **各 No-Op 按钮**（ES/SB/TBL/TIME/EVS/CI 行） | 快捷按钮——直接发送空操作指令 |

**子系统列表与对应模块**：

| 子系统 | Packet ID | cFS 源码位置 | 实际功能 |
|--------|-----------|-------------|---------|
| Executive Services | 0x1806 | `cfe/modules/es/` | 应用生命周期管理、系统状态监控 |
| Software Bus | 0x1803 | `cfe/modules/sb/` | 消息发布/订阅中枢 |
| Table Services | 0x1804 | `cfe/modules/tbl/` | 配置表加载/校验/管理 |
| Time Services | 0x1805 | `cfe/modules/time/` | 系统时钟同步 |
| Event Services | 0x1801 | `cfe/modules/evs/` | 事件日志与分发 |
| Command Ingest | 0x1884 | `apps/ci_lab/` | UDP 指令接收与注入（Lab） |
| Telemetry Output | 0x1880 | `apps/to_lab/` | 遥测收集与 UDP 转发（Lab） |
| Sample App | 0x1882 | `apps/sample_app/` | 示例应用模板 |

### 2.3 Display Page（指令详情页）

**功能**：显示某个子系统的所有可用指令。点击 Command System 表格中的「Display Page」按钮打开。指令以底层 C 宏名形式展示，点击「Send」按钮发送。

> 📝 **命名说明**：GUI 中指令名称为 C 语言宏形式（如 `CFE_ES_NOOP_CC`），功能码（如 `0x00`）不显示在界面上。为阅读方便，下文以易读名称备注。

以 **Executive Services** 的 Display Page 为例，实测 ES 共约 25 条指令，分为两类：

**A 类：无参数指令（点击 Send 立即执行，无需额外输入）——共 5 条：**

| GUI 显示名称 | 易读名称 | 说明 |
|-------------|---------|------|
| `CFE_ES_NOOP_CC` | Noop | 空操作，测试连通性 |
| `CFE_ES_RESET_COUNTERS_CC` | Reset Counters | 重置 ES 内部计数器 |
| `CFE_ES_CLEAR_SYSLOG_CC` | Clear Syslog | 清空系统日志 |
| `CFE_ES_CLEAR_ER_LOG_CC` | Clear ER Log | 清空事件日志 |
| `CFE_ES_RESET_PR_COUNT_CC` | Reset PR Count | 重置处理器复位计数 |

**B 类：有参数指令（点击 Send 后弹出 Parameter Dialog，需填入参数）——其余全部指令：**

| GUI 显示名称 | 易读名称 | 推测功能 | 参数待验证 |
|-------------|---------|---------|:---:|
| `CFE_ES_START_APP_CC` | Start App | 启动新应用 | ⚠️ |
| `CFE_ES_STOP_APP_CC` | Stop App | 停止指定应用 | ⚠️ |
| `CFE_ES_RESTART_APP_CC` | Restart App | 重启指定应用 | ⚠️ |
| `CFE_ES_RELOAD_APP_CC` | Reload App | 重新加载应用 | ⚠️ |
| `CFE_ES_DELETE_CDS_CC` | Delete CDS | 删除关键数据存储 | ⚠️ |
| `CFE_ES_QUERY_ALL_CC` | Query All | 查询所有应用 | ⚠️ |
| `CFE_ES_QUERY_ALL_TASKS_CC` | Query All Tasks | 查询所有任务 | ⚠️ |
| `CFE_ES_QUERY_ONE_CC` | Query One | 查询单个应用 | ⚠️ |
| `CFE_ES_SHELL_CC` | Shell | 执行 Shell 命令 | ⚠️ |
| `CFE_ES_SEND_MEM_POOL_STATS_CC` | Send Mem Pool Stats | 发送内存池统计 | ⚠️ |
| `CFE_ES_WRITE_SYSLOG_CC` | Write Syslog | 写入系统日志 | ⚠️ |
| `CFE_ES_OVER_WRITE_SYSLOG_CC` | Overwrite Syslog | 覆写系统日志 | ⚠️ |
| `CFE_ES_WRITE_ER_LOG_CC` | Write ER Log | 写入事件日志 | ⚠️ |
| `CFE_ES_DUMP_CDS_REGISTRY_CC` | Dump CDS Registry | 转储 CDS 注册表 | ⚠️ |
| `CFE_ES_RESTART_CC` | Restart | 处理器复位 | ⚠️ |
| `CFE_ES_SET_MAX_PR_COUNT_CC` | Set Max PR Count | 设置最大复位次数 | ⚠️ |
| `CFE_ES_START_PERF_DATA_CC` | Start Perf Data | 启动性能数据采集 | ⚠️ |
| `CFE_ES_STOP_PERF_DATA_CC` | Stop Perf Data | 停止性能数据采集 | ⚠️ |
| `CFE_ES_SET_PERF_FILTER_MASK_CC` | Set Perf Filter | 设置性能过滤器 | ⚠️ |
| `CFE_ES_SET_PERF_TRIGGER_MASK_CC` | Set Perf Trigger | 设置性能触发器 | ⚠️ |

> 📝 **待测试**：B 类每条的参数名、数据类型、功能含义均需逐条在 GUI 上点击 Send 后记录 Parameter Dialog 中显示的参数信息来验证。功能码由系统内部维护，不在 GUI 上展示。

### 2.4 Parameter Dialog（参数对话框）

**功能**：为需要参数的命令提供参数输入界面。点击「Send」后若该指令在 `ParameterFiles/` 中有对应的参数定义文件，则弹出此对话框。

**表格列说明**：

| 列名 | 含义 | 是否可编辑 |
|------|------|:---:|
| Parameter | 参数名（C 结构体字段名） | 否（只读） |
| Description | 参数说明文字（来自 C 头文件注释，经常为空） | 否（只读） |
| Input | 用户实际输入参数值的位置 | **是** |

**示例——Enable Tlm 指令参数**：

| 字段 | 含义 | 输入示例 |
|------|------|---------|
| dest_IP | 遥测数据发送到的目标 IP 地址（字符串类型） | `127.0.0.1` |

> 📝 **已知问题**：Enable Tlm 快捷按钮在特定条件下存在异常，但目前暂未复现并继续研究。

### 2.5 Telemetry System Main Page（遥测系统主页）

**功能**：遥测包分发中心，显示各类遥测包的实时接收计数。

**表格列说明**：

| 列名 | 含义 |
|------|------|
| Subsystem/Page | 遥测页面描述 |
| Packet ID | CCSDS 消息 ID |
| Display Page | 按钮——点击打开该遥测包的数据解析界面 |
| Packet Count | 该类遥测包累计接收数量 |

**可用遥测页面一览**：

| 页面 | 消息 ID | 实测 Packet Count > 0？ |
|------|---------|----------------------|
| Event Messages | 0x808 | ✅（事件产生后即有数据） |
| ES HK Tlm | 0x800 | ✅ |
| EVS HK Tlm | 0x801 | ✅ |
| SB HK Tlm | 0x803 | ✅ |
| TBL HK Tlm | 0x804 | ✅ |
| TIME HK Tlm | 0x805 | ✅ |
| TIME DIAG Tlm 1 | 0x806 | ❌（需主动请求） |
| TIME DIAG Tlm 2 | 0x806 | ❌（需主动请求） |
| SB STATs Tlm | 0x80A | ❌（需主动请求） |
| SB PipeDepthStats Tlm 1 | 0x80A | ❌ |
| SB PipeDepthStats Tlm 2 | 0x80A | ❌ |
| ES APP Tlm | 0x80B | ❌（需主动请求） |
| TBL REG Tlm | 0x80C | ❌（需主动请求） |
| SB ALLSUBs Tlm | 0x80D | ❌（需主动请求） |
| SB OneSub Tlm | 0x80E | ❌（需主动请求） |
| ES Shell Tlm | 0x80F | ❌（需主动请求） |
| ES MEMSTATS Tlm | 0x810 | ❌（需主动请求） |
| ES BlockStats Tlm 1 | 0x810 | ❌ |
| Sample App HK Tlm | 0x883 | ✅ |

> 📝 **测试条件**：以上数据为**仅发送 Enable Tlm（`dest_IP = 127.0.0.1`）后的基线状态**，未额外发送其他指令。
>
> **实测小结**：20 个遥测页面中 **7 个**有持续数据——5 个核心 `HK Tlm`（ES/EVS/SB/TBL/TIME，约 1Hz 周期性发布）+ Sample App HK Tlm + Event Messages。Event Messages 在系统初始化阶段可能为 0，一旦有应用产生事件（如 SC 的 RTS 执行、NOOP 指令响应），即开始收到数据。实测 Observed Event Messages 内容示例：
> ```
> EVENT --> SC-INFORMATION Event ID: 52 : No-op command. Version 7.0.1.255
> EVENT --> SC-INFORMATION Event ID: 86 : RTS 002 Execution Completed
> ```
> 其余 `STATS`/`DIAG`/`APP`/`REG`/`Shell`/`MEMSTATS` 类需通过对应子系统的 Display Page 发送指令主动触发一次性下发。

> ⚠️ **Packet Count 的行为特性**：
>
> 1. **计数不跨进程**：Telemetry System Main Page 中的 Packet Count 是当前进程内存变量。**关闭并重新打开 Telemetry System 窗口后，所有计数归零**。
> 2. **Display Page 只收新包**：点击「Display Page」会启动一个全新的独立 Python 进程（`subprocess.Popen`），该进程从这一刻起建立 ZMQ 订阅。**打开前已产生的遥测包不会显示**，仅显示打开后新收到的包。因此可能出现「Packet Count = 7，但 Display Page 中为空」的现象——7 个包是之前攒的，Display Page 打开后还没收到新的。
> 3. **周期性遥测刷新**：`HK Tlm` 类约 1Hz 周期性发布，打开 Display Page 后等待 1~2 秒即可看到数据刷新。

### 2.6 遥测数据展示页（点击 Display Page 弹出）

**功能**：以表格形式实时显示某个遥测包各字段的当前值。

**表格列说明**：

| 列 | 含义 |
|----|------|
| Telemetry Point Label | 数据字段名称 |
| Telemetry Point Value | 当前值 |

数据解析方式：读取对应的 `.txt` 定义文件（如 `cfe-es-hk-tlm.txt`），按"字节偏移 + 字节长度 + Python struct 类型"逐字段解析 CCSDS 包体。

---

## 三、关键遥测数据字段详解

> ⚠️ **通用提示**：以下字段表基于 GroundSystem 自带的 `.txt` 定义文件整理。由于定义文件版本可能与实际 cFS 7.0.1 固件发出的包大小不完全匹配，部分字段可能因偏移量超出包长而不显示（如 3.1 中 ES HK Tlm 仅显示 36 个中的 6 个）。**以 GUI 实际显示为准**，其余字段待逐一核实。

### 3.1 ES HK Tlm — Executive Services 健康状态（0x800）

> ⚠️ 定义文件 `cfe-es-hk-tlm.txt` 包含 36 个字段（最高偏移量 152 字节），但实测 ES HK Tlm 的 Display Page **仅显示 6 个字段**。原因是当前 cFS 7.0.1 实际发出的 ES HK 遥测包体较短（约 64 字节），定义文件中偏移量超出实际包长的 30 个字段被 `GenericTelemetry.py` 自动跳过不显示。这是 GroundSystem 的遥测定义文件与实际固件版本不匹配导致的，**非操作错误**。

**实测显示的 6 个字段**：

| 字段 | 偏移 | 类型 | 含义 | 安全审计意义 |
|------|------|------|------|------------|
| Command Counter | 12 | uint8 | ES 收到的指令数 | 异常升高 → 指令洪泛攻击 |
| Error Counter | 13 | uint8 | ES 错误次数 | >0 → 系统运行异常 |
| Registered Core Apps | 48 | uint32 | 核心应用注册数 | 异常减少 → 关键服务崩溃 |
| Registered CFS Apps | 52 | uint32 | 任务应用注册数 | 异常增加 → 恶意应用注入 |
| Registered Tasks | 56 | uint32 | 后台任务注册数 | 系统负载指标 |
| Registered Libs | 60 | uint32 | 共享库注册数 | 异常增加 → 恶意库注入 |

> 📝 **定义文件中未显示的字段**（偏移量超出实际包长）：所有版本号字段（cFE/OSAL/PSP）、系统日志统计、复位信息、性能监控、堆内存统计等 30 个字段均不显示。如需查看这些信息，可能需要在 cFS 编译时调整 ES HK 包的大小配置，或使用更新版本的 GroundSystem 遥测定义文件。 |

### 3.2 SB HK Tlm — Software Bus 健康状态（0x803）

| 字段 | 偏移 | 类型 | 安全审计意义 |
|------|------|------|------------|
| Command Counter | 12 | uint8 | SB 收到的指令数 |
| Error Counter | 13 | uint8 | SB 错误数 |
| **NoSubscribersCounter** | 14 | uint8 | 消息无订阅者 → 可能的消息注入/探测攻击 |
| **MsgSendErrorCounter** | 15 | uint8 | 发送失败 → 总线过载或 DoS |
| MsgReceiveErrorCounter | 16 | uint8 | 接收失败 |
| InternalErrorCounter | 17 | uint8 | 内部错误 |
| CreatePipeErrorCounter | 18 | uint8 | 管道创建失败 |
| SubscribeErrorCounter | 19 | uint8 | 订阅失败 |
| PipeOptsErrorCounter | 20 | uint8 | 管道选项错误 |
| **DuplicateSubscriptionsCounter** | 21 | uint8 | 重复订阅 → 可能的重放攻击 |
| GetPipeIdByNameErrorCounter | 22 | uint8 | 管道查找失败 |
| Spare2Align | 23 | uint8 | 对齐填充字节（无功能含义） |
| **PipeOverflowErrorCounter** | 24 | uint16 | 管道溢出 → 订阅者处理不及，潜在 DoS |
| **MsgLimitErrorCounter** | 36 | uint16 | 消息大小超限 → 可能缓冲区溢出尝试 |
| MemPoolHandle | 28 | uint32 | 内存池句柄 |
| MemInUse | 32 | uint32 | SB 内存使用量 |
| UnmarkedMem | 36 | uint32 | 未标记内存（潜在内存泄漏） |

### 3.3 TBL HK Tlm — Table Services 健康状态（0x804）

| 字段 | 偏移 | 类型 | 说明 |
|------|------|------|------|
| Command Counter | 12 | uint8 | 表服务收到的指令数 |
| Error Counter | 13 | uint8 | 表服务错误次数 |
| Num tables | 14 | uint16 | 已注册表数量 |
| Num load pending | 16 | uint16 | 待加载表数量 |
| Validation cnt | 18 | uint16 | 校验计数 |
| Last valid CRC | 20 | uint32 | 最近有效 CRC 值 |
| Last valid status | 24 | uint32 | 最近有效状态码 |
| Active buffer | 28 | uint8 | 当前活动缓冲区编号 |
| Last valid tbl | 29 | string(40) | 最近有效表名（空 → 空白框 □） |
| Success count | 69 | uint8 | 表加载成功次数 |
| Failed count | 70 | uint8 | 表加载失败次数 |
| Num requests | 71 | uint8 | 表操作请求数 |
| Num free bufs | 72 | uint8 | 空闲缓冲区数 |
| pad1 | 73 | uint8 | 对齐填充字节（无功能） |
| pad2 | 74 | uint16 | 对齐填充字节（无功能） |
| Mem pool hdl | 76 | uint32 | 内存池句柄（Hex 显示） |
| Last upd (secs) | 80 | uint32 | 最近更新 — 秒部分 |
| Last upd (subs) | 84 | uint32 | 最近更新 — 亚秒部分 |
| Last upd table name | 88 | string(40) | 最近更新表名（空 → 空白框 □） |
| Last file loaded | 128 | string(64) | 最近加载的文件路径（空 → 空白框 □） |
| Last file dumped | 192 | string(64) | 最近转储的文件路径（空 → 空白框 □） |
| LastTableLoaded | 256 | string(40) | 最近加载的表名（空 → 空白框 □） |

> 📝 **实测现象**：字符串类型字段在未初始化时显示为空白方框（`□`），这是 C 语言空字符 `\0` 在 Qt 表格中的正常渲染结果，不影响功能。`pad1`/`pad2` 为 C 结构体对齐填充字节，无实际功能含义。

### 3.4 TIME HK Tlm — Time Services 健康状态（0x805）

| 字段 | 说明 |
|------|------|
| Command/Error Counter | 时间服务指令/错误计数 |
| ClockStateFlags | 时钟状态标志位 |
| ClockStateAPI | 时钟状态 API 值 |
| LeapSeconds | 闰秒数 |
| SecondsMET / SubsecsMET | 任务运行时间（Mission Elapsed Time） |
| SecondsSTCF / SubsecsSTCF | 航天器时间（Spacecraft Time Correlation Factor） |

---

## 四、功能测试结果

| 编号 | 测试项 | 操作路径 | 预期结果 | 实测结果 | 说明 |
|------|--------|---------|---------|---------|------|
| T01 | 订阅表修复后遥测 | Enable Tlm → Start Telemetry | Packet Count > 0 | ✅ 通过 | 所有已订阅遥测类型正常接收 |
| T02 | 航天器自动检测 | Enable Tlm 后观察 Main Window | 下拉框出现 127.0.0.1 | ✅ 通过 | RoutingService 检测到遥测包后自动添加 |
| T03 | ES NOOP | 指令系统 → ES → NOOP | 终端显示 No-op 日志 | ✅ 通过 | `EVS Port1 ... CFE_ES 3: No-op command: cFS Versions: cfe v7.0.1+dev1...` |
| T04 | ES NOOP 后计数器 | 发 NOOP → 观察 ES HK Tlm | Command Counter +1 | ✅ 通过 | |
| T05 | SB NOOP | 指令系统 → SB → NOOP | 终端显示 SB No-op | ✅ 通过 | `CFE_SB 28: No-op Cmd Rcvd: CFE_SB v7.0.1+dev1` |
| T06 | EVS NOOP | 指令系统 → EVS → NOOP | 终端显示 EVS No-op | ✅ 通过 | `CFE_EVS 0: No-op Cmd Rcvd: CFE_EVS v7.0.1+dev1` |
| T07 | TBL NOOP | 指令系统 → TBL → NOOP | 终端显示 TBL No-op | ✅ 通过 | `CFE_TBL 10: No-op Cmd Rcvd: CFE_TBL v7.0.1+dev1` |
| T08 | TIME NOOP | 指令系统 → TIME → NOOP | 终端显示 Time No-op | ✅ 通过 | `CFE_TIME 4: No-op Cmd Rcvd: CFE_TIME v7.0.1+dev1` |
| T09 | CI_LAB NOOP | 指令系统 → CI → NOOP | 终端显示 CI No-op | ✅ 通过 | `CI_LAB 5: CI: NOOP command. Version 7.0.1.255` |
| T10 | TO_LAB NOOP | 指令系统 → TO → NOOP | 终端显示 TO No-op | ✅ 通过 | `TO_LAB 16: TO: NOOP command. TO Lab v7.0.1+dev1` |
| T11 | Sample App NOOP | 指令系统 → Sample App → NOOP | 终端显示 Sample No-op | ✅ 通过 | `SAMPLE_APP 3: SAMPLE: NOOP command v7.0.0+dev1` |
| T12 | ES HK Tlm 数据 | 遥测系统 → ES HK → Display Page | 各字段有实际数值 | ✅ 通过 | 实测显示 6 个字段（定义文件含 36 个，其余因包体截断不显示） |
| T13 | EVS HK Tlm 数据 | 遥测系统 → EVS HK → Display Page | 各字段有实际数值 | ✅ 通过 | |
| T14 | SB HK Tlm 数据 | 遥测系统 → SB HK → Display Page | 各字段有实际数值 | ✅ 通过 | |
| T15 | TBL HK Tlm 数据 | 遥测系统 → TBL HK → Display Page | 各字段有实际数值 | ✅ 通过 | 字符串字段显示空白框（未加载表） |
| T16 | TIME HK Tlm 数据 | 遥测系统 → TIME HK → Display Page | 各字段有实际数值 | ✅ 通过 | |
| T17 | ES APP Tlm | 遥测系统 → ES APP → Display Page | Packet Count > 0 | — 未测 | 需向 ES 发送查询指令触发，待后续测试 |
| T18 | ES MEMSTATS Tlm | 遥测系统 → ES MEMSTATS → Display Page | Packet Count > 0 | — 未测 | 需向 ES 发送统计指令触发，待后续测试 |
| T19 | SB STATS Tlm | 遥测系统 → SB STATS → Display Page | Packet Count > 0 | — 未测 | 需向 SB 发送统计指令触发，待后续测试 |
| T20 | ES Reset Counters | 指令系统 → ES → Reset Counters | ES Command/Error Counter 归零 | ✅ 通过 | 终端：`CFE_ES 4: Reset Counters command`。仅重置 ES 内部计数器，不影响 GroundSystem 显示或 CCSDS Sequence Count |
| T21 | 遥测持续更新 | 观察遥测页 30 秒 | Sequence Count 递增 | ✅ 通过 | 周期性遥测正常运行 |

---

## 五、数据解析原理

GroundSystem 收到 CCSDS 遥测原始字节流后，通过以下流程解析：

```
CCSDS 包（原始字节）
    │
    ▼
RoutingService.py ─── 读取前 2 字节获取 Stream ID
    │                  ZMQ PUB: GroundSystem.{host}.Telemetry.{StreamID}
    ▼
TelemetrySystem.py ─── 匹配 telemetry-pages.txt 中的 Packet ID
    │                  找到对应解析类（GenericTelemetry.py）
    ▼
GenericTelemetry.py ─── 读取对应的 .txt 定义文件
    │                  按行解析 CSV：字段名, 偏移, 长度, 类型, 显示方式
    ▼
struct.unpack(fmt, data[offset:offset+length])
    │
    ▼
QTableWidget 显示
```

**关键文件对应关系**：

| 文件 | 作用 |
|------|------|
| `telemetry-pages.txt` | 遥测页面注册表：描述 → Python 类 → Packet ID → 定义文件 |
| `cfe-es-hk-tlm.txt` | ES 健康遥测字段定义（36 个字段） |
| `cfe-sb-hk-tlm.txt` | SB 健康遥测字段定义（17 个字段） |
| `cfe-tbl-hk-tlm.txt` | TBL 健康遥测字段定义（22 个字段） |
| `cfe-time-hk-tlm.txt` | TIME 健康遥测字段定义（8 个字段） |
| `command-pages.txt` | 指令页面注册表 |
| `quick-buttons.txt` | 快捷按钮定义表 |
| `GenericTelemetry.py` | 通用遥测解析引擎 |
| `cmdUtil.c` | CCSDS 指令组包与 UDP 发送 |
| `MiniCmdUtil.py` | Python 版指令组包（有 bug，字符串参数处理异常） |

---

## 六、已发现的一些问题（B01暂未复现和研究，可能只是偶然）

| 编号 | 缺陷描述 | 影响 |
|------|---------|------|
| B01 | Enable Tlm 快捷按钮 `MiniCmdUtil` 异常（`ValueError`，非确定性） | 详见独立文档 `GroundSystem-BugTracker.md` |
| B02 | TO_LAB 订阅表默认全部注释，导致默认配置下遥测不工作 | 新用户必须手动修改 `to_lab_sub.c` 并重新编译 |
| B03 | TBL HK Tlm 字符串字段在空值状态下渲染为无意义空白框 | 用户体验问题，建议显示 "(empty)" 或 "N/A" |
| B04 | 遥测定义文件（`.txt`）与 cFS 7.0.1 实际固件发出的包大小不匹配（如 ES HK Tlm 定义 36 字段但实发仅 6 字段） | 部分遥测字段不显示，信息缺失。真实任务中需对齐定义文件与固件 |

---

## 七、尚未完成的内容

**Display Page 层面**：

- ES 以外子系统的 Display Page（SB、EVS、TBL、TIME、CI_LAB、TO_LAB、Sample App）尚未逐指令验证。仅确认了它们的 NOOP 指令可正常发送
- 所有 B 类有参数指令的具体参数名、数据类型、功能含义尚未测试——当前仅记录了 ES 的完整指令名列表，参数信息为空（标记 ⚠️ 待验证）

**遥测数据字段验证**：

- **EVS HK Tlm**（0x801）：尚未逐字段核对
- **TIME HK Tlm**（0x805）：尚未逐字段核对
- 所有按需遥测页面（TIME DIAG、SB STATS、ES APP、TBL REG、SB ALLSUBs、SB OneSub、ES Shell、ES MEMSTATS）尚未通过发送指令触发，Packet Count 均保持为 0