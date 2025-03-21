---
title: Zephyr RTOS 快速入门指南
tags: 
  - zephyr
  - RTOS
  - embedded
categories: 
  - nodic
  - zephyr
abbrlink: 47
date: 2023-12-21 10:00:00
---
# Zephyr RTOS 快速入门

本章节将帮助您快速上手 Zephyr RTOS 开发。

## 环境搭建

### 系统要求

Zephyr 开发环境支持以下操作系统：
- Windows 10 或更新版本（推荐使用 WSL2）
- macOS 10.15 或更新版本
- Ubuntu 18.04 或更新版本
- 其他 Linux 发行版

### 安装工具链

1. 安装依赖工具
   ```bash
   # Ubuntu/Debian
   sudo apt install --no-install-recommends git cmake ninja-build gperf
   sudo apt install ccache dfu-util device-tree-compiler wget
   sudo apt install python3-dev python3-pip python3-setuptools python3-tk python3-wheel xz-utils file
   sudo apt install make gcc gcc-multilib g++-multilib libsdl2-dev
   ```

2. 安装 West 工具
   ```bash
   pip3 install --user -U west
   echo 'export PATH=~/.local/bin:"$PATH"' >> ~/.bashrc
   source ~/.bashrc
   ```

3. 获取 Zephyr 源码
   ```bash
   west init ~/zephyrproject
   cd ~/zephyrproject
   west update
   ```

4. 安装 Python 依赖
   ```bash
   pip3 install --user -r ~/zephyrproject/zephyr/scripts/requirements.txt
   ```

### SDK 安装

1. 下载 Zephyr SDK
   ```bash
   cd ~
   wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/zephyr-sdk-0.16.1_linux-x86_64.tar.gz
   ```

2. 安装 SDK
   ```bash
   cd ~
   tar xvf zephyr-sdk-0.16.1_linux-x86_64.tar.gz
   cd zephyr-sdk-0.16.1
   ./setup.sh
   ```

## Hello World 示例

### 创建项目

1. 创建应用目录
   ```bash
   cd ~/zephyrproject/zephyr
   west build -b qemu_x86 samples/hello_world
   ```

2. 构建并运行
   ```bash
   west build -t run
   # 或使用 QEMU 直接运行
   west run
   ```

### 示例说明

Hello World 示例的源码位于 `samples/hello_world/src/main.c`：

```c
#include <zephyr/kernel.h>

void main(void)
{
    printk("Hello World! %s\n", CONFIG_BOARD);
}
```

这个简单的示例展示了：
- 基本的应用程序结构
- 内核头文件的使用
- 打印输出的方法
- 配置宏的使用

## 基本概念

### 项目结构

Zephyr 项目典型结构：
```
app/
├── CMakeLists.txt          # 构建系统配置
├── prj.conf               # 项目配置文件
└── src/
    └── main.c            # 应用程序源码
```

### 重要工具

1. West 命令行工具
   - 项目管理
   - 构建系统
   - 调试工具
   - 固件更新

2. CMake 构建系统
   - 跨平台构建
   - 依赖管理
   - 配置系统

3. Kconfig 配置系统
   - 内核配置
   - 驱动配置
   - 应用配置

### 开发流程

1. 创建应用
   ```bash
   west init -m https://github.com/zephyrproject-rtos/zephyr --mr v2.9.1 zephyrproject
   cd zephyrproject
   west update
   ```

2. 配置项目
   - 修改 prj.conf 添加需要的功能
   - 配置目标板卡
   - 设置应用程序选项

3. 构建运行
   ```bash
   west build -b 目标板卡 应用目录
   west flash
   ```

4. 调试开发
   - 使用 west debug 启动调试
   - 查看系统日志
   - 使用 SEGGER SystemView 分析

## 下一步

完成快速入门后，建议：
1. 尝试更多[示例代码](/nodic/zephyr/examples/README)
2. 学习[系统架构](/nodic/zephyr/architecture/README)
3. 了解[核心模块](/nodic/zephyr/core/README)使用
4. 参考[开发指南](/nodic/zephyr/development/README)开始实际项目