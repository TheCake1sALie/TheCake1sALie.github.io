---
categories: ["nasa-cFS学习"]
tags: ["Linux", "nasa-cFS" , "OS"]
title: "nasa-cFS/README.md解释"
date: 2026-05-15
draft: True
---

deepseek解释github/cFS/README.md
我们开始，从头到尾，逐段细致地阅读和讲解这份 `README.md`。不用担心航天背景，我会把每个术语都“翻译”成你熟悉的概念。

---

## 开篇徽章
```markdown
[![Build and execute CFS with multiple configurations](https://github.com/nasa/cfs/actions/workflows/test-cfs-qemu.yml/badge.svg)](https://github.com/nasa/cFS/blob/dev/.github/workflows/test-cfs-qemu.yml)
```
这是一个 **GitHub Actions 的持续集成徽章**。意思是：每当有代码更新，GitHub 会自动按照 `.github/workflows/test-cfs-qemu.yml` 的脚本去编译并运行 cFS，然后把结果（通过/失败）显示在这里。对你来说，**这说明 cFS 有一套自动化测试，你可以用它来验证你搭建的环境是否“健康”**。

---

## 标题与第一段
```markdown
# Core Flight System - BUNDLE

The Core Flight System (cFS) is a generic flight software architecture framework
used on flagship spacecraft, human spacecraft, cubesats, and Raspberry Pi.
This repository is a bundle of submodules that make up the cFS framework.
Note the "lab" apps are intended as examples only, and enable this bundle to
build, execute, receive commands, and send telemetry.
This is not a flight distribution, which is typically made up of the cFE,
OSAL, PSP, and a selection of flight apps that correspond to specific mission
requirements.
```

**逐句讲解：**

- **cFS 是一个通用的飞行软件架构框架**。  
  “通用”意味着它不为一颗特定卫星设计，而是可以裁剪、扩展，适用于多种航天器——从旗舰级的大卫星（如火星车）、载人飞船，到微小的立方星（cubesats），甚至树莓派。**这说明它的应用范围极广。**

- **这个仓库是组成 cFS 框架的“子模块捆绑包（bundle）”。**  
  实际源代码分散在多个 Git 仓库（如 cFE, OSAL 等），这里用 Git 的 `submodule` 机制把它们“绑”在一起，方便你一次性拿到整套东西。

- **“lab”类应用仅是示例，它们让这个捆绑包能编译、运行、收指令、发遥测。**  
  这些示例应用（后文会看到，如 Command Ingest、Telemetry Output、Scheduler 等）**不是真正上天的东西**，只是教学和开发用的“乐高积木模型”，让你能立刻看到 cFS 动起来。

- **这不是一个真实的飞行版本。**  
  一个实际任务只挑它需要的部分：核心框架（cFE）、操作系统抽象层（OSAL）、平台支持包（PSP），再配上任务专属的应用。  
  **这对你是好消息**：你拿到的是一个干干净净的“骨架+示例”，非常适合学习和安全实验，不会被无关的复杂任务逻辑干扰。

---

## 关于测试与验证的声明
```markdown
This bundle has not been fully verified as an operational system, and is
provided as a starting point vs an end product. Testing of this bundle
consists of building, executing, sending setup commands and verifying
receipt of telemetry. Unit testing is also run, but extensive analysis is
not performed. All verification and validation per mission requirements is
the responsibility of the mission …
```

**核心意思：**

- 这个开源捆绑包**没有经过完整的、类似于上天产品那样的全面验证**。
- 目前的测试只做到：能编译、跑起来、发指令、收到遥测、单元测试通过。
- **没有进行深入的安全分析**。  
- 任何实际任务需要的严格验证，由任务自己负责。

**对你的安全研究意义巨大：**  
官方明确说了“extensive analysis is not performed”，这就是说，**这个开源版本很可能残留着未发现的安全漏洞**。你正好可以在这里施展拳脚——官方没做的深入分析，你来做。

---

## 发行版说明
```markdown
## Distributions

This is the open-source version of cFS, released under an Apache 2.0 license.
The open source cFS is limited to the framework and common apps, libraries,
and tools, which includes and is limited to: cFE, OSAL, PSP, Command Ingest
(Lab), Telemetry Output (Lab), Scheduler (Lab), Sample App, Sample Lib,
Data Storage, File Manager, HouseKeeping, Health and Safety, Memory Dwell,
CFDP File Transfer, CheckSum, Limit Checker, Memory Manager, Stored Command,
cFS Ground System, elf2cfetbl, and tblCRCTool.
Changes to the open repositories are limited to bug fixes and minor
enhancements to those components.

A Government-use (Distro C) version of cFS with features for a full flight
mission is available through a Software User Agreement.
```

**细致讲解：**

- **开源版（你拿到的）** 使用 **Apache 2.0 许可证**，非常宽松，允许商用、修改、分发。
- 它列出了一大串包含的组件，这里我们按角色分类解释：

| 类别 | 组件 | 在你的“卫星安卓系统”比喻中的角色 |
|------|------|--------------------------------|
| 核心 | cFE, OSAL, PSP | 操作系统内核 + 硬件抽象 + 平台适配 |
| 实验室示例应用 | Command Ingest (Lab) | **指令接收员**：接收地面发来的指令 |
|  | Telemetry Output (Lab) | **遥测发报员**：向地面发送卫星状态数据 |
|  | Scheduler (Lab) | **调度员**：定时触发某些任务 |
|  | Sample App, Sample Lib | **示范模块**：演示怎么开发 cFS 应用 |
| 通用应用 | Data Storage (DS) | **数据仓库**：管理数据存储 |
|  | File Manager (FM) | **文件管家**：管理文件系统 |
|  | Housekeeping (HK) | **内务官**：定期收集所有应用的“健康报告” |
|  | Health and Safety (HS) | **安全监察官**：监控系统异常，必要时重启或降级 |
|  | Memory Dwell (MD) | **内存观察者**：读取特定内存地址的内容（调试用） |
|  | CFDP File Transfer (CF) | **星际文件快递**：用 CCSDS 标准协议传文件 |
|  | CheckSum (CS) | **校验员**：校验内存或文件完整性 |
|  | Limit Checker (LC) | **阈值警报器**：监测参数是否超限 |
|  | Memory Manager (MM) | **内存管理者**：管理内存池，分配/释放 |
|  | Stored Command (SC) | **指令剧本**：自动按时序执行预先存储的指令 |
| 地面工具 | cFS Ground System | **地面控制站软件**：图形界面，与天上的 cFS 通信 |
| 工具 | elf2cfetbl, tblCRCTool | 转换工具：把编译后的表格数据转成 cFS 能用的格式 |

- 官方声明：**开源版的修改仅限于 bug 修复和小改进。** 真正用于全任务的高级功能，需要一个政府版（Distro C），需要签协议才能拿到。

**对你的启示：**  
你当前接触的组件已经非常丰富。这些应用就是你的安全审计对象。比如：
- 指令接收（Command Ingest）如果被恶意篡改，会发生什么？
- 遥测输出能否被伪造或监听？
- 内存驻留（Memory Dwell）和内存管理器（Memory Manager）是否存在内存破坏漏洞？

---

## 联系方式与支持
```markdown
## Contact

cFS/cFE is owned and maintained by NASA Goddard Space Flight Center's
Flight Software Systems Branch.
…
To connect with the cFS team, please send an email to cfs-program@lists.nasa.gov
```

这部分说明项目的归属（NASA 戈达德太空飞行中心）和联系途径。对你有用的信息是：**官方提供培训和支持**，但可能需要额外协议。如果你未来需要深入的合作，这也是一个入口。

---

## 文档索引
```markdown
## References Documentation
- cFE User's Guide
- OSAL User's Guide
- cFE App Developer's Guide
- Training documentation
- cFS Overview
```

这些都是权威学习资料，你可以直接下载 PDF。**强烈建议你之后阅读 `cFE User's Guide` 和 `cFE App Developer's Guide`**，它们会解释 cFS 的内部消息机制、系统调用、应用如何收发消息——这些是你的安全分析的直接依据。

---

## 发布说明
```markdown
## Release Notes
…
**v7.0.1: OFFICIAL RELEASE**:
- Note: The cFS bundle structure and build system will be updated in a coming
  release…
- Includes cFE 7.0.1 (cFE, PSP, framework apps, and framework tools) and OSAL 7.0.1
```

当前你拿到的是 **v7.0.1 版本**。注意，构建系统和捆绑结构未来会有变化，但你可以用这个稳定版开始研究。

---

## 寻求帮助
```markdown
## Getting Help
### Discussions
### Join the mailing list
### Contact the cFS Product Team
```

你可以通过 GitHub Discussions 提问，或者订阅邮件列表。如果遇到编译或理解问题，可以直接从社区获取帮助，这也是你融入 cFS 用户生态的入口。

---

## 环境搭建（Setup）——**这是你动手的第一步**
```markdown
## Setup

Ensure the following software are installed: Make, CMake, GCC, and Git.
To setup the cFS BUNDLE directly from the latest set of interoperable
repositories:

    git clone https://github.com/nasa/cFS.git
    cd cFS
    git submodule init
    git submodule update
```

**详细解释：**

- 你需要安装四个基础工具：
  - `make`：构建自动化工具。
  - `CMake`：跨平台构建系统生成器。
  - `GCC`：GNU C 编译器。
  - `Git`：版本控制。
- **克隆仓库并初始化子模块**是拿到所有源代码的关键步骤。  
  `git submodule init` 和 `git submodule update` 会去下载 `cFE`、`OSAL`、`PSP` 以及各个应用的实际代码到子目录。  
  这一步之后，你本地就有一份完整的、可编译的 cFS 源代码了。

---

## 编译与运行快速入门（Build and Run Quick Start）——**这是最关键的部分**
```markdown
## Build and Run Quick Start

make native_std.prep    # Sets up the build tree
make native_std.install # Compiles the software and stages it to the exe directory
make native_std.runtest # Executes the tests
make native_std.lcov    # Executes lcov to collect coverage metrics

In order to boot CFE, the default linux PSP requires that the working directory
be set to the location of the staged binaries:

    cd build-native_std/exe/cpu1/
    ./core-cpu1

Should see startup messages, and CFE_ES_Main entering OPERATIONAL state.
```

**逐命令解释（用比喻）：**

- `make native_std.prep`  
  **“画图纸，搭脚手架”**。`native_std` 是针对你当前这台 Linux 机器的配置。`prep` 目标会调用 CMake 生成构建目录和 Makefile。

- `make native_std.install`  
  **“砌墙、盖房”**。编译所有源代码，并把生成的可执行文件、库文件放到一个统一的目录 `build-native_std/exe/cpu1/` 下。注意，cFS 模拟一个 CPU（cpu1 是名字）。

- `make native_std.runtest`  
  **“压测验收”**。运行所有单元测试，检查各个模块是否正常。  
  如果测试有失败，说明你环境可能有问题。

- `make native_std.lcov`  
  **“测量测试覆盖率”**。用 `lcov` 工具统计代码覆盖率，让你知道测试执行了百分之多少的源代码。对你安全测试来说，**覆盖率越低的地方，可能越容易藏匿未发现的漏洞**。

- **运行 cFS 核心**：  
  `cd build-native_std/exe/cpu1/`  
  `./core-cpu1`  
  这时，你会看到一系列启动日志，最后出现 `CFE_ES_Main entering OPERATIONAL state` 表示**核心飞行执行器（cFE）的主函数已经进入了正常运行状态**。这就是你“卫星的安卓系统”启动了。

> ⚠️ 你必须从 `build-native_std/exe/cpu1/` 这个目录启动程序，因为可执行文件要在这个相对路径下找到启动脚本和动态库。

---

## 其他构建配置
```markdown
### Other Build configurations
 - native_std: 本地开发调试版
 - native_eds: 启用 EDS（电子数据表）的版本
 - osal: 单独编译 OSAL 库
 - edslib: 单独编译电子数据表库
 - pc686_rtems5: 交叉编译到 RTEMS 5 实时系统，可在 QEMU 里跑
 - gr712_rtems5: 交叉编译到 GR712 处理器 + RTEMS 5
 - rpi_vxworks7: 树莓派 + VxWorks 7
 - rpi_linux: 树莓派 + Linux
 - qemu_yocto_linux: Yocto/空间级 Linux，交叉编译并在 QEMU 中运行
```

**对你安全研究的意义**：  
你可以先用 `native_std` 在本地 PC 上快速验证漏洞，然后扩展到 `pc686_rtems5` 或 `qemu_yocto_linux` 等更接近真实航天环境的配置，**在模拟器中攻击 cFS，观察不同底层 OS 下漏洞表现形式是否一致**。这是非常标准的嵌入式安全研究路径。

---

## 其他构建目标
```markdown
### Other build goals
 - prep: 准备构建树
 - compile: 只编译，不安装
 - install: 编译并安装到输出目录
 - runtest: 执行单元测试
 - lcov: 计算覆盖率
 - detaildesign: 生成 Doxygen 文档
 - usersguide: 生成用户指南
 - osalguide: 生成 OSAL 指南
 - image: 生成容器/VM 可启动镜像
```

这些 goal 你可以自由组合，比如 `make rpi_linux.install`。**生成文档的命令**（`detaildesign` 等）可以帮你导出 cFS 内部函数调用关系，辅助逆向和分析。

---

## 发送指令和接收遥测——**把你的“卫星”和地面连起来**
```markdown
## Send commands, receive telemetry

The cFS-GroundSystem tool can be used to send commands and receive telemetry.
…
1. Install PyQt5 and PyZMQ
2. Compile cmdUtil and start the ground system executable
       cd tools/cFS-GroundSystem/Subsystems/cmdUtil
       make
       cd ../..
       python3 GroundSystem.py
3. Select "Start Command System"
4. Select "Enable Tlm"
5. Enter IP address of system executing cFS, 127.0.0.1 if running locally
6. Select "Start Telemetry System"
```

**大白话讲解：**

- **cFS-GroundSystem** 是一个 Python 写的图形界面工具，就像卫星地面控制中心的操作屏幕。
- 它通过 PyZMQ（一种消息队列库）与你的本地 cFS 进程通信（IP `127.0.0.1` 表示本机）。
- 启动后，你可以：
  - 点击按钮发送“指令”（比如 no-op 空操作指令），看到指令计数器增长。
  - 接收并显示遥测数据（比如系统状态、各应用的心跳包）。

**这对安全研究非常重要**：  
你可以在这里：
- 抓包分析指令和遥测的格式。
- 尝试构造畸形指令，观察 cFS 是否崩溃（模糊测试的入口）。
- 分析消息的来源验证机制是否存在弱点。

---

## 添加自定义应用
```markdown
### Adding Custom Apps

See the cFE Application Developer's Guide for information about creating new
apps. It is encouraged to use sample_app or skeleton_app as a starting point.
Once the application has been created, it must be added to targets.cmake to
be compiled and to the .scr file to be dynamically loaded at runtime.
```

你可以自己写一个 cFS 应用，插进去一起跑。例如，你可以写一个**安全监控应用**，监控其他应用的异常行为；或者写一个**攻击工具包**，在系统内故意执行恶意操作，验证防御机制。添加的方法：修改 `targets.cmake` 加入编译，在 `.scr` 启动脚本中让它被动态加载。

---

## 相关能力（生态）
```markdown
## Related Capability
- Distributions: CFS-101 虚拟机发行版
- Other Ground station software: EDS-GroundStation
- Other Apps: BPNode, CS, CF, DS, FM, HK, HS, LC, MD, MM, SBN, SC, SCA, Skeleton App
- Other Interfaces: cFS COSMOS Plugin, cFS Command Line Tools, SIL, ECI, SBN-Client
- Other Libraries: BPLib, CFS_IO_LIB, cfs_lib, EdsLib, fs_lib
- Other Tools: CTF (测试框架), CCDD, DTNTools, Perfutils-java, gen_sch_tbl, CryptoLib
```

这里列出了 cFS 的开源生态。对你安全研究最重要的可能是：
- **CryptoLib**：实现了 CCSDS 空间数据链路安全协议，**正是你研究通信安全、加密认证机制的好靶子**（之前提到过它存在 CVE-2024-449XX）。
- **SBN (软件总线网络)**：让多颗卫星或模块之间通信，**这是分布式系统的安全信任边界，很有研究价值**。
- **CTF (cFS 测试框架)**：可以帮你编写自动化安全测试用例。

---

## 总结：你现在手里有什么？

读完这份 `README.md`，你应该清楚：

1.  **这是一个完整的、可运行的开源“卫星操作系统”学习平台。**
2.  **它在你本地 Linux 电脑上就能跑**，你马上就能拥有一个“虚拟卫星”的核心软件环境。
3.  **它有清晰的模块划分和示例应用**，你可以针对性地分析每个模块的代码和通信协议。
4.  **官方承认没有做深度安全分析**，这留下了广大的研究空间。

**建议你立刻执行以下操作，作为正式启动：**

1.  在 Linux 环境（虚拟机即可）中，严格按照 `Setup` 和 `Build and Run Quick Start` 的步骤操作，直到看见 `CFE_ES_Main entering OPERATIONAL state`。
2.  搭建地面系统，尝试发送 `noop` 指令并观察遥测。
3.  把成功运行的截图或日志记录下来，作为你项目的第一个里程碑。

当你亲手让这个“航天系统”在你的电脑上活起来的那一刻，你对它的理解和感觉会完全不同。一旦你完成了这个环境搭建，我们就可以进一步深入到具体的源码模块（比如软件总线 cFE SB）和已经公开的 CVE 漏洞分析中。有任何卡住的地方，随时告诉我，我们一步步解决。