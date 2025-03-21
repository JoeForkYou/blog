---
abbrlink: 51
title: power_management
date: '2025-03-21 20:49:56'
---
# Zephyr USB 电源管理指南

## 1. USB 电源管理基础

### 1.1 电源管理配置 (prj.conf)
```plaintext
# USB 设备栈支持
CONFIG_USB_DEVICE_STACK=y

# 电源管理支持
CONFIG_PM=y
CONFIG_PM_DEVICE=y
CONFIG_PM_DEVICE_RUNTIME=y

# USB 电源管理
CONFIG_USB_DEVICE_BOS=y
CONFIG_USB_DEVICE_OS_DESC=y
CONFIG_USB_DEVICE_REMOTE_WAKEUP=y
```

### 1.2 电源描述符配置
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>

/* 配置描述符 */
static const struct usb_cfg_descriptor cfg_desc = {
    .bLength = sizeof(struct usb_cfg_descriptor),
    .bDescriptorType = USB_DESC_CONFIGURATION,
    .wTotalLength = 0, /* 运行时计算 */
    .bNumInterfaces = 1,
    .bConfigurationValue = 1,
    .iConfiguration = 0,
    /* 支持远程唤醒 */
    .bmAttributes = USB_CONFIG_ATT_ONE | USB_CONFIG_ATT_SELF_POWERED | USB_CONFIG_ATT_WAKEUP,
    .bMaxPower = 50, /* 100 mA */
};

/* BOS 描述符 */
static const struct usb_bos_descriptor bos_desc = {
    .bLength = sizeof(struct usb_bos_descriptor),
    .bDescriptorType = USB_DESC_BOS,
    .wTotalLength = sizeof(struct usb_bos_descriptor) +
                   sizeof(struct usb_dev_cap_platform_descriptor),
    .bNumDeviceCaps = 1,
};

/* 平台描述符 */
static const struct usb_dev_cap_platform_descriptor platform_desc = {
    .bLength = sizeof(struct usb_dev_cap_platform_descriptor),
    .bDescriptorType = USB_DESC_DEVICE_CAPABILITY,
    .bDevCapabilityType = USB_DC_PLATFORM,
    .bReserved = 0,
    /* MS OS 2.0 描述符 */
    .PlatformCapabilityUUID = {
        0xDF, 0x60, 0xDD, 0xD8, 0x89, 0x45, 0xC7, 0x4C,
        0x9C, 0xD2, 0x65, 0x9D, 0x9E, 0x64, 0x8A, 0x9F
    },
};
```

## 2. USB 挂起/恢复处理

### 2.1 挂起/恢复回调
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/pm/pm.h>

/* USB 设备状态回调 */
static void usb_status_cb(enum usb_dc_status_code status, const uint8_t *param)
{
    switch (status) {
    case USB_DC_SUSPEND:
        printk("USB suspended\n");
        /* 进入低功耗模式 */
        pm_device_action_run(PM_DEVICE_ACTION_SUSPEND);
        break;
    case USB_DC_RESUME:
        printk("USB resumed\n");
        /* 退出低功耗模式 */
        pm_device_action_run(PM_DEVICE_ACTION_RESUME);
        break;
    default:
        break;
    }
}

/* 初始化 USB 电源管理 */
void usb_pm_init(void)
{
    /* 注册状态回调 */
    usb_register_status_callback(usb_status_cb);
}
```

### 2.2 远程唤醒实现
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>

/* 远程唤醒标志 */
static volatile bool remote_wakeup_enabled;

/* 处理 SET_FEATURE/CLEAR_FEATURE 请求 */
static int usb_feature_handler(struct usb_setup_packet *setup,
                              int32_t *len, uint8_t **data)
{
    if (setup->bRequest == USB_REQUEST_SET_FEATURE) {
        if (setup->wValue == USB_FEATURE_REMOTE_WAKEUP) {
            remote_wakeup_enabled = true;
            return 0;
        }
    } else if (setup->bRequest == USB_REQUEST_CLEAR_FEATURE) {
        if (setup->wValue == USB_FEATURE_REMOTE_WAKEUP) {
            remote_wakeup_enabled = false;
            return 0;
        }
    }

    return -EINVAL;
}

/* 发送远程唤醒 */
int usb_remote_wakeup(void)
{
    if (!remote_wakeup_enabled) {
        return -EACCES;
    }

    return usb_dc_wakeup_request();
}
```

## 3. 总线供电管理

### 3.1 总线供电配置
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>

/* 配置描述符 - 总线供电 */
static const struct usb_cfg_descriptor bus_powered_cfg_desc = {
    .bLength = sizeof(struct usb_cfg_descriptor),
    .bDescriptorType = USB_DESC_CONFIGURATION,
    .wTotalLength = 0, /* 运行时计算 */
    .bNumInterfaces = 1,
    .bConfigurationValue = 1,
    .iConfiguration = 0,
    /* 总线供电 */
    .bmAttributes = USB_CONFIG_ATT_ONE,
    .bMaxPower = 50, /* 100 mA */
};

/* 配置描述符 - 自供电 */
static const struct usb_cfg_descriptor self_powered_cfg_desc = {
    .bLength = sizeof(struct usb_cfg_descriptor),
    .bDescriptorType = USB_DESC_CONFIGURATION,
    .wTotalLength = 0, /* 运行时计算 */
    .bNumInterfaces = 1,
    .bConfigurationValue = 1,
    .iConfiguration = 0,
    /* 自供电 */
    .bmAttributes = USB_CONFIG_ATT_ONE | USB_CONFIG_ATT_SELF_POWERED,
    .bMaxPower = 0, /* 0 mA */
};
```

### 3.2 电源状态监控
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/usb/usb_device.h>

/* VBUS 检测引脚 */
static const struct gpio_dt_spec vbus_gpio =
    GPIO_DT_SPEC_GET(DT_NODELABEL(vbus_detect), gpios);

/* VBUS 中断回调 */
static void vbus_callback(const struct device *port,
                         struct gpio_callback *cb,
                         gpio_port_pins_t pins)
{
    /* 检查 VBUS 状态 */
    int vbus_present = gpio_pin_get_dt(&vbus_gpio);

    if (vbus_present) {
        printk("VBUS connected\n");
        /* 启用 USB 设备 */
        usb_enable(NULL);
    } else {
        printk("VBUS disconnected\n");
        /* 禁用 USB 设备 */
        usb_disable();
    }
}

static struct gpio_callback vbus_cb_data;

/* 初始化 VBUS 检测 */
void vbus_detection_init(void)
{
    /* 配置 VBUS 检测引脚 */
    gpio_pin_configure_dt(&vbus_gpio, GPIO_INPUT);
    gpio_pin_interrupt_configure_dt(&vbus_gpio, GPIO_INT_EDGE_BOTH);

    /* 设置中断回调 */
    gpio_init_callback(&vbus_cb_data, vbus_callback, BIT(vbus_gpio.pin));
    gpio_add_callback(vbus_gpio.port, &vbus_cb_data);
}
```

## 4. 电源协商 (USB PD)

### 4.1 USB PD 配置
```plaintext
# USB PD 配置 (prj.conf)
CONFIG_USBC_STACK=y
CONFIG_USBC_PD=y
CONFIG_USBC_PD_TCPC=y
CONFIG_USBC_PD_POLICY=y
```

### 4.2 USB PD 实现
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usbc/usbc_pd.h>

/* USB PD 事件回调 */
static void pd_event_handler(const struct device *dev,
                            enum usbc_pd_event event,
                            void *user_data)
{
    switch (event) {
    case USBC_PD_EVT_SOURCE_ATTACHED:
        printk("USB PD source attached\n");
        break;
    case USBC_PD_EVT_SOURCE_DETACHED:
        printk("USB PD source detached\n");
        break;
    case USBC_PD_EVT_POWER_READY:
        printk("USB PD power ready\n");
        break;
    default:
        break;
    }
}

/* 初始化 USB PD */
void usb_pd_init(void)
{
    const struct device *pd_dev = DEVICE_DT_GET(DT_NODELABEL(usbc_pd));

    /* 注册 PD 事件回调 */
    usbc_pd_register_callback(pd_dev, pd_event_handler, NULL);
}

/* 请求电源数据对象 */
void request_power(void)
{
    const struct device *pd_dev = DEVICE_DT_GET(DT_NODELABEL(usbc_pd));
    struct usbc_pd_pdo pdo;

    /* 获取源能力 */
    int ret = usbc_pd_get_source_capabilities(pd_dev, &pdo, 1);
    if (ret <= 0) {
        return;
    }

    /* 请求 5V/2A */
    struct usbc_pd_rdo rdo = {
        .object_pos = 1,
        .max_current = 200, /* 2A */
        .op_current = 200,  /* 2A */
    };

    usbc_pd_request_power(pd_dev, &rdo);
}
```

## 5. 电源状态转换

### 5.1 电源状态机
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/pm/pm.h>

/* 电源状态 */
enum power_state {
    POWER_STATE_ACTIVE,
    POWER_STATE_IDLE,
    POWER_STATE_SUSPEND,
    POWER_STATE_OFF,
};

static enum power_state current_state = POWER_STATE_ACTIVE;
static uint32_t last_activity;

/* 电源状态转换 */
void update_power_state(void)
{
    uint32_t now = k_uptime_get_32();
    uint32_t idle_time = now - last_activity;

    switch (current_state) {
    case POWER_STATE_ACTIVE:
        if (idle_time > 5000) {
            /* 5秒无活动，进入空闲状态 */
            current_state = POWER_STATE_IDLE;
            /* 降低时钟频率 */
            pm_device_action_run(PM_DEVICE_ACTION_TURN_OFF);
        }
        break;
    case POWER_STATE_IDLE:
        if (idle_time > 30000) {
            /* 30秒无活动，进入挂起状态 */
            current_state = POWER_STATE_SUSPEND;
            /* 进入深度睡眠 */
            pm_device_action_run(PM_DEVICE_ACTION_SUSPEND);
        }
        break;
    case POWER_STATE_SUSPEND:
        /* 在挂起状态下保持，直到有外部唤醒 */
        break;
    default:
        break;
    }
}

/* 记录活动 */
void record_activity(void)
{
    last_activity = k_uptime_get_32();

    /* 如果不在活动状态，恢复到活动状态 */
    if (current_state != POWER_STATE_ACTIVE) {
        current_state = POWER_STATE_ACTIVE;
        pm_device_action_run(PM_DEVICE_ACTION_RESUME);
    }
}
```

### 5.2 低功耗模式
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/pm/pm.h>

/* 低功耗模式配置 */
struct pm_state_info states[] = {
    {PM_STATE_RUNTIME_IDLE, 0},
    {PM_STATE_SUSPEND_TO_IDLE, 0},
    {PM_STATE_SUSPEND_TO_RAM, 0},
};

/* 进入低功耗模式 */
void enter_low_power_mode(enum pm_state state)
{
    /* 准备进入低功耗模式 */
    switch (state) {
    case PM_STATE_RUNTIME_IDLE:
        /* 轻度睡眠，快速唤醒 */
        break;
    case PM_STATE_SUSPEND_TO_IDLE:
        /* 中度睡眠，关闭更多外设 */
        break;
    case PM_STATE_SUSPEND_TO_RAM:
        /* 深度睡眠，只保留关键唤醒源 */
        break;
    default:
        return;
    }

    /* 设置唤醒源 */
    pm_device_wakeup_enable(DEVICE_DT_GET(DT_NODELABEL(usbd)), true);

    /* 进入低功耗模式 */
    pm_state_force(0, &(struct pm_state_info){
        .state = state
    });
}
```

## 6. 电池供电设备

### 6.1 电池电量监控
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/sensor.h>
#include <zephyr/usb/usb_device.h>

/* 电池状态 */
struct battery_status {
    uint8_t level;       /* 0-100% */
    bool charging;       /* 是否充电 */
    uint16_t voltage_mv; /* 电压（毫伏） */
    int16_t current_ma;  /* 电流（毫安） */
};

static struct battery_status bat_status;

/* 更新电池状态 */
void update_battery_status(void)
{
    const struct device *bat_dev = DEVICE_DT_GET(DT_NODELABEL(battery));
    struct sensor_value val;

    /* 读取电池电量 */
    sensor_sample_fetch(bat_dev);
    sensor_channel_get(bat_dev, SENSOR_CHAN_GAUGE_STATE_OF_CHARGE, &val);
    bat_status.level = val.val1;

    /* 读取充电状态 */
    sensor_channel_get(bat_dev, SENSOR_CHAN_GAUGE_STATE_OF_CHARGE, &val);
    bat_status.charging = (val.val1 > 0);

    /* 读取电压 */
    sensor_channel_get(bat_dev, SENSOR_CHAN_GAUGE_VOLTAGE, &val);
    bat_status.voltage_mv = val.val1 * 1000 + val.val2 / 1000;

    /* 读取电流 */
    sensor_channel_get(bat_dev, SENSOR_CHAN_GAUGE_CURRENT, &val);
    bat_status.current_ma = val.val1 * 1000 + val.val2 / 1000;
}

/* 根据电池状态调整 USB 配置 */
void adjust_usb_power_config(void)
{
    /* 如果电池电量低于阈值，降低 USB 功耗 */
    if (bat_status.level < 10 && !bat_status.charging) {
        /* 降低 USB 端点数量 */
        usb_disable_endpoint(0x81);
        usb_disable_endpoint(0x01);

        /* 降低 USB 速度 */
        usb_dc_set_device_speed(USB_SPEED_FULL);
    }
}
```

### 6.2 充电控制
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/* 充电控制引脚 */
static const struct gpio_dt_spec charge_en =
    GPIO_DT_SPEC_GET(DT_NODELABEL(charge_enable), gpios);

/* 充电状态引脚 */
static const struct gpio_dt_spec charge_stat =
    GPIO_DT_SPEC_GET(DT_NODELABEL(charge_status), gpios);

/* 充电电流设置 */
enum charge_current {
    CHARGE_CURRENT_100MA,
    CHARGE_CURRENT_500MA,
    CHARGE_CURRENT_1000MA,
    CHARGE_CURRENT_2000MA,
};

/* 设置充电电流 */
void set_charge_current(enum charge_current current)
{
    const struct device *i2c_dev = DEVICE_DT_GET(DT_NODELABEL(i2c0));
    uint8_t reg_addr = 0x02;
    uint8_t reg_val;

    /* 根据电流选择寄存器值 */
    switch (current) {
    case CHARGE_CURRENT_100MA:
        reg_val = 0x01;
        break;
    case CHARGE_CURRENT_500MA:
        reg_val = 0x02;
        break;
    case CHARGE_CURRENT_1000MA:
        reg_val = 0x04;
        break;
    case CHARGE_CURRENT_2000MA:
        reg_val = 0x08;
        break;
    default:
        return;
    }

    /* 写入充电控制器寄存器 */
    i2c_write(i2c_dev, &reg_val, 1, 0x6B);
}

/* 启用/禁用充电 */
void set_charging(bool enable)
{
    gpio_pin_set_dt(&charge_en, enable ? 1 : 0);
}

/* 获取充电状态 */
bool is_charging(void)
{
    return gpio_pin_get_dt(&charge_stat) == 1;
}
```

## 7. USB 电源管理最佳实践

1. 电源状态管理：
   - 实现完整的电源状态机
   - 正确处理挂起/恢复事件
   - 使用适当的低功耗模式

2. 远程唤醒：
   - 正确实现远程唤醒功能
   - 仅在主机允许时使用
   - 提供唤醒源配置

3. 总线供电管理：
   - 遵循 USB 总线供电规范
   - 监控电流消耗
   - 实现动态功率调整

4. 电池管理：
   - 监控电池状态
   - 根据电池电量调整功耗
   - 实现智能充电控制

5. USB PD 实现：
   - 正确协商电源能力
   - 处理电源角色转换
   - 支持多种电源配置文件

6. 测试验证：
   - 测试不同电源状态转换
   - 验证唤醒功能
   - 测量实际功耗

7. 错误处理：
   - 检测电源故障
   - 实现安全关断
   - 提供电源状态诊断

8. 兼容性：
   - 支持不同 USB 主机的电源管理
   - 遵循 USB 电源规范
   - 考虑向后兼容性