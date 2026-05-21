---
categories: ["nasa-cFS学习"]
tags: ["Linux", "nasa-cFS" , "OS"]
title: "使用SSH在vscode里分析虚拟机内cFS代码"
date: 2026-05-16
draft: True
---

Deepseek指导版

---
好的，这是完整的 SSH 远程连接操作指南，包含方案二（动态 IP 手动更新）的全部步骤。

---

## 🔗 VS Code Remote-SSH 连接 cFS 虚拟机完整指南

### 📋 前置条件
- Windows 宿主机已安装 **VS Code**
- Ubuntu 虚拟机在 VMware 中正常运行
- Ubuntu 已安装 SSH 服务（首次需要执行安装步骤）

---

### 🛠️ 第一步：在 Ubuntu 虚拟机中安装并启用 SSH 服务（仅首次）

在 Ubuntu 终端中执行：

```bash
sudo apt install openssh-server -y
sudo systemctl enable ssh --now
```

验证是否启动成功：

```bash
sudo systemctl status ssh
```

看到 `active (running)` 即表示成功。

---

### 📡 第二步：查看 Ubuntu 的当前 IP 地址

```bash
hostname -I
```

输出示例：`192.168.107.131`

> ⚠️ 每次虚拟机重启后 IP 可能变化，连接前务必先查一次。

---

### 💻 第三步：在 VS Code 中安装 Remote-SSH 扩展

1. 打开 Windows 上的 VS Code
2. 点击左侧活动栏的 **扩展** 图标（或按 `Ctrl+Shift+X`）
3. 搜索 **Remote - SSH**（作者：Microsoft）
4. 点击 **安装**

---

### ➕ 第四步：添加 SSH 主机（仅首次）

1. 点击 VS Code 左下角的 **绿色 `><` 图标**，或按 `F1` 输入 `Remote-SSH: Connect to Host...`
2. 选择 **“Add New SSH Host...”**（添加新 SSH 主机）
3. 输入连接命令（**请替换成你查到的 IP**）：
   ```
   ssh eric@192.168.107.131
   ```
4. 按 `Enter`，选择 SSH 配置文件保存位置（推荐默认的 `C:\Users\你的用户名\.ssh\config`）
5. 右下角弹出提示，点击 **“Connect”**（连接）

---

### 🔐 第五步：首次连接时确认主机指纹

- 弹出“主机指纹”提示时，点击 **“Continue”**（继续）
- 输入 Ubuntu 用户的登录密码（`eric` 的密码）
- 等待连接完成，左下角状态栏变为 **`SSH: 192.168.107.131`**

---

### 📂 第六步：打开 cFS 工程目录

1. 在 VS Code 左侧点击 **资源管理器**（文件图标）
2. 点击 **“打开文件夹”**
3. 输入路径：
   ```
   /home/eric/File/cFS
   ```
4. 点击 **确定**，cFS 源码即加载到 VS Code 中

---

### 🔁 第七步：后续每次连接的操作流程

1. 确保 Ubuntu 虚拟机正在运行
2. 在 Ubuntu 终端执行 `hostname -I` 查看最新 IP
3. **如果 IP 变了**，按第八步更新配置；如果没变，直接跳到第4步
4. 打开 VS Code，点击左下角绿色 `><` 图标 → 选择 **“Connect to Host...”**
5. 选择你的主机（如 `eric@192.168.107.131`）
6. 输入密码，即可自动恢复上次打开的所有文件和终端

---

### 🔧 第八步：IP 变化后更新 SSH 配置

1. 按 `F1`，输入 `Remote-SSH: Open SSH Configuration File...`
2. 选择 `C:\Users\你的用户名\.ssh\config`
3. 找到类似下面这段配置：
   ```
   Host 192.168.107.131
       HostName 192.168.107.131
       User eric
   ```
4. 将 `HostName` 后面的旧 IP 改为新 IP（两行都改）
5. 保存文件，重新按第七步连接即可

---

### 🧩 推荐安装的扩展（在远程环境中）

连接到 SSH 后，按 `Ctrl+Shift+X` 打开扩展面板，搜索并安装以下扩展（**点击“在 SSH: … 中安装”**）：

- **C/C++** (Microsoft)：智能代码补全、跳转、错误提示
- **Python** (Microsoft)：查看 `.py` 地面系统脚本
- **CMake Tools** (Microsoft)：如果需要重新配置构建

---

### ✅ 环境就绪检查清单

| 检查项 | 现象 |
|--------|------|
| Ubuntu 虚拟机运行中 | 能登录，能执行命令 |
| SSH 服务运行 | `sudo systemctl status ssh` 显示 active |
| VS Code 已连上 | 左下角显示 `SSH: <IP>` |
| cFS 文件夹已打开 | 左侧资源管理器中能看到 `apps`、`cfe`、`osal` 等目录 |
| C/C++ 扩展已安装 | 打开 `.c` 文件有语法高亮和补全提示 |

---

恭喜！现在你可以在 Windows 的 VS Code 中舒适地浏览、搜索、分析 cFS 的全部源码。准备好后告诉我，我们开始深入分析核心组件（例如软件总线 `cfe/sb`）或复现 CVE。