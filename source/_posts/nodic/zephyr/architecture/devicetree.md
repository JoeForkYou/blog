---
title: Zephyr 设备树
tags: 
  - zephyr
  - devicetree
  - embedded
categories: 
  - nodic
  - zephyr
  - architecture
abbrlink: 15156
date: 2023-12-21 10:00:00
mermaid: true
---

# Zephyr 设备树

## 版本信息
- 版本：V1.0
- 更新时间：2025年03月20日 23:15

设备树（Device Tree）是 Zephyr RTOS 用于描述硬件配置的机制。它提供了一种统一的方式来描述系统硬件，使得硬件配置与软件代码分离，提高了代码的可移植性和可维护性。

## 设备树概述

### 什么是设备树

设备树是一种描述硬件的数据结构，它以树形结构表示硬件设备及其属性。在 Zephyr 中，设备树用于：

1. 描述硬件配置
2. 生成设备驱动的初始化代码
3. 配置中断和引脚复用
4. 定义内存映射

### 设备树的优势

1. **硬件描述与软件分离**：便于硬件配置的修改和维护
2. **跨平台兼容性**：同一套代码可以适用于不同的硬件平台
3. **动态配置**：支持运行时修改设备配置
4. **标准化**：采用行业标准的描述方式，提高可读性和互操作性

## 设备树语法

### 基本结构

设备树文件（.dts）的基本结构如下：

```dts
/dts-v1/;

/ {
    model = "Example Board";
    compatible = "example,board";

    chosen {
        zephyr,console = &uart0;
    };

    soc {
        uart0: uart@40002000 {
            compatible = "example,uart";
            reg = <0x40002000 0x1000>;
            status = "okay";
        };
    };
};
```

### 节点

节点表示系统中的设备或总线。每个节点可以包含属性和子节点。

```dts
soc {
    uart0: uart@40002000 {
        // 节点属性
    };
};
```

### 属性

属性用于描述节点的特征。常见的属性包括：

1. **compatible**：指定设备的兼容性字符串
2. **reg**：指定设备的寄存器地址范围
3. **status**：指定设备的启用状态
4. **interrupts**：指定设备的中断配置

```dts
uart0: uart@40002000 {
    compatible = "example,uart";
    reg = <0x40002000 0x1000>;
    status = "okay";
    interrupts = <5 0>;
};
```

### 标签和引用

使用标签（label）可以在设备树中引用其他节点：

```dts
uart0: uart@40002000 {
    // uart0 是一个标签
};

chosen {
    zephyr,console = &uart0;  // 使用 &uart0 引用标签
};
```

## Zephyr 中的设备树使用

### 设备树源文件

Zephyr 的设备树源文件主要包括：

1. **SoC 级别的 .dtsi 文件**：描述 SoC 的通用硬件配置
2. **板级的 .dts 文件**：描述特定开发板的硬件配置
3. **覆盖文件 (.overlay)**：用于修改或扩展现有的设备树配置

### 编译过程

设备树的编译过程如下：

1. 预处理：合并所有相关的 .dts 和 .dtsi 文件
2. 编译：将预处理后的文件编译成设备树二进制文件（.dtb）
3. 生成：基于 .dtb 文件生成 C 头文件，供应用程序使用

### 在代码中使用设备树

Zephyr 提供了一系列宏来访问设备树信息：

```c
#include <zephyr/devicetree.h>

// 检查节点是否启用
#if DT_NODE_HAS_STATUS(DT_NODELABEL(uart0), okay)
    // 使用 uart0
#endif

// 获取属性值
#define UART_ADDR DT_REG_ADDR(DT_NODELABEL(uart0))

// 遍历节点
#define UART_INIT(node_id) \
    uart_init(DT_LABEL(node_id), DT_REG_ADDR(node_id));

DT_FOREACH_STATUS_OKAY(example_uart, UART_INIT)
```

## 设备树叠加（Overlays）

设备树叠加允许在不修改原始 .dts 文件的情况下更改或添加设备树配置。

### 创建叠加文件

创建一个 .overlay 文件，例如 `my_board.overlay`：

```dts
/ {
    leds {
        compatible = "gpio-leds";
        led0: led_0 {
            gpios = <&gpio0 13 GPIO_ACTIVE_LOW>;
        };
    };
};

&uart0 {
    status = "disabled";
};
```

### 应用叠加文件

在构建时指定叠加文件：

```bash
west build -b my_board -- -DDTC_OVERLAY_FILE=my_board.overlay
```

## 最佳实践

1. **模块化设计**：将通用配置放在 .dtsi 文件中，特定配置放在 .dts 文件中
2. **使用标签**：为重要节点添加标签，方便引用
3. **保持简洁**：只包含必要的硬件描述，避免过度配置
4. **使用叠加文件**：通过叠加文件进行小的修改，而不是直接修改原始文件
5. **验证设备树**：使用 `dtc` 工具检查设备树的语法和结构
6. **文档化**：为自定义的设备树绑定编写文档，说明属性的用途和预期值

## 常见问题解决

1. **节点未启用**：检查节点的 `status` 属性是否设置为 "okay"
2. **属性未定义**：确保属性名称拼写正确，并检查是否在正确的节点中定义
3. **绑定问题**：确保使用了正确的 `compatible` 字符串，并且相应的驱动程序存在
4. **地址冲突**：检查 `reg` 属性，确保设备地址范围不重叠
5. **中断配置错误**：验证 `interrupts` 属性的格式是否正确，中断号是否有效

## 总结

设备树是 Zephyr RTOS 中描述硬件配置的强大工具。通过合理使用设备树，可以提高代码的可移植性和可维护性。深入理解设备树的概念和使用方法，对于开发高质量的 Zephyr 应用程序至关重要。