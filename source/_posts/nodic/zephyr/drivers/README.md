---
abbrlink: 22
title: Zephyr 驱动系统指南
date: '2025-03-21 20:49:56'
---
# Zephyr 设备驱动开发指南

## 1. 驱动开发基础

### 1.1 驱动框架配置 (prj.conf)
```plaintext
# 基础驱动支持
CONFIG_SERIAL=y
CONFIG_GPIO=y
CONFIG_I2C=y
CONFIG_SPI=y
CONFIG_PINMUX=y
CONFIG_PINCTRL=y

# 设备树支持
CONFIG_DTS=y
CONFIG_HAS_DTS=y

# 驱动调试支持
CONFIG_DEVICE_SHELL=y
CONFIG_DEVICE_POWER_MANAGEMENT=y
```

### 1.2 基本驱动结构
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>

/* 驱动配置结构体 */
struct my_driver_config {
    const struct device *gpio_dev;
    uint32_t pin;
    gpio_flags_t flags;
};

/* 驱动数据结构体 */
struct my_driver_data {
    bool initialized;
    struct k_mutex lock;
};

/* 驱动API函数原型 */
typedef int (*my_driver_api_configure)(const struct device *dev,
                                     uint32_t config);
typedef int (*my_driver_api_write)(const struct device *dev,
                                 uint32_t value);
typedef int (*my_driver_api_read)(const struct device *dev,
                                uint32_t *value);

/* 驱动API结构体 */
struct my_driver_api {
    my_driver_api_configure configure;
    my_driver_api_write write;
    my_driver_api_read read;
};

/* 驱动初始化函数 */
static int my_driver_init(const struct device *dev)
{
    const struct my_driver_config *config = dev->config;
    struct my_driver_data *data = dev->data;
    int ret;

    /* 检查依赖设备 */
    if (!device_is_ready(config->gpio_dev)) {
        return -ENODEV;
    }

    /* 初始化互斥锁 */
    k_mutex_init(&data->lock);

    /* 配置GPIO */
    ret = gpio_pin_configure(config->gpio_dev,
                           config->pin,
                           config->flags);
    if (ret < 0) {
        return ret;
    }

    data->initialized = true;
    return 0;
}

/* 驱动API实现 */
static int my_driver_configure(const struct device *dev,
                             uint32_t config)
{
    struct my_driver_data *data = dev->data;
    int ret = 0;

    /* 获取锁 */
    k_mutex_lock(&data->lock, K_FOREVER);

    /* 实现配置逻辑 */

    /* 释放锁 */
    k_mutex_unlock(&data->lock);

    return ret;
}

static int my_driver_write(const struct device *dev,
                          uint32_t value)
{
    const struct my_driver_config *config = dev->config;
    struct my_driver_data *data = dev->data;
    int ret;

    k_mutex_lock(&data->lock, K_FOREVER);

    ret = gpio_pin_set(config->gpio_dev,
                      config->pin,
                      value);

    k_mutex_unlock(&data->lock);

    return ret;
}

static int my_driver_read(const struct device *dev,
                         uint32_t *value)
{
    const struct my_driver_config *config = dev->config;
    struct my_driver_data *data = dev->data;
    int ret;

    k_mutex_lock(&data->lock, K_FOREVER);

    ret = gpio_pin_get(config->gpio_dev,
                      config->pin);
    if (ret >= 0) {
        *value = ret;
        ret = 0;
    }

    k_mutex_unlock(&data->lock);

    return ret;
}

/* 驱动API结构体实例 */
static const struct my_driver_api my_driver_api_funcs = {
    .configure = my_driver_configure,
    .write = my_driver_write,
    .read = my_driver_read,
};

/* 驱动实例定义 */
#define MY_DRIVER_INIT(inst)                                              \
    static const struct my_driver_config my_driver_config_##inst = {      \
        .gpio_dev = DEVICE_DT_GET(DT_INST_GPIO_CTLR(inst, gpios)),      \
        .pin = DT_INST_GPIO_PIN(inst, gpios),                           \
        .flags = DT_INST_GPIO_FLAGS(inst, gpios),                       \
    };                                                                   \
                                                                        \
    static struct my_driver_data my_driver_data_##inst;                 \
                                                                        \
    DEVICE_DT_INST_DEFINE(inst,                                         \
                         my_driver_init,                                \
                         NULL,                                          \
                         &my_driver_data_##inst,                        \
                         &my_driver_config_##inst,                      \
                         POST_KERNEL,                                   \
                         CONFIG_MY_DRIVER_INIT_PRIORITY,               \
                         &my_driver_api_funcs);

/* 为每个设备树实例创建驱动实例 */
DT_INST_FOREACH_STATUS_OKAY(MY_DRIVER_INIT)
```

### 1.3 设备树绑定
```yaml
# dts/bindings/my-driver.yaml
description: My Driver Device

compatible: "vendor,my-driver"

include: base.yaml

properties:
  gpios:
    type: phandle-array
    required: true
    description: GPIO for control

  label:
    type: string
    required: false
    description: Human readable string describing the device
```

## 2. GPIO驱动开发

### 2.1 GPIO配置
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/* GPIO设备获取 */
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);
static const struct gpio_dt_spec button = GPIO_DT_SPEC_GET(DT_ALIAS(sw0), gpios);

int gpio_example(void)
{
    int ret;

    /* 检查设备就绪 */
    if (!device_is_ready(led.port)) {
        return -ENODEV;
    }

    if (!device_is_ready(button.port)) {
        return -ENODEV;
    }

    /* 配置GPIO */
    ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE);
    if (ret < 0) {
        return ret;
    }

    ret = gpio_pin_configure_dt(&button, GPIO_INPUT);
    if (ret < 0) {
        return ret;
    }

    /* GPIO操作 */
    gpio_pin_toggle_dt(&led);
    
    int val = gpio_pin_get_dt(&button);
    if (val >= 0) {
        printk("Button state: %d\n", val);
    }

    return 0;
}
```

### 2.2 GPIO中断
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/* 中断回调数据 */
static struct gpio_callback button_cb_data;

/* 中断回调函数 */
void button_pressed_cb(const struct device *dev,
                      struct gpio_callback *cb,
                      uint32_t pins)
{
    printk("Button pressed! pins: %x\n", pins);
}

int gpio_interrupt_example(void)
{
    const struct gpio_dt_spec button = GPIO_DT_SPEC_GET(DT_ALIAS(sw0), gpios);
    int ret;

    if (!device_is_ready(button.port)) {
        return -ENODEV;
    }

    /* 配置GPIO为输入并启用中断 */
    ret = gpio_pin_configure_dt(&button,
                               GPIO_INPUT |
                               GPIO_INT_DEBOUNCE |
                               GPIO_INT_EDGE_TO_ACTIVE);
    if (ret < 0) {
        return ret;
    }

    /* 初始化回调结构体 */
    gpio_init_callback(&button_cb_data,
                      button_pressed_cb,
                      BIT(button.pin));

    /* 添加回调 */
    ret = gpio_add_callback(button.port, &button_cb_data);
    if (ret < 0) {
        return ret;
    }

    /* 使能中断 */
    ret = gpio_pin_interrupt_configure_dt(&button,
                                        GPIO_INT_EDGE_TO_ACTIVE);
    if (ret < 0) {
        return ret;
    }

    return 0;
}
```

## 3. I2C驱动开发

### 3.1 I2C配置
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/i2c.h>

#define I2C_ADDR 0x50

int i2c_example(void)
{
    const struct device *const i2c_dev = DEVICE_DT_GET(DT_NODELABEL(i2c0));
    uint8_t data[2];
    int ret;

    if (!device_is_ready(i2c_dev)) {
        return -ENODEV;
    }

    /* 写数据 */
    data[0] = 0x00;  /* 寄存器地址 */
    data[1] = 0x42;  /* 数据 */
    ret = i2c_write(i2c_dev, data, sizeof(data), I2C_ADDR);
    if (ret < 0) {
        return ret;
    }

    /* 读数据 */
    uint8_t reg = 0x00;
    ret = i2c_write_read(i2c_dev,
                        I2C_ADDR,
                        &reg, 1,
                        data, 1);
    if (ret < 0) {
        return ret;
    }

    printk("Read value: 0x%02x\n", data[0]);
    return 0;
}
```

### 3.2 I2C设备驱动
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/i2c.h>

/* 设备配置结构体 */
struct my_i2c_config {
    const struct device *i2c_dev;
    uint16_t addr;
};

/* 设备数据结构体 */
struct my_i2c_data {
    struct k_mutex lock;
};

/* 驱动API */
static int my_i2c_read_reg(const struct device *dev,
                          uint8_t reg,
                          uint8_t *val)
{
    const struct my_i2c_config *config = dev->config;
    struct my_i2c_data *data = dev->data;
    int ret;

    k_mutex_lock(&data->lock, K_FOREVER);

    ret = i2c_write_read(config->i2c_dev,
                        config->addr,
                        &reg, 1,
                        val, 1);

    k_mutex_unlock(&data->lock);

    return ret;
}

static int my_i2c_write_reg(const struct device *dev,
                           uint8_t reg,
                           uint8_t val)
{
    const struct my_i2c_config *config = dev->config;
    struct my_i2c_data *data = dev->data;
    uint8_t buf[2];
    int ret;

    buf[0] = reg;
    buf[1] = val;

    k_mutex_lock(&data->lock, K_FOREVER);

    ret = i2c_write(config->i2c_dev,
                    buf, sizeof(buf),
                    config->addr);

    k_mutex_unlock(&data->lock);

    return ret;
}

/* 初始化函数 */
static int my_i2c_init(const struct device *dev)
{
    const struct my_i2c_config *config = dev->config;
    struct my_i2c_data *data = dev->data;

    if (!device_is_ready(config->i2c_dev)) {
        return -ENODEV;
    }

    k_mutex_init(&data->lock);

    return 0;
}

/* 驱动API结构体 */
static const struct my_i2c_driver_api {
    int (*read_reg)(const struct device *dev,
                    uint8_t reg,
                    uint8_t *val);
    int (*write_reg)(const struct device *dev,
                     uint8_t reg,
                     uint8_t val);
} my_i2c_api = {
    .read_reg = my_i2c_read_reg,
    .write_reg = my_i2c_write_reg,
};

/* 设备定义宏 */
#define MY_I2C_INIT(inst)                                              \
    static const struct my_i2c_config my_i2c_config_##inst = {        \
        .i2c_dev = DEVICE_DT_GET(DT_INST_BUS(inst)),                 \
        .addr = DT_INST_REG_ADDR(inst),                              \
    };                                                                \
                                                                      \
    static struct my_i2c_data my_i2c_data_##inst;                    \
                                                                      \
    DEVICE_DT_INST_DEFINE(inst,                                       \
                         my_i2c_init,                                 \
                         NULL,                                        \
                         &my_i2c_data_##inst,                        \
                         &my_i2c_config_##inst,                      \
                         POST_KERNEL,                                 \
                         CONFIG_I2C_INIT_PRIORITY,                   \
                         &my_i2c_api);

/* 为每个设备树实例创建驱动实例 */
DT_INST_FOREACH_STATUS_OKAY(MY_I2C_INIT)
```

## 4. SPI驱动开发

### 4.1 SPI配置
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/spi.h>

int spi_example(void)
{
    const struct device *const spi = DEVICE_DT_GET(DT_NODELABEL(spi0));
    uint8_t tx_buffer[] = {0x01, 0x02, 0x03};
    uint8_t rx_buffer[sizeof(tx_buffer)];
    int ret;

    if (!device_is_ready(spi)) {
        return -ENODEV;
    }

    /* SPI配置 */
    struct spi_config spi_cfg = {
        .operation = SPI_WORD_SET(8) | SPI_TRANSFER_MSB |
                    SPI_MODE_CPOL | SPI_MODE_CPHA,
        .frequency = 1000000,
    };

    struct spi_buf tx_buf = {
        .buf = tx_buffer,
        .len = sizeof(tx_buffer)
    };
    struct spi_buf rx_buf = {
        .buf = rx_buffer,
        .len = sizeof(rx_buffer)
    };

    struct spi_buf_set tx = {
        .buffers = &tx_buf,
        .count = 1
    };
    struct spi_buf_set rx = {
        .buffers = &rx_buf,
        .count = 1
    };

    /* 执行传输 */
    ret = spi_transceive(spi, &spi_cfg, &tx, &rx);
    if (ret < 0) {
        return ret;
    }

    return 0;
}
```

### 4.2 SPI设备驱动
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/spi.h>

/* 设备配置结构体 */
struct my_spi_config {
    const struct device *spi_dev;
    struct spi_config spi_cfg;
    struct spi_cs_control cs_ctrl;
};

/* 设备数据结构体 */
struct my_spi_data {
    struct k_mutex lock;
};

/* 驱动API */
static int my_spi_transfer(const struct device *dev,
                          const uint8_t *tx_buf,
                          uint8_t *rx_buf,
                          size_t len)
{
    const struct my_spi_config *config = dev->config;
    struct my_spi_data *data = dev->data;
    struct spi_buf tx = {
        .buf = (void *)tx_buf,
        .len = len
    };
    struct spi_buf rx = {
        .buf = rx_buf,
        .len = len
    };
    struct spi_buf_set tx_bufs = {
        .buffers = &tx,
        .count = 1
    };
    struct spi_buf_set rx_bufs = {
        .buffers = &rx,
        .count = 1
    };
    int ret;

    k_mutex_lock(&data->lock, K_FOREVER);

    ret = spi_transceive(config->spi_dev,
                        &config->spi_cfg,
                        &tx_bufs,
                        &rx_bufs);

    k_mutex_unlock(&data->lock);

    return ret;
}

/* 初始化函数 */
static int my_spi_init(const struct device *dev)
{
    const struct my_spi_config *config = dev->config;
    struct my_spi_data *data = dev->data;

    if (!device_is_ready(config->spi_dev)) {
        return -ENODEV;
    }

    k_mutex_init(&data->lock);

    return 0;
}

/* 驱动API结构体 */
static const struct my_spi_driver_api {
    int (*transfer)(const struct device *dev,
                   const uint8_t *tx_buf,
                   uint8_t *rx_buf,
                   size_t len);
} my_spi_api = {
    .transfer = my_spi_transfer,
};

/* 设备定义宏 */
#define MY_SPI_INIT(inst)                                              \
    static const struct my_spi_config my_spi_config_##inst = {        \
        .spi_dev = DEVICE_DT_GET(DT_INST_BUS(inst)),                 \
        .spi_cfg = {                                                  \
            .operation = SPI_WORD_SET(8) | SPI_TRANSFER_MSB |        \
                        SPI_MODE_CPOL | SPI_MODE_CPHA,               \
            .frequency = DT_INST_PROP(inst, spi_max_frequency),      \
        },                                                           \
        .cs_ctrl = SPI_CS_CONTROL_INIT(DT_INST_SPI_DEV(inst),       \
                                      DT_INST_SPI_CS_GPIOS(inst)),   \
    };                                                               \
                                                                     \
    static struct my_spi_data my_spi_data_##inst;                   \
                                                                     \
    DEVICE_DT_INST_DEFINE(inst,                                      \
                         my_spi_init,                                \
                         NULL,                                       \
                         &my_spi_data_##inst,                       \
                         &my_spi_config_##inst,                     \
                         POST_KERNEL,                                \
                         CONFIG_SPI_INIT_PRIORITY,                  \
                         &my_spi_api);

/* 为每个设备树实例创建驱动实例 */
DT_INST_FOREACH_STATUS_OKAY(MY_SPI_INIT)
```

## 5. 驱动测试

### 5.1 单元测试
```c
#include <zephyr/ztest.h>
#include <zephyr/drivers/gpio.h>

/* 测试夹具设置 */
static void *gpio_setup(void)
{
    const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(gpio0));
    zassert_true(device_is_ready(dev), "GPIO device not ready");
    return (void *)dev;
}

/* GPIO测试用例 */
ZTEST_SUITE(gpio_test, NULL, gpio_setup, NULL, NULL, NULL);

ZTEST(gpio_test, test_gpio_configure)
{
    const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(gpio0));
    int ret;

    ret = gpio_pin_configure(dev, 0, GPIO_OUTPUT);
    zassert_equal(ret, 0, "Failed to configure GPIO pin");

    ret = gpio_pin_set(dev, 0, 1);
    zassert_equal(ret, 0, "Failed to set GPIO pin");

    ret = gpio_pin_get(dev, 0);
    zassert_equal(ret, 1, "Unexpected GPIO pin state");
}
```

### 5.2 集成测试
```c
#include <zephyr/ztest.h>
#include <zephyr/drivers/i2c.h>
#include <zephyr/drivers/spi.h>

/* I2C和SPI组合测试 */
ZTEST(driver_test, test_i2c_spi_transfer)
{
    const struct device *i2c_dev = DEVICE_DT_GET(DT_NODELABEL(i2c0));
    const struct device *spi_dev = DEVICE_DT_GET(DT_NODELABEL(spi0));
    uint8_t data = 0;
    int ret;

    /* I2C测试 */
    ret = i2c_reg_read_byte(i2c_dev, 0x50, 0x00, &data);
    zassert_equal(ret, 0, "I2C read failed");

    /* SPI测试 */
    struct spi_config spi_cfg = {
        .operation = SPI_WORD_SET(8),
        .frequency = 1000000,
    };

    ret = spi_write(spi_dev, &spi_cfg, &data, 1);
    zassert_equal(ret, 0, "SPI write failed");
}
```

## 6. 驱动Shell命令

### 6.1 Shell命令实现
```c
#include <zephyr/shell/shell.h>
#include <zephyr/drivers/gpio.h>

/* GPIO shell命令 */
static int cmd_gpio_get(const struct shell *shell,
                       size_t argc, char **argv)
{
    const struct device *dev;
    uint32_t pin;
    int ret;

    if (argc != 3) {
        shell_error(shell, "Wrong parameters count");
        return -EINVAL;
    }

    dev = device_get_binding(argv[1]);
    if (!dev) {
        shell_error(shell, "Device not found: %s", argv[1]);
        return -ENODEV;
    }

    pin = strtoul(argv[2], NULL, 10);
    ret = gpio_pin_get(dev, pin);

    if (ret < 0) {
        shell_error(shell, "Failed to get pin state");
        return ret;
    }

    shell_print(shell, "Pin %d state: %d", pin, ret);
    return 0;
}

/* Shell命令注册 */
SHELL_STATIC_SUBCMD_SET_CREATE(gpio_cmds,
    SHELL_CMD(get, NULL, "Get GPIO pin state",
              cmd_gpio_get),
    SHELL_SUBCMD_SET_END
);

SHELL_CMD_REGISTER(gpio, &gpio_cmds,
                  "GPIO commands", NULL);
```

### 6.2 Shell命令配置
```plaintext
# Shell配置 (prj.conf)
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_SERIAL=y
CONFIG_SHELL_PROMPT_UART="gpio:~$ "

# GPIO Shell配置
CONFIG_GPIO_SHELL=y
```

## 7. 电源管理

### 7.1 设备电源管理
```c
#include <zephyr/kernel.h>
#include <zephyr/pm/device.h>

/* 电源管理回调 */
static int my_driver_pm_action(const struct device *dev,
                             enum pm_device_action action)
{
    switch (action) {
    case PM_DEVICE_ACTION_RESUME:
        /* 恢复设备 */
        break;
    case PM_DEVICE_ACTION_SUSPEND:
        /* 挂起设备 */
        break;
    case PM_DEVICE_ACTION_TURN_OFF:
        /* 关闭设备 */
        break;
    case PM_DEVICE_ACTION_TURN_ON:
        /* 打开设备 */
        break;
    default:
        return -ENOTSUP;
    }

    return 0;
}

/* 设备定义 */
PM_DEVICE_DT_INST_DEFINE(0, my_driver_pm_action);

/* 使用电源管理 */
void power_management_example(void)
{
    const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(my_dev));
    int ret;

    /* 挂起设备 */
    ret = pm_device_action_run(dev,
                              PM_DEVICE_ACTION_SUSPEND);
    if (ret < 0) {
        printk("Failed to suspend device\n");
    }

    /* 恢复设备 */
    ret = pm_device_action_run(dev,
                              PM_DEVICE_ACTION_RESUME);
    if (ret < 0) {
        printk("Failed to resume device\n");
    }
}
```

### 7.2 电源状态配置
```plaintext
# 电源管理配置 (prj.conf)
CONFIG_PM=y
CONFIG_PM_DEVICE=y
CONFIG_PM_DEVICE_RUNTIME=y
```