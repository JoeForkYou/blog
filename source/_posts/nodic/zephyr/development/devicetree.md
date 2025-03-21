---
abbrlink: 13
title: devicetree
date: '2025-03-21 20:49:56'
---
# Zephyr 设备树配置

本文档详细介绍了 Zephyr RTOS 的设备树配置系统，包括基本概念、节点和属性、绑定文件以及覆盖文件的使用方法。

## 设备树基础

### 概述

设备树是描述硬件配置的数据结构，它以层次化的方式表示系统中的硬件设备及其属性。

### 基本语法

```dts
/* 基本节点结构 */
node-name@unit-address {
    compatible = "vendor,device";
    reg = <address size>;
    status = "okay";
    property1 = <value1>;
    property2 = "string-value";
    #address-cells = <1>;
    #size-cells = <1>;

    child-node {
        ...
    };
};
```

### 常用属性

1. **compatible**：标识设备类型和驱动程序
2. **reg**：设备寄存器地址和大小
3. **status**：设备状态（"okay"、"disabled"）
4. **label**：设备标签
5. **interrupts**：中断配置
6. **clocks**：时钟配置
7. **gpios**：GPIO 配置

## 节点和属性

### 节点类型

1. **根节点 (/)**
```dts
/ {
    model = "Board Name";
    compatible = "vendor,board";
    #address-cells = <1>;
    #size-cells = <1>;
};
```

2. **总线节点**
```dts
i2c@40003000 {
    compatible = "vendor,i2c";
    reg = <0x40003000 0x1000>;
    #address-cells = <1>;
    #size-cells = <0>;
    status = "okay";
};
```

3. **设备节点**
```dts
sensor@48 {
    compatible = "vendor,sensor";
    reg = <0x48>;
    status = "okay";
    int-gpios = <&gpio0 24 GPIO_ACTIVE_LOW>;
};
```

### 属性类型

1. **整数属性**
```dts
reg = <0x40000000 0x1000>;
interrupts = <24 2>;
clock-frequency = <100000>;
```

2. **字符串属性**
```dts
compatible = "vendor,device";
label = "UART_0";
status = "okay";
```

3. **布尔属性**
```dts
hw-flow-control;
auto-enable;
```

4. **数组属性**
```dts
pwms = <&pwm0 0 1000000 PWM_POLARITY_NORMAL>,
       <&pwm1 1 2000000 PWM_POLARITY_INVERTED>;
```

5. **引用属性（phandle）**
```dts
clocks = <&clk_ctrl 0>;
dmas = <&dma0 0 &dma0 1>;
```

## 绑定文件

### 基本结构

```yaml
description: My Device Driver

compatible: "vendor,my-device"

include: base.yaml

properties:
    reg:
        required: true
        type: array
        description: Device registers

    interrupts:
        required: true
        type: array
        description: Device interrupts

    clock-frequency:
        required: false
        type: int
        default: 100000
        description: Clock frequency in Hz

child-binding:
    description: Child node properties
    properties:
        reg:
            required: true
            type: int
            description: Child address
```

### 属性定义

1. **基本属性**
```yaml
properties:
    my-property:
        type: int
        required: true
        description: Property description
```

2. **枚举属性**
```yaml
properties:
    operation-mode:
        type: string
        required: true
        enum:
            - "normal"
            - "fast"
            - "slow"
        description: Operating mode
```

3. **数组属性**
```yaml
properties:
    pin-config:
        type: array
        required: true
        description: Pin configuration
```

4. **GPIO 属性**
```yaml
properties:
    enable-gpios:
        type: phandle-array
        required: true
        description: Enable GPIO
```

### 约束定义

```yaml
properties:
    min-frequency:
        type: int
        required: true
        constraints:
            min: 1000
            max: 1000000

    supported-modes:
        type: array
        required: true
        constraints:
            min-items: 1
            max-items: 4
```

## 覆盖文件

### 基本用法

1. **应用程序覆盖文件 (app.overlay)**
```dts
/* 修改现有节点 */
&i2c0 {
    clock-frequency = <400000>;
    status = "okay";

    /* 添加新设备 */
    sensor@48 {
        compatible = "vendor,sensor";
        reg = <0x48>;
        status = "okay";
    };
};

/* 添加新节点 */
/ {
    my-device {
        compatible = "vendor,my-device";
        status = "okay";
    };
};
```

2. **板级覆盖文件 (board.overlay)**
```dts
/* 禁用设备 */
&uart1 {
    status = "disabled";
};

/* 修改 GPIO 配置 */
&gpio0 {
    ngpios = <16>;
};
```

### 常用修改

1. **修改时钟频率**
```dts
&i2c0 {
    clock-frequency = <400000>;
};
```

2. **配置 GPIO**
```dts
&gpio0 {
    ngpios = <32>;
    gpio-reserved-ranges = <10 2>;
};
```

3. **添加设备**
```dts
/ {
    leds {
        compatible = "gpio-leds";
        led0: led_0 {
            gpios = <&gpio0 13 GPIO_ACTIVE_LOW>;
            label = "Green LED";
        };
    };
};
```

4. **修改中断配置**
```dts
&uart0 {
    interrupts = <3 0>;
    interrupt-names = "rx";
};
```

## 使用设备树

### 在代码中访问

1. **检查节点状态**
```c
#define LED0_NODE DT_ALIAS(led0)

#if !DT_NODE_HAS_STATUS(LED0_NODE, okay)
#error "LED device not enabled"
#endif
```

2. **获取属性值**
```c
/* 获取寄存器地址 */
#define UART_ADDR DT_REG_ADDR(DT_NODELABEL(uart0))

/* 获取中断号 */
#define UART_IRQ DT_IRQN(DT_NODELABEL(uart0))

/* 获取时钟频率 */
#define I2C_FREQ DT_PROP(DT_NODELABEL(i2c0), clock_frequency)
```

3. **获取 GPIO 信息**
```c
/* 定义 GPIO 规范 */
static const struct gpio_dt_spec led =
    GPIO_DT_SPEC_GET(DT_NODELABEL(led0), gpios);

/* 使用 GPIO */
if (!device_is_ready(led.port)) {
    return -ENODEV;
}
gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE);
```

4. **遍历节点**
```c
/* 遍历所有 I2C 设备 */
#define FOREACH_I2C_DEVICE(node_id) \
    DT_FOREACH_CHILD(DT_NODELABEL(i2c0), node_id)

/* 使用宏 */
#define PROCESS_I2C_DEVICE(node_id) \
    do { \
        if (DT_NODE_HAS_STATUS(node_id, okay)) { \
            /* 处理设备 */ \
        } \
    } while (0)

FOREACH_I2C_DEVICE(PROCESS_I2C_DEVICE);
```

### 编译时检查

1. **节点存在性检查**
```c
BUILD_ASSERT(DT_NODE_EXISTS(DT_NODELABEL(i2c0)),
            "I2C0 node not found");
```

2. **属性检查**
```c
BUILD_ASSERT(DT_NODE_HAS_PROP(DT_NODELABEL(uart0), current_speed),
            "UART speed not configured");
```

3. **兼容性检查**
```c
BUILD_ASSERT(DT_HAS_COMPAT_STATUS_OKAY(vendor_device),
            "Required device not found");
```

## 调试技巧

### 1. 查看处理后的设备树

```bash
# 生成预处理后的设备树
west build -t devicetree_preprocessed

# 查看最终的设备树
west build -t devicetree_generated
```

### 2. 验证设备树语法

```bash
# 使用 dtc 工具验证
dtc -I dts -O dts -o /dev/null build/zephyr/zephyr.dts
```

### 3. 检查节点路径

```c
/* 在代码中打印节点路径 */
LOG_INF("Node path: %s", DT_NODE_PATH(DT_NODELABEL(my_device)));
```

### 4. 调试绑定问题

```bash
# 查看绑定警告
west build -t devicetree_generated -- -DDTC_WARN_UNDEFINED_BINDING=y
```

## 最佳实践

### 1. 设备树组织

- 使用逻辑分组
- 保持结构清晰
- 适当使用标签
- 文档化配置

### 2. 属性命名

- 使用标准属性名
- 遵循命名约定
- 清晰描述用途
- 添加适当注释

### 3. 覆盖文件使用

- 最小化修改
- 保持兼容性
- 文档化更改
- 验证修改效果

### 4. 代码集成

- 使用设备树宏
- 验证配置
- 错误处理
- 维护性考虑

## 常见问题

### 1. 节点不可见

**问题**：无法访问设备树节点

**解决方案**：
- 检查节点状态
- 验证路径正确
- 确认编译选项
- 检查依赖关系

### 2. 属性访问错误

**问题**：无法获取属性值

**解决方案**：
- 检查属性类型
- 验证属性存在
- 确认访问方法
- 检查绑定文件

### 3. 绑定问题

**问题**：绑定文件不生效

**解决方案**：
- 检查文件位置
- 验证语法正确
- 确认兼容性字符串
- 更新构建系统

### 4. 覆盖冲突

**问题**：覆盖文件冲突

**解决方案**：
- 检查优先级
- 解决冲突
- 统一配置
- 文档化选择

## 总结

设备树是 Zephyr RTOS 中描述硬件配置的核心机制。通过正确使用节点、属性、绑定文件和覆盖文件，可以灵活地配置和管理硬件资源。本文档提供了详细的指导和实例，帮助开发者更好地理解和使用设备树系统。