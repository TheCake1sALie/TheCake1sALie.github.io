---
categories: [""]
tags: ["Linux", "TrustZone" , "Qemu" , "OP-TEE"]
title: "None"
date: 2026-05-05
draft: True
---
# 从零搭建 QEMU + OP-TEE TrustZone 仿真环境：一次踩坑与成功的记录

## 引言
最近我在学习 ARM TrustZone 和可信执行环境（TEE），决定在 QEMU 上搭建一个开源的 OP-TEE 环境练练手。过程远比想象中波折：网络代理、磁盘空间爆满、编译链错误、串口登录失败……但最终看到 `41358 subtests of which 0 failed` 的那一刻，一切都值了。这篇文章记录了我的完整操作、踩过的坑和解决方法，希望能帮到同样在探索 TrustZone 的你。

---

## 1. 背景速览：TrustZone 与 OP-TEE
ARM TrustZone 是一种硬件安全扩展，它将处理器划分为**安全世界（Secure World）**和**普通世界（Normal World）**，两者在硬件层面隔离。安全世界可以运行一个独立的操作系统（称为 TEE），处理密钥、指纹等敏感数据；普通世界则运行常规的 Linux/Android 等。

OP-TEE 是由 Linaro 主导的开源 TEE 实现，它包含了安全世界操作系统（`optee_os`）、与普通世界通信的客户端库、测试套件等。QEMU 则可以模拟支持 TrustZone 的 ARMv8 平台，让我们无需真实开发板就能调试整个软件栈。

---

## 2. 实验环境
- **宿主机**：VMware 上的 Ubuntu 22.04 虚拟机（初始磁盘 40 GB，后扩容至 60 GB）
- **模拟目标**：QEMU 模拟 ARM Cortex-A 系列，TrustZone 使能
- **版本**：OP-TEE 主线主干（`qemu_v8.xml` manifest）
- **关键组件**：arm-trusted-firmware（BL1）、OP-TEE OS、Linux 内核、Buildroot 根文件系统、Rust 工具链

---

## 3. 搭建步骤回顾

### 3.1 安装系统依赖
打开终端，一次性安装所有编译需要的工具和库（确保 VPN 已连上，因为后续需要访问 Google 源）：
```bash
sudo apt update
sudo apt install -y adb acpica-tools autoconf automake bc bison build-essential \
    ccache cpio cscope curl device-tree-compiler e2tools expect fastboot flex \
    ftp-upload gdisk git libgnutls28-dev libattr1-dev libcap-ng-dev libfdt-dev \
    libftdi-dev libglib2.0-dev libgmp3-dev libhidapi-dev libmpc-dev libncurses5-dev \
    libpixman-1-dev libslirp-dev libssl-dev libtool libusb-1.0-0-dev make mtools \
    netcat ninja-build python3-cryptography python3-pip python3-pyelftools \
    python3-serial python3-tomli python-is-python3 rsync swig unzip uuid-dev wget \
    xdg-utils xsltproc xterm xz-utils zlib1g-dev
```
部分包名在不同 Ubuntu 版本可能微调（比如 `libgmp3-dev` 实际安装 `libgmp-dev`），遇到 `E: Unable to locate package` 时按提示纠正即可。

### 3.2 下载 repo 工具并拉取源码
OP-TEE 使用 `repo` 管理多个 git 仓库。由于从 Google 存储下载，请务必让终端走代理。
```bash
# 清理可能存在的旧 repo
sudo rm -f /usr/local/bin/repo
rm -rf ~/.repoconfig ~/.repo_

# 下载 repo 启动脚本
sudo curl -o /usr/local/bin/repo https://storage.googleapis.com/git-repo-downloads/repo
sudo chmod a+x /usr/local/bin/repo

# 创建工作目录并初始化 manifest
mkdir -p ~/File/optee && cd ~/File/optee
repo init -u https://github.com/OP-TEE/manifest.git -m qemu_v8.xml
repo sync -j2
```
首次 sync 会下载约 19 个仓库，耗时较久。成功后 `~/File/optee` 下会看到 `optee_os`、`build`、`linux` 等目录。

### 3.3 编译工具链
```bash
cd ~/File/optee/build
make toolchains -j2
```
这一步会下载 ARM 交叉编译器（AArch32 和 AArch64）以及 Rust 工具链。如果网络不稳定，可以多试几次；Make 具备断点续传能力。

### 3.4 编译整个系统并运行
```bash
make run -j2
```
这个命令会依次编译 arm-trusted-firmware、OP-TEE OS、Linux、Buildroot 以及 QEMU，最后自动启动 QEMU 模拟器。第一次编译可能长达一两个小时，请耐心等待。

---

## 4. 那些折磨我但最终被解决的坑

### 4.1 `repo` 显示 `<not installed>`
执行 `repo version` 时出现 `<repo not installed>` 是正常的，因为完整的 repo 工具链尚未克隆。只需直接进行 `repo init`，repo 会自动完成自我安装。

### 4.2 buildroot 编译 busybox 时 `Library resolv is needed` 报错
根本原因是交叉编译环境误检测了主机的 `libresolv`。解决方法：
1. 确保主机已安装 `libc6-dev`（通常已有）。
2. 彻底清理 buildroot 输出：
   ```bash
   cd ~/File/optee/build
   make buildroot-clean
   ```
3. 重新执行 `make run -j2`。

### 4.3 磁盘空间不足（`No space left on device`）
编译到 Rust 组件时突然报空间不足。检查发现虚拟机当初只分配了 40 GB，而 `out-br` 和 `toolchains` 占据了几十 GB。解决步骤：
1. 关闭虚拟机，在 VMware 设置中扩展虚拟磁盘到 60 GB。
2. 启动虚拟机，用 `sudo gparted` 调整根分区 `/dev/sda3` 大小以占用新空间。
3. 删除编译缓存：
   ```bash
   rm -rf ~/File/optee/out-br
   rm -f ~/File/optee/toolchains/*.tar.xz
   sudo apt clean
   sudo journalctl --vacuum-size=100M
   ```
4. 验证 `df -h /` 显示有至少 30 G 可用空间，然后重新编译。

### 4.4 启动后卡在 `soc_term: accepted fd ...` 无法登录
QEMU 启动时因为 `-S` 参数会暂停，必须在 QEMU monitor 中输入 `c` 继续执行。但很多人（包括我）误以为在普通终端直接敲 `c` 就行。正确操作：
- 在 QEMU 所在的终端按 `Ctrl+A` 然后松手，再单独按 `C`，进入 QEMU monitor（提示符 `(qemu)`）。
- 输入 `c` 并回车。
- 然后切换到 Normal World 对应的 xterm 窗口或 stdio 终端，按几次回车即可看到 `buildroot login:`。
- 用户 `root`，无密码。

如果上述方法不生效，可以重新启动 QEMU 并禁用多窗口模式，将所有输出合并到当前终端：
```bash
make run-only QEMU_SERIAL_CONSOLE=stdio
```

---

## 5. 验证：运行 xtest 测试
登录 Normal World 后，确认 `tee-supplicant` 已在后台运行：
```bash
ps | grep tee-supplicant
```
然后执行：
```bash
xtest
```
大约五分钟后，测试全部通过：
```
+-----------------------------------------------------
41358 subtests of which 0 failed
143 test cases of which 0 failed
0 test cases were skipped
TEE test application done!
```
这意味着安全世界与普通世界通信正常，OP-TEE 全部功能验证通过。

---

## 6. 总结与下一站
至此，一个完整的 QEMU+OP-TEE 仿真环境搭建成功。回顾整个过程，遇到的问题主要集中在网络、磁盘空间和对 QEMU 工作模式的理解上，都是初学者容易踩的坑。

通过这个环境，我们可以：
- 修改或开发自己的可信应用（TA），放到 `optee_examples` 中运行。
- 用 `xtest -t benchmark` 测试性能。
- 使用调试器 GDB 连接 QEMU 调试 OP-TEE OS 内核。

安全世界的门已经打开，下一步我计划深入学习 GlobalPlatform TEE 内部 API，编写一个简单的加解密 TA。欢迎一起交流！

---

**参考资源**
- [OP-TEE 官方文档](https://optee.readthedocs.io/)
- [QEMU 官方文档](https://www.qemu.org/docs/master/)
- [ARM TrustZone 介绍](https://developer.arm.com/ip-products/security-ip/trustzone)