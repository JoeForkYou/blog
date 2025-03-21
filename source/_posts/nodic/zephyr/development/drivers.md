---
abbrlink: 15
title: drivers
date: '2025-03-21 20:49:56'
---
# Zephyr 驱动开发

本文档详细介绍了 Zephyr RTOS 的驱动开发过程，包括驱动模型、API 设计、设备树绑定以及实际示例。

## 驱动模型

### 基本概念

Zephyr 的驱动模型基于以下核心概念：

1. **设备对象**：表示硬件设备的软件抽象
2. **驱动 API**：定义设备操作接口
3. **设备树**：描述硬件配置
4. **设备实例**：运行时的设备表示

### 驱动结构

典型的驱动程序包含以下部分：

```c
/* 驱动 API 定义 */
struct my_driver_api {
    int (*init)(const struct device *dev);
    int (*read)(const struct device *dev, void *buf, size_t len);
    int (*write)(const struct device *dev, const void *buf, size_t len);
};

/* 设备数据结构 */
struct my_driver_data {
    uint32_t config_value;
    uint8_t status;
    struct k_sem lock;
};

/* 设备配置结构 */
struct my_driver_config {
    uint32_t base_addr;
    uint32_t irq_num;
    uint32_t clock_freq;
};

/* 驱动实例化 */
#define MY_DRIVER_INIT(n)                                            \
    static struct my_driver_data my_driver_data_##n = {             \
        .status = 0,                                                \
    };                                                              \
                                                                    \
    static const struct my_driver_config my_driver_config_##n = {   \
        .base_addr = DT_INST_REG_ADDR(n),                          \
        .irq_num = DT_INST_IRQN(n),                                \
        .clock_freq = DT_INST_PROP(n, clock_frequency),            \
    };                                                              \
                                                                    \
    DEVICE_DT_INST_DEFINE(n,                                       \
                         my_driver_init,                            \
                         NULL,                                      \
                         &my_driver_data_##n,                       \
                         &my_driver_config_##n,                     \
                         POST_KERNEL,                               \
                         CONFIG_KERNEL_INIT_PRIORITY_DEVICE,        \
                         &my_driver_api);

/* 为每个实例生成代码 */
DT_INST_FOREACH_STATUS_OKAY(MY_DRIVER_INIT)
```

## 驱动 API

### API 设计原则

1. **一致性**
   - 遵循 Zephyr API 命名约定
   - 保持参数顺序一致
   - 使用统一的错误码

2. **可重入性**
   - 保护共享资源
   - 避免全局变量
   - 使用线程安全机制

3. **错误处理**
   - 返回有意义的错误码
   - 提供详细的错误信息
   - 实现错误恢复机制

### API 实现示例

```c
/* 驱动初始化函数 */
static int my_driver_init(const struct device *dev)
{
    struct my_driver_data *data = dev->data;
    const struct my_driver_config *config = dev->config;

    /* 初始化同步原语 */
    k_sem_init(&data->lock, 1, 1);

    /* 配置硬件 */
    /* ... */

    return 0;
}

/* 读取函数 */
static int my_driver_read(const struct device *dev,
                         void *buf, size_t len)
{
    struct my_driver_data *data = dev->data;
    const struct my_driver_config *config = dev->config;
    int ret = 0;

    /* 获取锁 */
    k_sem_take(&data->lock, K_FOREVER);

    /* 执行读取操作 */
    /* ... */

    /* 释放锁 */
    k_sem_give(&data->lock);

    return ret;
}

/* 写入函数 */
static int my_driver_write(const struct device *dev,
                          const void *buf, size_t len)
{
    struct my_driver_data *data = dev->data;
    const struct my_driver_config *config = dev->config;
    int ret = 0;

    /* 获取锁 */
    k_sem_take(&data->lock, K_FOREVER);

    /* 执行写入操作 */
    /* ... */

    /* 释放锁 */
    k_sem_give(&data->lock);

    return ret;
}

/* API 结构体 */
static const struct my_driver_api my_driver_api = {
    .init = my_driver_init,
    .read = my_driver_read,
    .write = my_driver_write,
};
```

## 设备树绑定

### 绑定文件

设备树绑定文件 (*.yaml) 定义了设备的属性：

```yaml
description: My Device Driver

compatible: "vendor,my-device"

include: base.yaml

properties:
    reg:
        required: true
        type: array
        description: Device registers location and length

    interrupts:
        required: true
        type: array
        description: Device interrupt lines

    clock-frequency:
        required: true
        type: int
        description: Device clock frequency in Hz

    enable-gpios:
        type: phandle-array
        description: GPIO for device enable control

child-binding:
    description: Child node properties

    properties:
        reg:
            required: true
            type: int
            description: Child device address
```

### 设备树节点

设备树中的设备节点：

```dts
my_device: my-device@40000000 {
    compatible = "vendor,my-device";
    reg = <0x40000000 0x1000>;
    interrupts = <10 2>;
    clock-frequency = <16000000>;
    enable-gpios = <&gpio0 15 GPIO_ACTIVE_HIGH>;
    status = "okay";

    child@0 {
        reg = <0>;
        status = "okay";
    };
};
```

## 驱动示例

### 1. GPIO 驱动

```c
#include <zephyr/drivers/gpio.h>

/* 驱动数据 */
struct gpio_driver_data {
    struct gpio_driver_config config;
    uint32_t pin_state;
    struct k_sem lock;
};

/* 驱动配置 */
struct gpio_driver_config {
    uint32_t base_addr;
    uint32_t port_num;
};

/* GPIO 配置函数 */
static int gpio_driver_configure(const struct device *dev,
                               gpio_pin_t pin,
                               gpio_flags_t flags)
{
    struct gpio_driver_data *data = dev->data;
    const struct gpio_driver_config *config = dev->config;
    int ret = 0;

    if (pin >= 32) {
        return -EINVAL;
    }

    k_sem_take(&data->lock, K_FOREVER);

    /* 配置 GPIO */
    if (flags & GPIO_OUTPUT) {
        /* 配置为输出 */
    } else if (flags & GPIO_INPUT) {
        /* 配置为输入 */
    }

    k_sem_give(&data->lock);

    return ret;
}

/* GPIO 获取函数 */
static int gpio_driver_get(const struct device *dev,
                          gpio_pin_t pin,
                          gpio_flags_t *flags)
{
    struct gpio_driver_data *data = dev->data;
    const struct gpio_driver_config *config = dev->config;
    int ret = 0;

    if (pin >= 32) {
        return -EINVAL;
    }

    k_sem_take(&data->lock, K_FOREVER);

    /* 读取 GPIO 状态 */
    *flags = (data->pin_state & BIT(pin)) ? GPIO_ACTIVE_HIGH : GPIO_ACTIVE_LOW;

    k_sem_give(&data->lock);

    return ret;
}

/* GPIO 设置函数 */
static int gpio_driver_set(const struct device *dev,
                          gpio_pin_t pin,
                          int value)
{
    struct gpio_driver_data *data = dev->data;
    const struct gpio_driver_config *config = dev->config;
    int ret = 0;

    if (pin >= 32) {
        return -EINVAL;
    }

    k_sem_take(&data->lock, K_FOREVER);

    /* 设置 GPIO 状态 */
    if (value) {
        data->pin_state |= BIT(pin);
    } else {
        data->pin_state &= ~BIT(pin);
    }

    k_sem_give(&data->lock);

    return ret;
}

/* GPIO 驱动 API */
static const struct gpio_driver_api gpio_driver_api = {
    .pin_configure = gpio_driver_configure,
    .pin_get = gpio_driver_get,
    .pin_set = gpio_driver_set,
};

/* 驱动初始化宏 */
#define GPIO_DRIVER_INIT(n)                                          \
    static struct gpio_driver_data gpio_driver_data_##n = {         \
        .pin_state = 0,                                            \
    };                                                             \
                                                                   \
    static const struct gpio_driver_config gpio_driver_config_##n = {\
        .base_addr = DT_INST_REG_ADDR(n),                         \
        .port_num = DT_INST_PROP(n, port),                        \
    };                                                             \
                                                                   \
    DEVICE_DT_INST_DEFINE(n,                                      \
                         gpio_driver_init,                         \
                         NULL,                                     \
                         &gpio_driver_data_##n,                    \
                         &gpio_driver_config_##n,                  \
                         POST_KERNEL,                              \
                         CONFIG_GPIO_INIT_PRIORITY,                \
                         &gpio_driver_api);

DT_INST_FOREACH_STATUS_OKAY(GPIO_DRIVER_INIT)
```

### 2. I2C 驱动

```c
#include <zephyr/drivers/i2c.h>

/* 驱动数据 */
struct i2c_driver_data {
    struct k_sem lock;
    uint32_t speed;
};

/* 驱动配置 */
struct i2c_driver_config {
    uint32_t base_addr;
    uint32_t irq_num;
};

/* I2C 传输函数 */
static int i2c_driver_transfer(const struct device *dev,
                              struct i2c_msg *msgs,
                              uint8_t num_msgs,
                              uint16_t addr)
{
    struct i2c_driver_data *data = dev->data;
    const struct i2c_driver_config *config = dev->config;
    int ret = 0;

    k_sem_take(&data->lock, K_FOREVER);

    /* 处理每个消息 */
    for (int i = 0; i < num_msgs; i++) {
        if (msgs[i].flags & I2C_MSG_READ) {
            /* 读取操作 */
        } else {
            /* 写入操作 */
        }
    }

    k_sem_give(&data->lock);

    return ret;
}

/* I2C 驱动 API */
static const struct i2c_driver_api i2c_driver_api = {
    .transfer = i2c_driver_transfer,
};

/* 驱动初始化宏 */
#define I2C_DRIVER_INIT(n)                                          \
    static struct i2c_driver_data i2c_driver_data_##n = {         \
        .speed = DT_INST_PROP(n, clock_frequency),               \
    };                                                             \
                                                                   \
    static const struct i2c_driver_config i2c_driver_config_##n = {\
        .base_addr = DT_INST_REG_ADDR(n),                         \
        .irq_num = DT_INST_IRQN(n),                              \
    };                                                             \
                                                                   \
    DEVICE_DT_INST_DEFINE(n,                                      \
                         i2c_driver_init,                         \
                         NULL,                                     \
                         &i2c_driver_data_##n,                    \
                         &i2c_driver_config_##n,                  \
                         POST_KERNEL,                              \
                         CONFIG_I2C_INIT_PRIORITY,                \
                         &i2c_driver_api);

DT_INST_FOREACH_STATUS_OKAY(I2C_DRIVER_INIT)
```

## 调试技巧

### 1. 使用日志

```c
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(my_driver, CONFIG_MY_DRIVER_LOG_LEVEL);

static int my_driver_function(const struct device *dev)
{
    LOG_DBG("Debug message");
    LOG_INF("Info message");
    LOG_WRN("Warning message");
    LOG_ERR("Error message");
    return 0;
}
```

### 2. 使用断言

```c
#include <zephyr/sys/check.h>

static int my_driver_function(const struct device *dev)
{
    int ret;

    /* 参数检查 */
    CHECKIF(dev == NULL) {
        return -EINVAL;
    }

    /* 状态检查 */
    CHECKIF(!device_is_ready(dev)) {
        return -ENODEV;
    }

    return 0;
}
```

### 3. 性能分析

```c
#include <zephyr/timing/timing.h>

static int my_driver_function(const struct device *dev)
{
    timing_t start_time, end_time;
    uint64_t cycles, ns;

    timing_init();
    timing_start();
    start_time = timing_counter_get();

    /* 执行操作 */

    end_time = timing_counter_get();
    cycles = timing_cycles_get(&start_time, &end_time);
    ns = timing_cycles_to_ns(cycles);
    timing_stop();

    LOG_INF("Operation took %llu ns", ns);
    return 0;
}
```

## 最佳实践

### 1. 驱动设计

- 使用标准 API
- 实现错误处理
- 支持电源管理
- 考虑并发访问

### 2. 资源管理

- 使用设备树配置
- 正确初始化资源
- 实现清理函数
- 避免资源泄漏

### 3. 性能优化

- 最小化关中断时间
- 使用 DMA（如适用）
- 优化数据传输
- 减少上下文切换

### 4. 可移植性

- 使用抽象层
- 避免硬编码
- 支持多平台
- 文档完善

## 常见问题

### 1. 初始化失败

**问题**：驱动初始化失败

**解决方案**：
- 检查设备树配置
- 验证硬件连接
- 确认时序要求
- 检查资源冲突

### 2. 通信错误

**问题**：与设备通信失败

**解决方案**：
- 检查总线配置
- 验证设备地址
- 确认协议实现
- 使用示波器分析

### 3. 并发问题

**问题**：多线程访问冲突

**解决方案**：
- 使用同步机制
- 保护共享资源
- 避免死锁
- 实现超时机制

### 4. 性能问题

**问题**：驱动性能不达标

**解决方案**：
- 优化数据路径
- 使用中断模式
- 实现 DMA 传输
- 减少等待时间

## 总结

Zephyr 驱动开发需要深入理解硬件特性和软件架构。通过遵循驱动模型、正确实现 API、使用设备树配置，可以开发出高质量的设备驱动程序。本文档提供了详细的指导和实例，帮助开发者更好地理解和实践 Zephyr 驱动开发。