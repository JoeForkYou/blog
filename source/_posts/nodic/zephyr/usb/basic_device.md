---
abbrlink: 48
title: basic_device
date: '2025-03-21 20:49:56'
---
# Zephyr USB 基础设备类指南

## 1. USB 基础配置

### 1.1 通用配置 (prj.conf)
```plaintext
# USB 设备栈支持
CONFIG_USB_DEVICE_STACK=y
CONFIG_USB_DEVICE_MANUFACTURER="Zephyr"
CONFIG_USB_DEVICE_PRODUCT="USB Device"
CONFIG_USB_DEVICE_VID=0x2FE3
CONFIG_USB_DEVICE_PID=0x0100
CONFIG_USB_DEVICE_SN="0123456789AB"

# USB 设备日志
CONFIG_USB_DRIVER_LOG_LEVEL_ERR=y
CONFIG_USB_DEVICE_LOG_LEVEL_ERR=y

# USB 缓冲区配置
CONFIG_USB_REQUEST_BUFFER_SIZE=128
CONFIG_USB_MAX_NUM_ENDPOINTS=8
```

### 1.2 设备描述符配置
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/usb_common.h>

/* 设备描述符 */
static const struct usb_device_descriptor dev_desc = {
    .bLength = sizeof(struct usb_device_descriptor),
    .bDescriptorType = USB_DESC_DEVICE,
    .bcdUSB = sys_cpu_to_le16(USB_2_0),
    .bDeviceClass = 0,
    .bDeviceSubClass = 0,
    .bDeviceProtocol = 0,
    .bMaxPacketSize0 = USB_MAX_PACKET_SIZE,
    .idVendor = sys_cpu_to_le16(CONFIG_USB_DEVICE_VID),
    .idProduct = sys_cpu_to_le16(CONFIG_USB_DEVICE_PID),
    .bcdDevice = sys_cpu_to_le16(0x0100),
    .iManufacturer = 1,
    .iProduct = 2,
    .iSerialNumber = 3,
    .bNumConfigurations = 1,
};

/* 配置描述符 */
static const struct usb_cfg_descriptor cfg_desc = {
    .bLength = sizeof(struct usb_cfg_descriptor),
    .bDescriptorType = USB_DESC_CONFIGURATION,
    .wTotalLength = 0, /* 将在运行时计算 */
    .bNumInterfaces = 1,
    .bConfigurationValue = 1,
    .iConfiguration = 0,
    .bmAttributes = USB_CONFIG_ATT_ONE | USB_CONFIG_ATT_SELF_POWERED,
    .bMaxPower = 50, /* 100 mA */
};
```

## 2. USB CDC-ACM 设备类

### 2.1 CDC-ACM 配置
```plaintext
# CDC-ACM 配置 (prj.conf)
CONFIG_USB_CDC_ACM=y
CONFIG_UART_LINE_CTRL=y
CONFIG_UART_CONSOLE_ON_DEV_NAME="CDC_ACM_0"
```

### 2.2 CDC-ACM 实现
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/class/usb_cdc.h>

static const struct device *cdc_dev;
static volatile bool configured;
static struct k_work_delayable tx_work;
static char tx_buffer[64];

/* USB 配置回调 */
static void cdc_acm_cfg_cb(const struct device *dev)
{
    configured = true;
}

/* USB 使能回调 */
static void cdc_acm_enable_cb(const struct device *dev)
{
    cdc_dev = dev;
}

/* CDC ACM 回调结构体 */
static const struct cdc_acm_ops ops = {
    .configured = cdc_acm_cfg_cb,
    .enable = cdc_acm_enable_cb,
};

void cdc_acm_init(void)
{
    int ret;

    /* 注册回调 */
    ret = usb_enable(NULL);
    if (ret != 0) {
        return;
    }

    /* 等待配置完成 */
    while (!configured) {
        k_sleep(K_MSEC(100));
    }
}

void cdc_acm_send_data(void)
{
    if (!configured) {
        return;
    }

    /* 准备发送数据 */
    strcpy(tx_buffer, "Hello from CDC ACM\r\n");
    
    /* 发送数据 */
    uart_fifo_fill(cdc_dev, tx_buffer, strlen(tx_buffer));
}
```

## 3. USB HID 设备类

### 3.1 HID 配置
```plaintext
# HID 配置 (prj.conf)
CONFIG_USB_HID=y
CONFIG_USB_HID_BOOT_PROTOCOL=y
CONFIG_USB_HID_POLL_INTERVAL_MS=8
```

### 3.2 HID 键盘实现
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/class/usb_hid.h>

/* HID 报告描述符 */
static const uint8_t hid_report_desc[] = {
    HID_USAGE_PAGE(HID_USAGE_GEN_DESKTOP),
    HID_USAGE(HID_USAGE_GEN_KEYBOARD),
    HID_COLLECTION(HID_COLLECTION_APPLICATION),
        HID_USAGE_PAGE(HID_USAGE_KEY),
        HID_USAGE_MIN8(0),
        HID_USAGE_MAX8(0xFF),
        HID_LOGICAL_MIN8(0),
        HID_LOGICAL_MAX8(1),
        HID_REPORT_SIZE(1),
        HID_REPORT_COUNT(8),
        HID_INPUT(0x02),
    HID_END_COLLECTION,
};

static const struct hid_ops ops = {
    .get_report = NULL,
    .get_idle = NULL,
    .get_protocol = NULL,
    .set_report = NULL,
    .set_idle = NULL,
    .set_protocol = NULL,
    .int_in_ready = NULL,
    .protocol_change = NULL,
};

void hid_keyboard_init(void)
{
    /* 初始化 HID */
    usb_hid_register_device(hid_report_desc, sizeof(hid_report_desc), &ops);
    
    /* 使能 USB */
    usb_enable(NULL);
}

void hid_keyboard_send_key(uint8_t key)
{
    uint8_t report[8] = { 0 };
    
    /* 设置按键 */
    report[0] = key;
    
    /* 发送报告 */
    hid_int_ep_write(report, sizeof(report), NULL);
}
```

## 4. USB MSC 设备类

### 4.1 MSC 配置
```plaintext
# MSC 配置 (prj.conf)
CONFIG_USB_MASS_STORAGE=y
CONFIG_DISK_ACCESS=y
CONFIG_FILE_SYSTEM=y
```

### 4.2 MSC 实现
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/class/usb_msc.h>
#include <zephyr/storage/disk_access.h>

/* 存储介质配置 */
#define DISK_SECTOR_SIZE 512
#define DISK_SECTOR_COUNT 2048
#define DISK_MEMORY_SIZE (DISK_SECTOR_SIZE * DISK_SECTOR_COUNT)

static uint8_t disk_memory[DISK_MEMORY_SIZE];

/* 磁盘操作回调 */
static int disk_init(void)
{
    return 0;
}

static int disk_status(void)
{
    return DISK_STATUS_OK;
}

static int disk_read(uint8_t *data, uint32_t sector, uint32_t count)
{
    memcpy(data, &disk_memory[sector * DISK_SECTOR_SIZE],
           count * DISK_SECTOR_SIZE);
    return 0;
}

static int disk_write(const uint8_t *data, uint32_t sector, uint32_t count)
{
    memcpy(&disk_memory[sector * DISK_SECTOR_SIZE], data,
           count * DISK_SECTOR_SIZE);
    return 0;
}

static int disk_ioctl(uint8_t cmd, void *buff)
{
    switch (cmd) {
    case DISK_IOCTL_GET_SECTOR_COUNT:
        *(uint32_t *)buff = DISK_SECTOR_COUNT;
        break;
    case DISK_IOCTL_GET_SECTOR_SIZE:
        *(uint32_t *)buff = DISK_SECTOR_SIZE;
        break;
    default:
        return -EINVAL;
    }
    return 0;
}

/* 磁盘操作结构体 */
static const struct disk_operations disk_ops = {
    .init = disk_init,
    .status = disk_status,
    .read = disk_read,
    .write = disk_write,
    .ioctl = disk_ioctl,
};

void msc_init(void)
{
    /* 注册磁盘设备 */
    disk_access_register("RAM", &disk_ops);
    
    /* 使能 USB */
    usb_enable(NULL);
}
```

## 5. 基础设备测试

### 5.1 设备枚举测试
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>

static void test_usb_device_status(void)
{
    enum usb_dc_status_code status;
    int ret;

    /* 获取设备状态 */
    ret = usb_dc_status(&status);
    if (ret == 0) {
        switch (status) {
        case USB_DC_RESET:
            printk("USB Reset\n");
            break;
        case USB_DC_CONNECTED:
            printk("USB Connected\n");
            break;
        case USB_DC_CONFIGURED:
            printk("USB Configured\n");
            break;
        case USB_DC_DISCONNECTED:
            printk("USB Disconnected\n");
            break;
        case USB_DC_SUSPEND:
            printk("USB Suspended\n");
            break;
        case USB_DC_RESUME:
            printk("USB Resumed\n");
            break;
        default:
            printk("USB Unknown state\n");
            break;
        }
    }
}
```

### 5.2 端点测试
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>

#define TEST_EP_OUT 0x01
#define TEST_EP_IN  0x81
#define MAX_PACKET  64

static void test_usb_endpoints(void)
{
    uint8_t data[MAX_PACKET];
    int ret;

    /* 配置端点 */
    ret = usb_dc_ep_configure(TEST_EP_OUT,
                             USB_DC_EP_BULK,
                             MAX_PACKET);
    if (ret != 0) {
        return;
    }

    ret = usb_dc_ep_configure(TEST_EP_IN,
                             USB_DC_EP_BULK,
                             MAX_PACKET);
    if (ret != 0) {
        return;
    }

    /* 使能端点 */
    usb_dc_ep_enable(TEST_EP_OUT);
    usb_dc_ep_enable(TEST_EP_IN);

    /* 测试数据传输 */
    memset(data, 0xAA, sizeof(data));
    ret = usb_dc_ep_write(TEST_EP_IN, data,
                         sizeof(data), NULL);
    if (ret < 0) {
        printk("Failed to write to EP\n");
    }
}
```

## 6. 最佳实践

1. 端点配置：
   - 合理分配端点资源
   - 使用适当的端点类型（控制、批量、中断、同步）
   - 设置合适的最大包大小

2. 描述符设计：
   - 确保描述符结构正确
   - 提供清晰的设备标识信息
   - 正确设置配置属性

3. 电源管理：
   - 实现正确的挂起/恢复处理
   - 遵守 USB 总线供电规范
   - 合理使用自供电/总线供电模式

4. 错误处理：
   - 实现完整的错误检查
   - 提供适当的错误恢复机制
   - 记录关键错误信息

5. 性能优化：
   - 使用适当的缓冲区大小
   - 实现高效的数据传输机制
   - 避免不必要的数据拷贝

6. 调试支持：
   - 添加适当的调试日志
   - 实现状态监控机制
   - 提供测试接口

7. 兼容性：
   - 遵循 USB 规范
   - 测试不同主机系统
   - 考虑向后兼容性

8. 安全性：
   - 实现适当的访问控制
   - 保护敏感数据
   - 防止缓冲区溢出