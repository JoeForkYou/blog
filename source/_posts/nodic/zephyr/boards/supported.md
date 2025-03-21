---
title: Zephyr 支持的开发板
tags: zephyr
categories: zephyr
abbrlink: 15163
date: 2025-03-21 01:00:42
mermaid: true
---

# Zephyr 支持的开发板

## 版本信息
- 版本：V1.0
- 更新时间：2025年03月21日 01:00

本文档详细介绍了 Zephyr RTOS 支持的主要开发板、它们的硬件特性对比以及开发板选择指南。

## 主要开发板列表

### ARM Cortex-M 系列

#### Nordic Semiconductor

1. **nRF52840 DK (nrf52840dk_nrf52840)**
   - 处理器: ARM Cortex-M4F @ 64 MHz
   - 闪存: 1 MB
   - RAM: 256 KB
   - 特性: BLE 5.0, NFC, USB, 多种无线协议

2. **nRF52 DK (nrf52dk_nrf52832)**
   - 处理器: ARM Cortex-M4F @ 64 MHz
   - 闪存: 512 KB
   - RAM: 64 KB
   - 特性: BLE 5.0, NFC

3. **nRF5340 DK (nrf5340dk_nrf5340)**
   - 处理器: ARM Cortex-M33 (应用核 + 网络核)
   - 闪存: 1 MB (应用核) + 256 KB (网络核)
   - RAM: 512 KB (应用核) + 64 KB (网络核)
   - 特性: BLE 5.2, NFC, 安全启动

#### STMicroelectronics

1. **STM32F4 Discovery (stm32f4_disco)**
   - 处理器: ARM Cortex-M4F @ 168 MHz
   - 闪存: 1 MB
   - RAM: 192 KB
   - 特性: 加速度计, 音频 DAC

2. **STM32F746G Discovery (stm32f746g_disco)**
   - 处理器: ARM Cortex-M7 @ 216 MHz
   - 闪存: 1 MB
   - RAM: 320 KB
   - 特性: LCD 触摸屏, 摄像头接口, 以太网

3. **STM32 Nucleo-F401RE (nucleo_f401re)**
   - 处理器: ARM Cortex-M4F @ 84 MHz
   - 闪存: 512 KB
   - RAM: 96 KB
   - 特性: Arduino 兼容接口

#### NXP

1. **FRDM-K64F (frdm_k64f)**
   - 处理器: ARM Cortex-M4F @ 120 MHz
   - 闪存: 1 MB
   - RAM: 256 KB
   - 特性: 以太网, SD 卡插槽, 加速度计

2. **MIMXRT1050-EVK (mimxrt1050_evk)**
   - 处理器: ARM Cortex-M7 @ 600 MHz
   - 闪存: 外部 QSPI 闪存
   - RAM: 512 KB SRAM + 外部 SDRAM
   - 特性: LCD 接口, 以太网, USB

#### Espressif

1. **ESP32 DevKitC (esp32)**
   - 处理器: Xtensa LX6 双核 @ 240 MHz
   - 闪存: 外部 4 MB
   - RAM: 520 KB SRAM
   - 特性: WiFi, BLE, 丰富的外设

2. **ESP32-C3 DevKitM (esp32c3_devkitm)**
   - 处理器: RISC-V 单核 @ 160 MHz
   - 闪存: 外部 4 MB
   - RAM: 400 KB SRAM
   - 特性: WiFi, BLE 5.0

### x86 系列

1. **UP Squared (up_squared)**
   - 处理器: Intel Apollo Lake
   - 内存: 取决于配置
   - 特性: 完整 x86 平台, PCIe, SATA

2. **MinnowBoard (minnowboard)**
   - 处理器: Intel Atom E3800 系列
   - 内存: 取决于配置
   - 特性: UEFI, PCIe, SATA

### RISC-V 系列

1. **SiFive HiFive1 Rev B (hifive1_revb)**
   - 处理器: SiFive Freedom E310 @ 320 MHz
   - 闪存: 16 MB
   - RAM: 16 KB
   - 特性: Arduino 兼容接口

2. **Microchip PolarFire SoC Icicle Kit (mpfs_icicle)**
   - 处理器: SiFive U54 四核 RISC-V @ 667 MHz
   - 内存: 取决于配置
   - 特性: FPGA, PCIe, GbE

## 硬件特性对比

### 处理能力

| 开发板 | 处理器 | 频率 | 性能指标 (CoreMark) |
|-------|-------|------|-------------------|
| nRF52840 DK | Cortex-M4F | 64 MHz | ~200 |
| STM32F746G Discovery | Cortex-M7 | 216 MHz | ~1000 |
| MIMXRT1050-EVK | Cortex-M7 | 600 MHz | ~3000 |
| ESP32 DevKitC | Xtensa LX6 双核 | 240 MHz | ~1500 |
| UP Squared | Intel Apollo Lake | 1.8 GHz | ~10000 |

### 内存资源

| 开发板 | 闪存 | RAM | 外部存储 |
|-------|------|-----|---------|
| nRF52840 DK | 1 MB | 256 KB | 可选外部 QSPI 闪存 |
| STM32F746G Discovery | 1 MB | 320 KB | SDRAM, QSPI 闪存 |
| FRDM-K64F | 1 MB | 256 KB | SD 卡插槽 |
| ESP32 DevKitC | 外部 4 MB | 520 KB | 支持 SD 卡 |
| HiFive1 Rev B | 16 MB | 16 KB | 无 |

### 通信接口

| 开发板 | UART | SPI | I2C | USB | 以太网 | 无线 |
|-------|------|-----|-----|-----|-------|------|
| nRF52840 DK | ✓ | ✓ | ✓ | ✓ | - | BLE 5.0, 802.15.4 |
| STM32F746G Discovery | ✓ | ✓ | ✓ | ✓ | ✓ | - |
| FRDM-K64F | ✓ | ✓ | ✓ | ✓ | ✓ | - |
| ESP32 DevKitC | ✓ | ✓ | ✓ | - | - | WiFi, BLE |
| UP Squared | ✓ | ✓ | ✓ | ✓ | ✓ | 可选 WiFi |

### 外设支持

| 开发板 | GPIO | ADC | DAC | PWM | 显示接口 | 摄像头 | 其他传感器 |
|-------|------|-----|-----|-----|---------|-------|-----------|
| nRF52840 DK | ✓ | ✓ | - | ✓ | - | - | 温度 |
| STM32F746G Discovery | ✓ | ✓ | ✓ | ✓ | LCD 触摸屏 | ✓ | 加速度计 |
| FRDM-K64F | ✓ | ✓ | ✓ | ✓ | - | - | 加速度计, 磁力计 |
| ESP32 DevKitC | ✓ | ✓ | ✓ | ✓ | 可扩展 | 可扩展 | 温度 |
| UP Squared | ✓ | - | - | - | HDMI, DP | - | - |

### 调试支持

| 开发板 | 调试接口 | 调试工具 | 串口 | JTAG/SWD | 追踪 |
|-------|---------|---------|------|----------|------|
| nRF52840 DK | J-Link OB | J-Link | ✓ | ✓ | ✓ |
| STM32F746G Discovery | ST-LINK/V2-1 | ST-LINK | ✓ | ✓ | ✓ |
| FRDM-K64F | OpenSDA | CMSIS-DAP | ✓ | ✓ | - |
| ESP32 DevKitC | UART | ESP-Prog | ✓ | ✓ | - |
| HiFive1 Rev B | FTDI | OpenOCD | ✓ | ✓ | - |

## 开发板选择指南

### 根据应用场景选择

1. **IoT 和无线应用**
   - **推荐**: nRF52840 DK, ESP32 DevKitC
   - **优势**: 集成无线功能, 低功耗, 丰富的协议栈

2. **实时控制系统**
   - **推荐**: STM32F746G Discovery, MIMXRT1050-EVK
   - **优势**: 高性能处理器, 丰富的外设, 实时响应

3. **网关和边缘计算**
   - **推荐**: UP Squared, ESP32 DevKitC
   - **优势**: 高性能, 多种通信接口, 丰富的存储选项

4. **入门和教育**
   - **推荐**: STM32 Nucleo-F401RE, FRDM-K64F
   - **优势**: 价格适中, 资源丰富, 社区支持好

5. **工业控制**
   - **推荐**: MIMXRT1050-EVK, STM32F746G Discovery
   - **优势**: 高可靠性, 丰富的接口, 实时性能

### 根据开发经验选择

1. **初学者**
   - **推荐**: STM32 Nucleo 系列, FRDM-K64F
   - **原因**: 文档丰富, 示例代码多, 社区支持好

2. **有经验的开发者**
   - **推荐**: nRF52840 DK, ESP32 DevKitC
   - **原因**: 功能强大, 灵活性高, 可实现复杂应用

3. **专业嵌入式工程师**
   - **推荐**: MIMXRT1050-EVK, UP Squared
   - **原因**: 高性能, 丰富的配置选项, 适合复杂系统

### 根据项目约束选择

1. **成本敏感**
   - **推荐**: STM32 Nucleo 系列, ESP32-C3 DevKitM
   - **优势**: 价格适中, 性价比高

2. **功耗敏感**
   - **推荐**: nRF52840 DK, nRF5340 DK
   - **优势**: 低功耗设计, 多种省电模式

3. **尺寸受限**
   - **推荐**: ESP32-C3 DevKitM, STM32 Nucleo-32 系列
   - **优势**: 小尺寸, 高集成度

4. **高性能需求**
   - **推荐**: MIMXRT1050-EVK, UP Squared
   - **优势**: 高频率处理器, 大内存

## 开发板获取方式

1. **官方渠道**
   - 制造商官网
   - 授权分销商

2. **电子元件分销商**
   - Mouser Electronics
   - Digi-Key
   - Element14 / Newark
   - RS Components

3. **在线零售商**
   - Amazon
   - Adafruit
   - SparkFun
   - Seeed Studio

## 开始使用

1. **环境准备**
   ```bash
   # 安装 Zephyr SDK
   wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/zephyr-sdk-0.16.1_linux-x86_64.tar.gz
   tar xvf zephyr-sdk-0.16.1_linux-x86_64.tar.gz
   cd zephyr-sdk-0.16.1
   ./setup.sh
   
   # 获取 Zephyr 源码
   west init -m https://github.com/zephyrproject-rtos/zephyr --mr v2.9.1 zephyrproject
   cd zephyrproject
   west update
   ```

2. **构建示例**
   ```bash
   # 针对特定开发板构建 Hello World 示例
   cd zephyrproject/zephyr
   west build -b nrf52840dk_nrf52840 samples/hello_world
   
   # 烧录到开发板
   west flash
   ```

3. **调试**
   ```bash
   # 启动调试会话
   west debug
   ```

## 总结

Zephyr RTOS 支持广泛的硬件平台，从低功耗微控制器到高性能处理器。选择合适的开发板对于项目成功至关重要。在选择时，应考虑：

1. 应用需求（处理能力、内存、通信接口）
2. 开发经验和可用资源
3. 项目约束（成本、功耗、尺寸）
4. 工具链和调试支持

通过本文档提供的信息，您可以更好地了解各种开发板的特点，并为您的项目选择最合适的硬件平台。