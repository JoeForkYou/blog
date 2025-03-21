---
abbrlink: 36
title: Zephyr 电源管理系统指南
date: '2025-03-21 20:49:56'
tags: zephyr
categories: zephyr
---
# Zephyr 电源管理系统指南

## 1. 电源管理配置

### 1.1 基础配置 (prj.conf)
```plaintext
# 电源管理基础配置
CONFIG_PM=y
CONFIG_PM_DEVICE=y
CONFIG_PM_DEVICE_RUNTIME=y

# 设备电源状态支持
CONFIG_PM_DEVICE_POWER_STATE=y
CONFIG_PM_DEVICE_RUNTIME_AUTO=y

# 系统电源状态支持
CONFIG_SYS_POWER_MANAGEMENT=y
CONFIG_SYS_PM_STATE_LOCK=y
CONFIG_SYS_PM_DIRECT_FORCE_MODE=y

# 调试支持
CONFIG_PM_DEBUG=y
```

### 1.2 设备树配置
```dts
/ {
    chosen {
        zephyr,power-manager = &power_manager;
    };

    power_manager: power_manager {
        compatible = "zephyr,power-manager";
        status = "okay";
    };
};
```

## 2. 设备电源管理

### 2.1 设备电源状态
```c
#include <zephyr/kernel.h>
#include <zephyr/pm/device.h>
#include <zephyr/pm/device_runtime.h>

/* 设备电源管理回调 */
static int my_device_pm_action(const struct device *dev,
                             enum pm_device_action action)
{
    switch (action) {
    case PM_DEVICE_ACTION_RESUME:
        /* 恢复设备 */
        break;
    case PM_DEVICE_ACTION_SUSPEND:
        /* 挂起设备 */
        break;
    case PM_DEVICE_ACTION_TURN_ON:
        /* 开启设备 */
        break;
    case PM_DEVICE_ACTION_TURN_OFF:
        /* 关闭设备 */
        break;
    default:
        return -ENOTSUP;
    }

    return 0;
}

/* 设备定义 */
PM_DEVICE_DT_DEFINE(DT_NODELABEL(my_device), my_device_pm_action);

void device_pm_example(void)
{
    const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(my_device));
    int ret;

    if (!device_is_ready(dev)) {
        return;
    }

    /* 启用运行时电源管理 */
    ret = pm_device_runtime_enable(dev);
    if (ret < 0) {
        return;
    }

    /* 获取设备 */
    ret = pm_device_runtime_get(dev);
    if (ret < 0) {
        return;
    }

    /* 使用设备 */

    /* 释放设备 */
    ret = pm_device_runtime_put(dev);
    if (ret < 0) {
        return;
    }
}
```

### 2.2 电源状态约束
```c
#include <zephyr/kernel.h>
#include <zephyr/pm/device.h>
#include <zephyr/pm/device_runtime.h>

/* 电源状态约束定义 */
static struct pm_device_constraint constraint;

void power_constraint_example(void)
{
    const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(my_device));
    int ret;

    /* 设置电源约束 */
    ret = pm_device_runtime_constraint_set(dev, PM_DEVICE_ACTIVE_STATE);
    if (ret < 0) {
        return;
    }

    /* 执行需要保持设备活动的操作 */

    /* 释放约束 */
    ret = pm_device_runtime_constraint_release(dev);
    if (ret < 0) {
        return;
    }
}
```

## 3. 系统电源管理

### 3.1 系统电源状态
```c
#include <zephyr/kernel.h>
#include <zephyr/pm/pm.h>

void system_power_example(void)
{
    /* 强制进入特定电源状态 */
    pm_state_force(0u, &(struct pm_state_info){
        .state = PM_STATE_RUNTIME_IDLE
    });

    /* 阻止进入低功耗状态 */
    pm_system_suspend_state_disable();

    /* 允许进入低功耗状态 */
    pm_system_suspend_state_enable();
}
```

### 3.2 电源管理回调
```c
#include <zephyr/kernel.h>
#include <zephyr/pm/pm.h>

/* 进入低功耗状态前回调 */
static int pm_policy_pre_entry(enum pm_state state, uint8_t substate_id)
{
    /* 准备进入低功耗状态 */
    return 0;
}

/* 退出低功耗状态后回调 */
static void pm_policy_post_exit(enum pm_state state, uint8_t substate_id)
{
    /* 从低功耗状态恢复 */
}

/* 注册回调 */
PM_STATE_INFO_DT_DEFINE(DT_NODELABEL(cpu0), pm_policy_pre_entry, pm_policy_post_exit);
```

## 4. 电源域管理

### 4.1 电源域配置
```dts
/ {
    power-domains {
        compatible = "power-domain-map";
        #power-domain-cells = <1>;

        pd_core: pd_core {
            #power-domain-cells = <0>;
            power-domains = <&pm 0>;
            domain-name = "core";
            status = "okay";
        };
    };
};
```

### 4.2 电源域控制
```c
#include <zephyr/kernel.h>
#include <zephyr/pm/device.h>
#include <zephyr/pm/device_runtime.h>

void power_domain_example(void)
{
    const struct device *pd_dev = DEVICE_DT_GET(DT_NODELABEL(pd_core));
    int ret;

    if (!device_is_ready(pd_dev)) {
        return;
    }

    /* 开启电源域 */
    ret = pm_device_action_run(pd_dev, PM_DEVICE_ACTION_TURN_ON);
    if (ret < 0) {
        return;
    }

    /* 关闭电源域 */
    ret = pm_device_action_run(pd_dev, PM_DEVICE_ACTION_TURN_OFF);
    if (ret < 0) {
        return;
    }
}
```

## 5. 电源监控

### 5.1 电池监控
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/sensor.h>

void battery_monitor_example(void)
{
    const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(battery));
    struct sensor_value voltage, current;
    int ret;

    if (!device_is_ready(dev)) {
        return;
    }

    /* 读取电池电压 */
    ret = sensor_sample_fetch(dev);
    if (ret < 0) {
        return;
    }

    ret = sensor_channel_get(dev, SENSOR_CHAN_VOLTAGE, &voltage);
    if (ret < 0) {
        return;
    }

    /* 读取电池电流 */
    ret = sensor_channel_get(dev, SENSOR_CHAN_CURRENT, &current);
    if (ret < 0) {
        return;
    }

    printk("Battery: %d.%02d V, %d.%02d mA\n",
           voltage.val1, voltage.val2,
           current.val1, current.val2);
}
```

### 5.2 功耗分析
```c
#include <zephyr/kernel.h>
#include <zephyr/pm/pm.h>
#include <zephyr/drivers/counter.h>

/* 功耗统计结构 */
struct power_stats {
    uint32_t active_time;
    uint32_t sleep_time;
    uint32_t deep_sleep_time;
};

static struct power_stats stats;

void power_analysis_example(void)
{
    const struct device *counter_dev = DEVICE_DT_GET(DT_NODELABEL(timer0));
    uint32_t start_time;

    if (!device_is_ready(counter_dev)) {
        return;
    }

    /* 记录开始时间 */
    start_time = counter_get_value(counter_dev);

    /* 执行操作 */
    k_msleep(1000);

    /* 更新统计信息 */
    stats.active_time += counter_get_value(counter_dev) - start_time;

    /* 打印统计信息 */
    printk("Power Statistics:\n");
    printk("Active Time: %u ms\n", stats.active_time);
    printk("Sleep Time: %u ms\n", stats.sleep_time);
    printk("Deep Sleep Time: %u ms\n", stats.deep_sleep_time);
}
```

## 6. 低功耗优化

### 6.1 设备优化
```c
#include <zephyr/kernel.h>
#include <zephyr/pm/device.h>
#include <zephyr/pm/device_runtime.h>

void device_optimization(void)
{
    const struct device *const devs[] = {
        DEVICE_DT_GET(DT_NODELABEL(uart0)),
        DEVICE_DT_GET(DT_NODELABEL(i2c0)),
        DEVICE_DT_GET(DT_NODELABEL(spi0)),
    };

    /* 禁用不需要的设备 */
    for (int i = 0; i < ARRAY_SIZE(devs); i++) {
        if (device_is_ready(devs[i])) {
            pm_device_action_run(devs[i], PM_DEVICE_ACTION_TURN_OFF);
        }
    }

    /* 配置GPIO唤醒源 */
    const struct device *gpio_dev = DEVICE_DT_GET(DT_NODELABEL(gpio0));
    if (device_is_ready(gpio_dev)) {
        gpio_pin_configure(gpio_dev, 0,
                         GPIO_INPUT | GPIO_INT_EDGE_RISING);
    }
}
```

### 6.2 系统优化
```c
#include <zephyr/kernel.h>
#include <zephyr/pm/pm.h>

/* 系统空闲回调 */
static void pm_system_idle(void)
{
    /* 检查是否可以进入低功耗状态 */
    if (pm_system_is_off()) {
        /* 准备进入深度睡眠 */
        pm_state_force(0u, &(struct pm_state_info){
            .state = PM_STATE_SOFT_OFF
        });
    } else {
        /* 进入轻度睡眠 */
        pm_state_force(0u, &(struct pm_state_info){
            .state = PM_STATE_RUNTIME_IDLE
        });
    }
}

/* 配置空闲钩子 */
SYS_PM_IDLE_HOOK_DEFINE(pm_system_idle);
```

## 7. 电源管理调试

### 7.1 状态跟踪
```c
#include <zephyr/kernel.h>
#include <zephyr/pm/pm.h>
#include <zephyr/logging/log.h>

LOG_MODULE_REGISTER(power_debug, LOG_LEVEL_DBG);

void power_debug_example(void)
{
    /* 记录电源状态变化 */
    LOG_INF("Power state change:");
    LOG_INF("CPU: %d", pm_device_state_get(PM_CPU_DEV));
    LOG_INF("System: %d", pm_device_state_get(PM_SYSTEM_DEV));

    /* 记录设备状态 */
    const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(my_device));
    enum pm_device_state state;
    
    ret = pm_device_state_get(dev, &state);
    if (ret == 0) {
        LOG_INF("Device state: %d", state);
    }
}
```

### 7.2 功耗测量
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/adc.h>

#define ADC_RESOLUTION 12
#define ADC_CHANNEL 0

void power_measurement(void)
{
    const struct device *adc_dev = DEVICE_DT_GET(DT_NODELABEL(adc0));
    int16_t buf;
    struct adc_sequence sequence = {
        .buffer = &buf,
        .buffer_size = sizeof(buf),
        .resolution = ADC_RESOLUTION,
    };

    if (!device_is_ready(adc_dev)) {
        return;
    }

    /* 配置ADC通道 */
    struct adc_channel_cfg channel_cfg = {
        .gain = ADC_GAIN_1,
        .reference = ADC_REF_INTERNAL,
        .acquisition_time = ADC_ACQ_TIME_DEFAULT,
        .channel_id = ADC_CHANNEL,
    };
    adc_channel_setup(adc_dev, &channel_cfg);

    /* 读取电流值 */
    adc_sequence_init_dt(&sequence);
    if (adc_read(adc_dev, &sequence) == 0) {
        /* 转换为实际电流值 */
        int32_t current_ua = buf * (3300000 / ((1 << ADC_RESOLUTION) - 1));
        printk("Current consumption: %d uA\n", current_ua);
    }
}
```