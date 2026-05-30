---
categories: ["nasa-cFS学习"]
tags: ["Linux", "nasa-cFS" , "OS" , "GroundSystem"]
title: "GroundSystem学习"
date: 2026-05-21
draft: True
---


---

## 🔍 从 GroundSystem 出发学习 cFS 日志与功能模块 — 完整指导

---

### 一、GroundSystem 整体架构

GroundSystem 是 cFS 的地面站工具，由 Python + PyQt5 编写，通过 **UDP** 与 cFS 通信，通过 **ZeroMQ** 做内部消息分发。

```
┌──────────────────────────────────────────────────┐
│              GroundSystem.py (主窗口)              │
│                                                    │
│  ┌──────────┐  ┌──────────┐  ┌─────────────────┐ │
│  │ 启动指令系统│  │ 启动遥测系统│  │ 选择航天器 IP  │ │
│  └─────┬────┘  └─────┬────┘  └────────┬────────┘ │
│        │              │                │          │
└────────┼──────────────┼────────────────┼──────────┘
         │              │                │
         ▼              ▼                ▼
┌──────────────────────────────────────────────────┐
│        RoutingService.py (ZMQ PUB/SUB 中枢)       │
│                                                    │
│  监听 UDP 2234  ←── cFS TO_LAB app 发来的遥测      │
│  发布到 ZMQ IPC: /tmp/GroundSystem-{user}          │
│  频道格式: GroundSystem.{hostname}.Telemetry.{id}  │
└──────────────────────┬───────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ TlmSystem│ │ CmdSystem│ │ FdlSystem│
   │ (遥测GUI) │ │ (指令GUI) │ │ (文件GUI) │
   └──────────┘ └──────────┘ └──────────┘
```

**关键文件清单：**

| 文件 | 功能 |
|------|------|
| GroundSystem.py | 主窗口入口，启动/停止各子系统 |
| RoutingService.py | UDP→ZMQ 桥接，消息路由中枢 |
| `TlmMQRecv.py` | ZMQ 订阅接收器 |
| `TlmUDPSender.py` | UDP 遥测发送器 |
| `Subsystems/cmdGui/CommandSystem.py` | 指令发送 GUI |
| `Subsystems/cmdGui/UdpCommands.py` | 通过 cmdUtil 发送 UDP 指令 |
| `Subsystems/tlmGUI/TelemetrySystem.py` | 遥测接收/解析 GUI |
| `Subsystems/tlmGUI/GenericTelemetry.py` | 通用遥测解析页面 |
| `Subsystems/cmdUtil/cmdUtil.c` | C 语言指令组包/发送工具 |

---

### 二、两大核心数据流

#### 2.1 指令下发流（地→星 Command Upload）

```
用户点击指令按钮
    │
    ▼
UdpCommands.py (Python GUI)
    │  根据 command-pages.txt 找到该子系统的 MsgID/Port
    │  读取 cfe-es-cmds.txt 等指令定义文件
    ▼
cmdUtil.c (C 程序)
    │  组装 CCSDS 包：
    │    ① CCSDS Primary Header   (6 bytes, Big Endian)
    │    ② CCSDS Extended Header  (4 bytes, 可选)
    │    ③ cFS Command Sec Header (2 bytes, Function Code + Checksum)
    │    ④ 应用载荷（参数数据）
    ▼
SendUdp.c → UDP socket sendto() → 127.0.0.1:1234
    │
    ▼
cFS 端: CI_LAB app 监听到 UDP 1234 → 解析 CCSDS → 发布到 SB 软件总线
    │
    ▼
目标 App（如 ES/SB/EVS）订阅对应 MsgID → 收到指令 → 执行功能码
```

**command-pages.txt 格式：**
```
描述,             指令定义文件,  MsgID,  端序,  Python类,       目标IP,    端口
Executive Services, CFE_ES_CMD,  0x1806, LE,   UdpCommands.py, 127.0.0.1, 1234
Software Bus,       CFE_SB_CMD,  0x1803, LE,   UdpCommands.py, 127.0.0.1, 1234
Event Services,     CFE_EVS_CMD,  0x1801, LE,   UdpCommands.py, 127.0.0.1, 1234
...
```

#### 2.2 遥测接收流（星→地 Telemetry Downlink）

```
cFS 端: 应用产生遥测 → 发布到 SB 软件总线
    │
    ▼
TO_LAB app 订阅遥测频道 → 组装 CCSDS 遥测包 → UDP sendto() → 127.0.0.1:2234
    │
    ▼
RoutingService.py (QThread)
    │  socket.recvfrom(4096) 监听 UDP 2234
    │  解析 CCSDS Stream ID (前2字节) → pkt_id
    │  自动识别新航天器 IP，加入列表
    ▼
ZMQ PUB 发布: GroundSystem.{hostname}.TelemetryPackets.{pkt_id}
    │
    ▼
TelemetrySystem.py (ZMQ SUB)
    │  订阅对应频道
    │  根据 telemetry-pages.txt 找到对应解析文件
    ▼
GenericTelemetry.py
    │  读取 .txt 定义文件（如 cfe-es-hk-tlm.txt）
    │  按 byte offset + length + struct format 逐字段解析
    │  在 GUI 表格中显示 Dec/Hex/Enm/Str
```

**telemetry-pages.txt 格式：**
```
描述,            Python类,             Packet ID, 解析定义文件
ES HK Tlm,      GenericTelemetry.py,   0x800,     cfe-es-hk-tlm.txt
EVS HK Tlm,     GenericTelemetry.py,   0x801,     cfe-evs-hk-tlm.txt
SB HK Tlm,      GenericTelemetry.py,   0x803,     cfe-sb-hk-tlm.txt
TBL HK Tlm,     GenericTelemetry.py,   0x804,     cfe-tbl-hk-tlm.txt
TIME HK Tlm,    GenericTelemetry.py,   0x805,     cfe-time-hk-tlm.txt
...
```

---

### 三、通过地面站学习各功能模块

GroundSystem 中的遥测和指令定义文件**直接对应** cFS 的核心服务模块。通过阅读这些文件，你可以快速理解每个模块的功能和状态。

#### 3.1 执行服务 ES (Executive Services) — `0x1806 / 0x800`

**遥测页**：`tlmGUI/cfe-es-hk-tlm.txt`
```
Command Counter       → 指令计数器
Error Counter         → 错误计数器
Registered Core Apps  → 已注册的核心应用数
Registered CFS Apps   → 已注册的 CFS 应用数
Registered Tasks      → 已注册的任务数
Registered Libs       → 已注册的库数
Reset Type/Subtype    → 复位类型（上电复位/软件复位等）
Processor Resets      → 处理器复位次数
Heap Bytes Free       → 堆空闲字节
Heap Blocks Free      → 堆空闲块数
Heap Max Blk Size     → 堆最大空闲块
cFE Core Checksum     → cFE 核心校验和
cFE/OSAL/PSP Version  → 各组件版本号
Syslog Bytes/Entries  → 系统日志使用情况
```

**对应源码**：`cfe/modules/es/fsw/`

#### 3.2 软件总线 SB (Software Bus) — `0x1803 / 0x803`

**遥测页**：`tlmGUI/cfe-sb-hk-tlm.txt`
```
NoSubscribersCounter        → 无订阅者消息计数（⚠️ 可能的安全事件）
MsgSendErrorCounter         → 消息发送错误
MsgReceiveErrorCounter      → 消息接收错误
CreatePipeErrorCounter      → 管道创建错误
SubscribeErrorCounter       → 订阅错误
DuplicateSubscriptionsCounter → 重复订阅计数
PipeOverflowErrorCounter    → 管道溢出（⚠️ 潜在 DoS）
MsgLimitErrorCounter        → 消息大小超限（⚠️ 潜在缓冲区溢出信号）
MemInUse / UnmarkedMem      → 内存使用/未标记内存
```

**对应源码**：`cfe/modules/sb/fsw/`

#### 3.3 事件服务 EVS (Event Services) — `0x1801 / 0x801`

**对应源码**：`cfe/modules/evs/fsw/`

#### 3.4 表管理 TBL (Table Services) — `0x1804 / 0x804`

**对应源码**：`cfe/modules/tbl/fsw/`

#### 3.5 时间服务 TIME (Time Services) — `0x1805 / 0x805`

**对应源码**：`cfe/modules/time/fsw/`

---

### 四、推荐的学习路径 🎯

```
第1步: 理解消息格式
   ├── 阅读 cmdUtil.c 的 CCSDS 包结构（Primary+Extended+CFS Secondary Header）
   └── 理解 CCSDS 标准的包格式（Packet Version, APID, Sequence, Length）

第2步: 跟踪指令下发全链路
   ├── GroundSystem GUI 点击按钮
   ├── → UdpCommands.py 调用 cmdUtil.c
   ├── → SendUdp.c 发送 UDP 到 127.0.0.1:1234
   ├── → apps/ci_lab/fsw/ 接收解析
   └── → cfe/modules/sb/fsw/ 发布到总线

第3步: 跟踪遥测上报全链路
   ├── cFS 应用产生遥测数据 → SB 发布
   ├── → apps/to_lab/fsw/ 订阅并打包
   ├── → UDP sendto 127.0.0.1:2234
   ├── → RoutingService.py 接收并 ZMQ 发布
   └── → TelemetrySystem.py → GenericTelemetry.py 解析显示

第4步: 逐个模块深入
   ├── ES: 理解应用生命周期管理
   ├── SB: 理解发布/订阅消息模型（安全分析核心）
   ├── EVS: 理解事件日志机制
   ├── TBL: 理解表管理
   └── TIME: 理解时间同步

第5步: 安全分析视角
   ├── SB 管道溢出/消息大小限制 → DoS
   ├── 无订阅者消息 → 可能的消息注入
   ├── CCSDS 包解析边界 → 缓冲区溢出
   └── cmdUtil 参数校验 → 命令注入
```

---

### 五、当前可直接操作的学习实验

1. **观察 SB 遥测计数器**：启动 GroundSystem → Enable Tlm → 查看 SB HK Tlm 页面，观察 `NoSubscribersCounter`、`PipeOverflowErrorCounter` 等字段的变化——这些计数器是安全分析的重要指标。

2. **发送 Noop 指令**：在 Command System 中选择不同模块（ES/SB/EVS...）→ 点击 Noop → 回到遥测页面观察 Command Counter 递增。

3. **阅读遥测定义文件**：`Subsystems/tlmGUI/cfe-sb-hk-tlm.txt` 中的每个字段，其 byte offset 和长度直接对应 `cfe/modules/sb/fsw/` 中 SB 的 Housekeeping 数据结构体——可以对照源码进行"定义→实现"双向验证。

需要我从以上哪个具体环节开始为你展开分析？建议先从 **SB（软件总线）** 入手，它是 cFS 消息通信的核心，也是安全审计最有价值的目标。