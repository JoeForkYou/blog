---
abbrlink: 46
title: installation
date: '2025-03-21 20:49:56'
---
# Zephyr 工具链安装

本文档详细介绍了如何在不同操作系统上安装和配置 Zephyr RTOS 开发所需的工具链，包括 SDK 安装步骤、交叉编译器配置以及常见问题解决方案。

## 支持的操作系统

Zephyr 开发工具链可以在以下操作系统上安装：

### Linux

- **Ubuntu** 20.04 LTS 或更高版本（推荐）
- **Fedora** 33 或更高版本
- **Clear Linux OS**
- 其他 Linux 发行版（可能需要额外配置）

### macOS

- macOS 10.15 (Catalina) 或更高版本
- 支持 Intel 和 Apple Silicon 处理器

### Windows

- Windows 10 或更高版本
- 支持原生安装或 WSL (Windows Subsystem for Linux)

## 先决条件

在安装 Zephyr SDK 之前，需要先安装一些基本依赖：

### Linux (Ubuntu) 依赖

```bash
sudo apt update
sudo apt install --no-install-recommends git cmake ninja-build gperf \
  ccache dfu-util device-tree-compiler wget \
  python3-dev python3-pip python3-setuptools python3-tk python3-wheel xz-utils file \
  make gcc gcc-multilib g++-multilib libsdl2-dev
```

### macOS 依赖

```bash
# 安装 Homebrew (如果尚未安装)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# 安装依赖
brew install cmake ninja gperf python3 ccache qemu dtc
```

### Windows 依赖

1. 安装 [Git for Windows](https://git-scm.com/download/win)
2. 安装 [Python 3.8](https://www.python.org/downloads/) 或更高版本
3. 安装 [CMake](https://cmake.org/download/)
4. 安装 [Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)

## 安装 West 工具

West 是 Zephyr 的元工具，用于管理多仓库项目。在所有平台上，可以使用 pip 安装 West：

```bash
# Linux/macOS
pip3 install --user -U west

# Windows
pip install west
```

在 Linux/macOS 上，确保将 `~/.local/bin` 添加到 PATH 环境变量：

```bash
echo 'export PATH=~/.local/bin:"$PATH"' >> ~/.bashrc
source ~/.bashrc
```

## 获取 Zephyr 源码

使用 West 初始化并获取 Zephyr 源码：

```bash
# 创建工作目录并初始化
west init -m https://github.com/zephyrproject-rtos/zephyr --mr v2.9.1 ~/zephyrproject

# 进入工作目录
cd ~/zephyrproject

# 获取所有项目代码
west update

# 安装 Python 依赖
pip3 install -r zephyr/scripts/requirements.txt
```

## SDK 安装步骤

### Linux SDK 安装

1. **下载 SDK**

```bash
cd ~
wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/zephyr-sdk-0.16.1_linux-x86_64.tar.gz
```

2. **解压 SDK**

```bash
tar xvf zephyr-sdk-0.16.1_linux-x86_64.tar.gz
```

3. **运行安装脚本**

```bash
cd zephyr-sdk-0.16.1
./setup.sh
```

4. **安装 udev 规则（可选，用于访问开发板）**

```bash
sudo cp -a ~/zephyr-sdk-0.16.1/sysroots/x86_64-pokysdk-linux/usr/share/openocd/contrib/60-openocd.rules /etc/udev/rules.d
sudo udevadm control --reload
```

### macOS SDK 安装

1. **安装 ARM 工具链**

```bash
brew install gcc-arm-embedded
```

2. **或者下载 Zephyr SDK（可选）**

```bash
cd ~
wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/zephyr-sdk-0.16.1_macos-x86_64.tar.gz
tar xvf zephyr-sdk-0.16.1_macos-x86_64.tar.gz
cd zephyr-sdk-0.16.1
./setup.sh
```

### Windows SDK 安装

1. **使用 WSL（推荐）**

在 WSL 中按照 Linux 安装步骤进行安装。

2. **原生 Windows 安装**

- 下载 [Zephyr SDK for Windows](https://github.com/zephyrproject-rtos/sdk-ng/releases)
- 解压到合适的位置，如 `C:\zephyr-sdk-0.16.1`
- 运行 `setup.cmd` 脚本
- 设置环境变量 `ZEPHYR_SDK_INSTALL_DIR` 指向 SDK 安装目录

## 交叉编译器配置

### 配置环境变量

在 Linux/macOS 上，将以下内容添加到 `~/.bashrc` 或 `~/.zshrc`：

```bash
export ZEPHYR_BASE=~/zephyrproject/zephyr
export ZEPHYR_SDK_INSTALL_DIR=~/zephyr-sdk-0.16.1
export ZEPHYR_TOOLCHAIN_VARIANT=zephyr
```

在 Windows 上，设置系统环境变量：

1. 打开"系统属性" > "环境变量"
2. 添加新的系统变量：
   - `ZEPHYR_BASE`: `C:\zephyrproject\zephyr`
   - `ZEPHYR_SDK_INSTALL_DIR`: `C:\zephyr-sdk-0.16.1`
   - `ZEPHYR_TOOLCHAIN_VARIANT`: `zephyr`

### 使用其他工具链

除了 Zephyr SDK，还可以使用其他工具链：

1. **GNU Arm Embedded**

```bash
export ZEPHYR_TOOLCHAIN_VARIANT=gnuarmemb
export GNUARMEMB_TOOLCHAIN_PATH=/path/to/arm-gnu-toolchain
```

2. **Espressif ESP32 工具链**

```bash
export ZEPHYR_TOOLCHAIN_VARIANT=espressif
export ESPRESSIF_TOOLCHAIN_PATH=/path/to/espressif-toolchain
```

3. **LLVM/Clang**

```bash
export ZEPHYR_TOOLCHAIN_VARIANT=llvm
```

## 验证安装

验证工具链安装是否成功：

```bash
# 检查 West 版本
west --version

# 构建示例应用
cd ~/zephyrproject
west build -b qemu_x86 zephyr/samples/hello_world

# 运行示例
west build -t run
```

## 安装其他工具

### 调试工具

1. **OpenOCD**

```bash
# Ubuntu
sudo apt install openocd

# macOS
brew install open-ocd

# Windows
# 下载并安装 OpenOCD 二进制包
```

2. **J-Link**

从 [SEGGER 网站](https://www.segger.com/downloads/jlink/) 下载并安装 J-Link 软件包。

3. **pyOCD**

```bash
pip3 install --user -U pyocd
```

### 烧录工具

1. **nrfjprog（用于 Nordic 设备）**

从 [Nordic 网站](https://www.nordicsemi.com/Software-and-tools/Development-Tools/nRF-Command-Line-Tools) 下载并安装。

2. **STM32 工具**

从 [ST 网站](https://www.st.com/en/development-tools/stm32cubeprog.html) 下载并安装 STM32CubeProgrammer。

## 工具链更新

定期更新工具链以获取最新功能和修复：

```bash
# 更新 West
pip3 install --user -U west

# 更新 Zephyr 源码
cd ~/zephyrproject
west update

# 更新 Python 依赖
pip3 install --user -U -r zephyr/scripts/requirements.txt

# 更新 SDK（需要重新下载并安装新版本）
```

## 常见问题解决

### 1. 找不到工具链

**问题**：构建时报错 "Unable to find toolchain"

**解决方案**：
- 确认 `ZEPHYR_SDK_INSTALL_DIR` 环境变量已正确设置
- 确认 SDK 已正确安装
- 检查 `ZEPHYR_TOOLCHAIN_VARIANT` 是否正确设置

### 2. 权限问题

**问题**：无法访问开发板或调试器

**解决方案**：
- 确认已安装 udev 规则
- 将用户添加到相关组：`sudo usermod -a -G dialout,plugdev $USER`
- 重新登录或重启系统使更改生效

### 3. 构建错误

**问题**：构建时出现编译错误

**解决方案**：
- 确认 CMake 和 Ninja 已正确安装
- 检查是否使用了正确的板子配置
- 验证 Zephyr 源码版本与 SDK 版本匹配

### 4. West 命令失败

**问题**：West 命令执行失败

**解决方案**：
- 确认 West 已正确安装：`west --version`
- 检查 Python 版本是否满足要求（3.6+）
- 在工作区根目录执行 West 命令

### 5. 找不到 Python 模块

**问题**：导入 Python 模块时报错

**解决方案**：
- 重新安装 Python 依赖：`pip3 install -r zephyr/scripts/requirements.txt`
- 考虑使用虚拟环境：`python3 -m venv .venv && source .venv/bin/activate`

### 6. Windows 特有问题

**问题**：在 Windows 上构建失败

**解决方案**：
- 使用短路径名称（避免空格和特殊字符）
- 考虑使用 WSL 替代原生 Windows 构建
- 确认 Visual Studio Build Tools 已正确安装

## 多版本管理

对于需要同时使用多个 Zephyr 版本的情况：

1. **使用不同工作目录**

```bash
# 创建特定版本的工作目录
west init -m https://github.com/zephyrproject-rtos/zephyr --mr v2.7.0 ~/zephyrproject-2.7
west init -m https://github.com/zephyrproject-rtos/zephyr --mr v2.9.1 ~/zephyrproject-2.9
```

2. **使用不同 SDK 版本**

为每个 Zephyr 版本安装匹配的 SDK 版本，并在使用时设置相应的环境变量。

## 总结

正确安装和配置 Zephyr 工具链是开发 Zephyr 应用程序的第一步。通过本文档的指导，您应该能够在不同操作系统上成功安装 Zephyr SDK 和相关工具，并准备好开始 Zephyr 开发。记住定期更新工具链以获取最新功能和修复，并参考常见问题解决部分来解决安装过程中可能遇到的问题。