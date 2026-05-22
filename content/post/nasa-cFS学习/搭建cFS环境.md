---
categories: ["nasa-cFS学习"]
tags: ["Linux", "nasa-cFS" , "OS"]
title: "从零搭建cFS环境"
date: 2026-05-15
draft: false
---

选择Deepseek辅助指导
---

## 📥 第一步：下载 Ubuntu 22.04 LTS 镜像

1.  打开浏览器，访问 Ubuntu 官网下载页面：
    `https://ubuntu.com/download/desktop`
2.  找到 **Ubuntu 22.04 LTS** 版本（桌面版），点击下载，会得到一个 `.iso` 文件（大约 4GB）。
3.  保存到电脑上，我选择的文件是 `ubuntu-22.04.5-desktop-amd64.iso`。

---

## 🖥️ 第二步：在 VMware 中创建新虚拟机

1.  打开 VMware Workstation，点击 **“创建新的虚拟机”**。
2.  选择 **“典型（推荐）”**，点击下一步。
3.  选择 **“安装程序光盘映像文件(iso)”**，浏览并选中刚才下载的 Ubuntu ISO 文件。
4.  在 **“简易安装信息”** 页面填写：
    *   全名：你的名字（随意）
    *   用户名：你将来登录 Ubuntu 的账号名
    *   密码：输入一个容易记的密码（注意大小写）
    *   确认密码
5.  虚拟机名称我选择改成 `Ubuntu 64 位 for cFS nos3`，位置选择一个有足够空间的盘符（建议剩余空间 >60GB）。
6.  **磁盘容量**：设置为 **100 GB**，并选择 **“将虚拟磁盘拆分为多个文件”**。
    > 为什么是 60GB？Ubuntu 系统约 10-15GB，cFS 及其编译产物约 5-10GB，再预留一些空间给 NOS3 或后续实验，60GB 比较稳妥。
7.  点击 **“自定义硬件”**：
    *   **内存**：建议设为 **8GB**（4096 MB 勉强可以，但编译会较慢）。
    *   **处理器**：至少 2 个核心。
    *   **网络适配器**：先选 **“NAT”**，简单方便；后续如需桥接再改。
8.  确认所有设置，点击完成，虚拟机会自动开机并从 ISO 安装系统。

---

## 🐧 第三步：完成 Ubuntu 安装

此时虚拟机会自动进入 Ubuntu 安装程序。因为是“简易安装”，VMware 会帮你填入大部分信息，你可能只需要等待它自动安装完成。如果弹出选择时区、键盘布局等，根据提示选择上海或默认即可。

安装完成后系统会重启，进入登录界面，输入你之前设置的用户名和密码登录。恭喜！一个全新的 Ubuntu 就装好了。

---

## 🔧 第四步：安装 VMware Tools 并更新系统

为了让虚拟机用得更顺手（全屏、剪贴板共享、文件拖拽等），必须安装 open-vm-tools。

登录后，按 `Ctrl+Alt+T` 打开终端，依次执行：

```bash
# 更新软件源
sudo apt update

# 安装 open-vm-tools 及其桌面组件
sudo apt install open-vm-tools open-vm-tools-desktop -y

# 升级系统中已安装的所有软件包（可选但推荐，保证环境最新）
sudo apt upgrade -y
```

安装完成后，**重启虚拟机**：
```bash
sudo reboot
```
重启后你会发现屏幕分辨率自动跟随窗口大小，鼠标也能自由进出，且可以从宿主机直接拖拽文件进 Ubuntu 了。

---

## 📂 第五步：配置共享文件夹（方便传递文件）

1.  在 Windows 上新建一个文件夹，比如 `D:\Virtual Machine Share`。
2.  在 VMware 菜单栏，点击 **虚拟机** -> **设置** -> **选项** 选项卡 -> **共享文件夹**。
3.  选择 **“总是启用”**，点击 **“添加”**，路径选 `D:\Virtual Machine Share`，名称保留默认的 `Virtual Machine Share`，点击下一步，完成。
4.  回到 Ubuntu 终端，我们需要挂载这个共享文件夹。先看看共享文件夹是否被系统识别：
    ```bash
    vmware-hgfsclient
    ```
    如果输出 `cFS_shared`，说明识别成功。
5.  创建挂载点并挂载：
    ```bash
    sudo mkdir /mnt/hgfs
    sudo mount -t fuse.vmhgfs-fuse .host:/Virtual\ Machines\ Share /mnt/hgfs -o allow_other
    ```
6.  测试：在 Windows 下往 `D:\cFS_shared` 丢一个文件，然后在 Ubuntu 终端输入 `ls /mnt/hgfs`，应该能看到该文件。

---

## 🧪 第六步：安装编译 cFS 所需的基础工具

现在环境干净了，我们安装 cFS README 里要求的 `make`, `cmake`, `gcc`, `git`：

```bash
sudo apt install make cmake gcc git -y
```

验证一下版本（没有报错就说明装好了）：
```bash
make --version
cmake --version
gcc --version
git --version
```

---

## 🚀 第七步：克隆 cFS 并下载子模块

```bash
# 进入主目录（或任何你想放代码的地方）
cd ~

# 新建新文件夹File
mkdir File
cd File

# 克隆 cFS bundle
git clone https://github.com/nasa/cFS.git

# 进入 cFS 目录
cd cFS

# 初始化并下载所有子模块（cFE, OSAL, PSP, 各种 apps）
git submodule init
git submodule update
```

这一步可能需要几分钟，取决于你的网速。如果中途卡住，可以 `Ctrl+C` 后重试，子模块下载是断点续传的。

### 1️⃣ `git submodule` 是什么？

`submodule` 中文是“**子模块**”。简单来说，子模块就是**一个 Git 仓库里面，引用了另一个独立的 Git 仓库**。主仓库（cFS BUNDLE）只记录子模块的“**快照指针**”，指向子仓库的某个特定提交，而不是把子仓库的所有文件都复制过来。

> 💡 **打个比方**：就像你的电脑上装了很多软件，每个软件都有自己的版本号和更新日志。cFS BUNDLE 就像你的操作系统，而 cFE、OSAL、PSP 这些子模块就像一个个独立的软件。操作系统只需要记录“我需要的是 QQ v1.2.3”，而不需要把 QQ 的所有历史版本都下载下来。

**cFS 为什么要用 submodule？** 因为 cFS 由一个核心框架和几十个独立 App 组成，每个 App 都由专门的团队负责开发、测试和更新。这种模块化管理有三大好处：
- **独立维护**：每个子模块都有自己的版本号和发布周期，互不干扰。
- **精准控制**：主仓库（cFS BUNDLE）会锁定每个子模块的特定稳定版本，确保所有组件都经过测试，能协同工作。
- **清晰复用**：如果有其他项目想用 cFS，可以只引用自己需要的子模块，不用把所有东西都下载下来。

---

## 🏗️ 第八步：编译 cFS

根据 README，我们使用 `native_std` 配置进行本地编译：

```bash
# 准备构建树（生成 CMake 缓存）
make native_std.prep

# 编译并安装到输出目录
make native_std.install
```

编译过程大约需要 5-15 分钟，期间会有很多绿色的编译信息滚过。如果最后没有 `error` 且返回提示符，即表示成功。

你也可以先跑一遍单元测试（非必需但强烈建议，用于确认环境完全正常）：
```bash
make native_std.runtest
```
如果测试全部通过，你会看到 `100% tests passed` 的提示。如果有失败项，记下来，我们可以排查。

---

## 🌟 第九步：启动 cFS 核心，看见历史性的一刻

```bash
cd build-native_std/exe/cpu1/
./core-cpu1
```

终端会疯狂刷屏，显示各种应用的初始化信息。静待十几秒，直到出现：

```
CFE_ES_Main: Entering OPERATIONAL state
```

🎉 **这颗“卫星的大脑”就在你的虚拟机里成功运行了！**
注意：滚动信息真的很多，可以用ctrl+shift+f组合键寻找指令CFE_ES_Main: Entering OPERATIONAL state

按 `Ctrl+C` 可以退出。

---

### 🌍 第十步：快速体验地面系统（完整版）

如果想立刻向 cFS 核心发送指令并接收遥测数据，请在**保持 `./core-cpu1` 运行的同时**，新开一个终端窗口，依次执行以下命令：

```bash
# 1. 安装地面系统所需的 Python 依赖和 Qt 图形库
sudo apt install python3-pip libcanberra-gtk-module libcanberra-gtk3-module -y
pip3 install PyQt5 PyZMQ

# 2. 补充安装 Qt 平台插件所需的 xcb 图形库（缺少会导致 GUI 无法启动）
sudo apt install libxcb-xinerama0 libxcb-icccm4 libxcb-image0 libxcb-keysyms1 libxcb-render-util0 libxcb-xkb1 libxkbcommon-x11-0 -y

# 3. 编译地面系统所需的指令发送工具 cmdUtil
cd ~/File/cFS/tools/cFS-GroundSystem/Subsystems/cmdUtil
make

# 4. 启动地面站图形界面
cd ../..
python3 GroundSystem.py
```
**目前还没有玩明白这个地面系统，指令很多，功能很密集**
---

✅ **至此，一个纯净、优化、可直接用于安全研究的 cFS 实验环境已经搭建完毕。**