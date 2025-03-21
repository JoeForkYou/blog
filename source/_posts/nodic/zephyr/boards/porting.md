---
title: 添加新板子到 Zephyr
tags: zephyr
categories: zephyr
abbrlink: 15162
date: 2025-03-21 00:45:18
mermaid: true
---

# 添加新板子到 Zephyr

## 版本信息
- 版本：V1.0
- 更新时间：2025年03月21日 00:45

本文档详细介绍了如何将新的开发板添加到 Zephyr RTOS 中，包括移植流程、设备树配置、驱动适配和调试方法。

## 移植流程概述

1. **准备工作**
   - 收集硬件文档
   - 确定 SoC 和 CPU 架构
   - 检查现有的类似板子支持

2. **创建板级支持包 (BSP)**
   - 添加新的板子目录
   - 创建基本配置文件

3. **配置设备树**
   - 定义板子特定的设备树
   - 配置引脚复用和外设

4. **适配驱动程序**
   - 确认所需驱动的可用性
   - 修改或创建新的驱动程序

5. **实现板级初始化**
   - 配置时钟系统
   - 设置中断控制器
   - 初始化关键外设

6. **配置构建系统**
   - 更新 CMakeLists.txt
   - 设置 Kconfig 选项

7. **测试和调试**
   - 编译基本示例
   - 使用调试工具验证功能

8. **文档和示例**
   - 编写板级文档
   - 创建板子特定的示例代码

## 详细步骤

### 1. 准备工作

- 收集硬件文档
  - 处理器数据手册
  - 开发板原理图
  - 引脚分配表

- 确定 SoC 和 CPU 架构
  ```bash
  # 检查 Zephyr 支持的架构
  ls zephyr/arch
  
  # 检查 SoC 支持
  ls zephyr/soc
  ```

- 检查类似板子
  ```bash
  # 查找类似的板子支持
  find zephyr/boards -name "*_defconfig" | grep -i <your_soc_family>
  ```

### 2. 创建板级支持包

- 添加新的板子目录
  ```bash
  mkdir -p zephyr/boards/<arch>/<your_board_name>
  cd zephyr/boards/<arch>/<your_board_name>
  ```

- 创建基本配置文件
  ```bash
  touch <your_board_name>.dts
  touch <your_board_name>_defconfig
  touch board.cmake
  touch Kconfig.board
  touch Kconfig.defconfig
  ```

### 3. 配置设备树

- 编辑 `<your_board_name>.dts`
  ```dts
  /dts-v1/;
  #include <your_soc.dtsi>

  / {
      model = "Your Board Name";
      compatible = "vendor,your-board-name";

      chosen {
          zephyr,console = &uart0;
          zephyr,shell-uart = &uart0;
          zephyr,sram = &sram0;
          zephyr,flash = &flash0;
      };

      /* 定义 LED */
      leds {
          compatible = "gpio-leds";
          led0: led_0 {
              gpios = <&gpio0 13 GPIO_ACTIVE_LOW>;
              label = "Green LED 0";
          };
      };

      /* 定义按钮 */
      buttons {
          compatible = "gpio-keys";
          button0: button_0 {
              gpios = <&gpio0 11 GPIO_ACTIVE_LOW>;
              label = "Push button switch 0";
          };
      };
  };

  &uart0 {
      status = "okay";
      current-speed = <115200>;
      tx-pin = <6>;
      rx-pin = <8>;
  };

  &i2c0 {
      status = "okay";
      sda-pin = <26>;
      scl-pin = <27>;
  };

  /* 添加其他外设配置 */
  ```

### 4. 适配驱动程序

- 检查驱动可用性
  ```bash
  ls zephyr/drivers
  ```

- 修改驱动程序（如果需要）
  ```c
  // 示例：修改 UART 驱动以支持新的硬件特性
  // 文件：zephyr/drivers/serial/uart_your_soc.c
  
  static int uart_your_soc_init(const struct device *dev)
  {
      // 实现初始化逻辑
  }

  static const struct uart_driver_api uart_your_soc_driver_api = {
      .poll_in = uart_your_soc_poll_in,
      .poll_out = uart_your_soc_poll_out,
      .err_check = uart_your_soc_err_check,
      // 添加其他必要的函数
  };

  // 定义设备
  #define UART_YOUR_SOC_INIT(n)                                            \
      static struct uart_your_soc_data uart_your_soc_data_##n = {          \
          // 初始化数据                                                     \
      };                                                                   \
                                                                           \
      static const struct uart_your_soc_config uart_your_soc_config_##n = {\
          // 配置数据                                                       \
      };                                                                   \
                                                                           \
      DEVICE_DT_INST_DEFINE(n,                                             \
                            uart_your_soc_init,                            \
                            NULL,                                          \
                            &uart_your_soc_data_##n,                       \
                            &uart_your_soc_config_##n,                     \
                            PRE_KERNEL_1,                                  \
                            CONFIG_SERIAL_INIT_PRIORITY,                   \
                            &uart_your_soc_driver_api);

  DT_INST_FOREACH_STATUS_OKAY(UART_YOUR_SOC_INIT)
  ```

### 5. 实现板级初始化

- 创建 `board.c` 文件
  ```c
  #include <zephyr/init.h>
  #include <zephyr/drivers/gpio.h>

  static int board_your_board_init(void)
  {
      // 配置时钟
      // 设置中断控制器
      // 初始化关键外设
      return 0;
  }

  SYS_INIT(board_your_board_init, PRE_KERNEL_1, CONFIG_KERNEL_INIT_PRIORITY_DEFAULT);
  ```

### 6. 配置构建系统

- 编辑 `CMakeLists.txt`
  ```cmake
  # 添加源文件
  zephyr_library_sources(board.c)

  # 包含头文件目录
  zephyr_library_include_directories(${ZEPHYR_BASE}/drivers)
  ```

- 配置 `Kconfig.board`
  ```
  config BOARD_YOUR_BOARD_NAME
      bool "Your Board Name"
      depends on SOC_SERIES_YOUR_SOC_SERIES
  ```

- 配置 `Kconfig.defconfig`
  ```
  if BOARD_YOUR_BOARD_NAME

  config BOARD
      default "your_board_name"

  if GPIO

  config GPIO_AS_PINRESET
      default y

  endif # GPIO

  endif # BOARD_YOUR_BOARD_NAME
  ```

### 7. 测试和调试

- 编译基本示例
  ```bash
  west build -b your_board_name samples/basic/blinky
  ```

- 使用调试工具
  ```bash
  west debug
  ```

### 8. 文档和示例

- 创建 `doc/board.rst`
  ```rst
  .. _your_board_name:

  Your Board Name
  ###############

  Overview
  ********

  The Your Board Name is a development board based on the YOUR_SOC.

  Hardware
  ********

  - YOUR_SOC ARM Cortex-M4 processor at 64 MHz
  - 512 KiB flash memory
  - 64 KiB RAM
  - GPIO
  - UART
  - I2C
  - SPI

  Supported Features
  ==================

  The your_board_name board configuration supports the following hardware features:

  +-----------+------------+-------------------------------------+
  | Interface | Controller | Driver/Component                    |
  +===========+============+=====================================+
  | NVIC      | on-chip    | nested vector interrupt controller  |
  +-----------+------------+-------------------------------------+
  | UART      | on-chip    | serial port-polling;                |
  |           |            | serial port-interrupt               |
  +-----------+------------+-------------------------------------+
  | GPIO      | on-chip    | gpio                                |
  +-----------+------------+-------------------------------------+
  | I2C       | on-chip    | i2c                                 |
  +-----------+------------+-------------------------------------+
  | SPI       | on-chip    | spi                                 |
  +-----------+------------+-------------------------------------+

  Connections and IOs
  ===================

  LED
  ---

  * LED0 (green) = P0.13

  Push buttons
  ------------

  * BUTTON0 = P0.11

  Programming and Debugging
  *************************

  Flashing
  ========

  Here is an example for the :ref:`hello_world` application.

  .. zephyr-app-commands::
     :zephyr-app: samples/hello_world
     :board: your_board_name
     :goals: flash

  Debugging
  =========

  Refer to the :ref:`cmake_debugging` guide for information about debugging Zephyr applications.

  References
  **********

  .. target-notes::

  .. _Your Board Name website: https://www.example.com/your_board
  .. _YOUR_SOC Datasheet: https://www.example.com/your_soc_datasheet.pdf
  ```

## 调试方法

1. **使用 OpenOCD**
   ```bash
   # 创建 OpenOCD 配置文件
   touch your_board.cfg
   
   # 编辑配置文件
   source [find interface/jlink.cfg]
   transport select swd
   source [find target/your_soc.cfg]
   
   # 启动 OpenOCD
   openocd -f your_board.cfg
   ```

2. **使用 GDB**
   ```bash
   # 启动 GDB
   arm-zephyr-eabi-gdb
   
   # 在 GDB 中连接到目标
   (gdb) target remote localhost:3333
   (gdb) monitor reset halt
   (gdb) load
   (gdb) continue
   ```

3. **使用 Segger J-Link**
   ```bash
   # 启动 J-Link GDB 服务器
   JLinkGDBServer -device YOUR_SOC -if SWD -speed 4000
   
   # 在另一个终端中启动 GDB
   arm-zephyr-eabi-gdb
   (gdb) target remote localhost:2331
   ```

4. **使用 pyOCD**
   ```bash
   # 安装 pyOCD
   pip install pyocd
   
   # 启动 pyOCD GDB 服务器
   pyocd gdbserver -t YOUR_SOC
   
   # 使用 GDB 连接
   arm-zephyr-eabi-gdb
   (gdb) target remote localhost:3333
   ```

## 最佳实践

1. **复用现有代码**
   - 尽可能使用现有的 SoC 和驱动程序代码
   - 只添加必要的板级特定代码

2. **遵循命名约定**
   - 使用一致的命名方式，如 `<vendor>_<board>`
   - 在设备树中使用描述性的标签

3. **文档化**
   - 详细记录硬件特性和支持的功能
   - 提供清晰的配置和使用说明

4. **测试覆盖**
   - 为所有支持的功能编写测试用例
   - 使用持续集成确保兼容性

5. **保持更新**
   - 跟踪上游 Zephyr 的变化
   - 定期更新板级支持包

## 常见问题

1. **编译错误**
   - 检查 Kconfig 和 CMake 配置
   - 验证所有必要的驱动程序都已启用

2. **设备树错误**
   - 确保设备树语法正确
   - 验证所有必要的节点都已定义

3. **驱动程序不工作**
   - 检查设备树配置
   - 验证驱动程序初始化顺序

4. **调试连接失败**
   - 检查硬件连接
   - 验证调试器配置

5. **性能问题**
   - 检查时钟配置
   - 优化中断和 DMA 设置

## 总结

将新的开发板添加到 Zephyr RTOS 需要详细了解硬件规格、熟悉 Zephyr 的架构和开发流程。通过遵循本文档中的步骤和最佳实践，您可以成功地将新的硬件平台集成到 Zephyr 生态系统中。记住，良好的文档和全面的测试对于确保您的贡献被社区接受和长期维护至关重要。