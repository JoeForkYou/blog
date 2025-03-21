---
abbrlink: 49
title: composite_device
date: '2025-03-21 20:49:56'
---
# Zephyr USB 复合设备指南

## 1. 复合设备基础

复合设备是指在单个物理 USB 设备中包含多个功能类的设备。例如，一个设备可以同时实现 CDC-ACM（串口）、HID（键盘/鼠标）和 MSC（存储）功能。

### 1.1 复合设备配置 (prj.conf)
```plaintext
# USB 设备栈基础配置
CONFIG_USB_DEVICE_STACK=y
CONFIG_USB_COMPOSITE_DEVICE=y

# 复合设备类配置
CONFIG_USB_CDC_ACM=y
CONFIG_USB_HID=y
CONFIG_USB_MASS_STORAGE=y

# 复合设备描述符配置
CONFIG_USB_DEVICE_MANUFACTURER="Zephyr"
CONFIG_USB_DEVICE_PRODUCT="Composite Device"
CONFIG_USB_DEVICE_VID=0x2FE3
CONFIG_USB_DEVICE_PID=0x0200
CONFIG_USB_DEVICE_SN="0123456789ABCDEF"

# 接口配置
CONFIG_USB_MAX_NUM_INTERFACES=8
CONFIG_USB_COMPOSITE_BUFFER_SIZE=512
```

### 1.2 复合设备描述符结构
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/usb_common.h>
#include <zephyr/usb/class/usb_cdc.h>
#include <zephyr/usb/class/usb_hid.h>
#include <zephyr/usb/class/usb_msc.h>

/* 配置描述符结构 */
struct usb_composite_desc {
    struct usb_cfg_descriptor cfg_desc;
    
    /* CDC-ACM 接口描述符 */
    struct usb_if_descriptor if0;
    struct usb_ep_descriptor if0_in_ep;
    struct usb_ep_descriptor if0_out_ep;
    
    /* HID 接口描述符 */
    struct usb_if_descriptor if1;
    struct usb_ep_descriptor if1_in_ep;
    
    /* MSC 接口描述符 */
    struct usb_if_descriptor if2;
    struct usb_ep_descriptor if2_in_ep;
    struct usb_ep_descriptor if2_out_ep;
} __packed;
```

## 2. 复合设备实现

### 2.1 设备描述符定义
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/usb_common.h>

/* 设备描述符 */
static const struct usb_device_descriptor dev_desc = {
    .bLength = sizeof(struct usb_device_descriptor),
    .bDescriptorType = USB_DESC_DEVICE,
    .bcdUSB = sys_cpu_to_le16(USB_2_0),
    .bDeviceClass = USB_BCC_MISCELLANEOUS,
    .bDeviceSubClass = 0x02,
    .bDeviceProtocol = 0x01,
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
static struct usb_composite_desc desc = {
    .cfg_desc = {
        .bLength = sizeof(struct usb_cfg_descriptor),
        .bDescriptorType = USB_DESC_CONFIGURATION,
        .wTotalLength = 0, /* 将在运行时计算 */
        .bNumInterfaces = 3, /* CDC + HID + MSC */
        .bConfigurationValue = 1,
        .iConfiguration = 0,
        .bmAttributes = USB_CONFIG_ATT_ONE | USB_CONFIG_ATT_SELF_POWERED,
        .bMaxPower = 50, /* 100 mA */
    },
    /* 各个接口描述符在子系统中定义 */
};

/* 语言 ID 描述符 */
static const uint8_t lang_desc[] = {
    4,
    USB_DESC_STRING,
    0x09, 0x04 /* 英语 */
};

/* 厂商字符串描述符 */
static const uint8_t mfr_desc[] = {
    USB_STRING_DESC_SIZE(CONFIG_USB_DEVICE_MANUFACTURER),
    USB_DESC_STRING,
    'Z', 0, 'e', 0, 'p', 0, 'h', 0, 'y', 0, 'r', 0
};

/* 产品字符串描述符 */
static const uint8_t product_desc[] = {
    USB_STRING_DESC_SIZE(CONFIG_USB_DEVICE_PRODUCT),
    USB_DESC_STRING,
    'C', 0, 'o', 0, 'm', 0, 'p', 0, 'o', 0, 's', 0, 'i', 0, 't', 0, 'e', 0,
    ' ', 0, 'D', 0, 'e', 0, 'v', 0, 'i', 0, 'c', 0, 'e', 0
};

/* 序列号字符串描述符 */
static const uint8_t sn_desc[] = {
    USB_STRING_DESC_SIZE(CONFIG_USB_DEVICE_SN),
    USB_DESC_STRING,
    '0', 0, '1', 0, '2', 0, '3', 0, '4', 0, '5', 0, '6', 0, '7', 0,
    '8', 0, '9', 0, 'A', 0, 'B', 0, 'C', 0, 'D', 0, 'E', 0, 'F', 0
};

/* 描述符列表 */
static const struct usb_desc_header *desc_list[] = {
    (struct usb_desc_header *)&dev_desc,
    (struct usb_desc_header *)&desc,
    NULL,
};

/* 字符串描述符列表 */
static const struct usb_desc_header *string_desc[] = {
    (struct usb_desc_header *)lang_desc,
    (struct usb_desc_header *)mfr_desc,
    (struct usb_desc_header *)product_desc,
    (struct usb_desc_header *)sn_desc,
    NULL,
};
```

### 2.2 复合设备初始化
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>

/* USB 设备状态回调 */
static void composite_status_cb(enum usb_dc_status_code status, const uint8_t *param)
{
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
        break;
    }
}

/* 初始化复合设备 */
int composite_device_init(void)
{
    int ret;

    /* 设置描述符 */
    ret = usb_set_config(desc_list);
    if (ret < 0) {
        printk("Failed to set USB configuration\n");
        return ret;
    }

    /* 设置字符串描述符 */
    ret = usb_enable_strings(string_desc);
    if (ret < 0) {
        printk("Failed to enable USB strings\n");
        return ret;
    }

    /* 注册状态回调 */
    ret = usb_register_status_callback(composite_status_cb);
    if (ret < 0) {
        printk("Failed to register USB status callback\n");
        return ret;
    }

    /* 使能 USB */
    ret = usb_enable(NULL);
    if (ret < 0) {
        printk("Failed to enable USB\n");
        return ret;
    }

    return 0;
}
```

## 3. CDC-ACM 组件实现

### 3.1 CDC-ACM 接口定义
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/class/usb_cdc.h>

/* CDC-ACM 设备配置 */
#define CDC_ACM_INTERFACE_0  0
#define CDC_ACM_INTERFACE_1  1
#define CDC_ACM_ENDPOINT_IN  0x81
#define CDC_ACM_ENDPOINT_OUT 0x01

static struct usb_cdc_acm_config cdc_acm_cfg = {
    .if0 = {
        .bLength = sizeof(struct usb_if_descriptor),
        .bDescriptorType = USB_DESC_INTERFACE,
        .bInterfaceNumber = CDC_ACM_INTERFACE_0,
        .bAlternateSetting = 0,
        .bNumEndpoints = 1,
        .bInterfaceClass = USB_BCC_CDC_CONTROL,
        .bInterfaceSubClass = ACM_SUBCLASS,
        .bInterfaceProtocol = 0,
        .iInterface = 0,
    },
    /* CDC 控制描述符 */
    .if0_header = {
        .bFunctionLength = sizeof(struct cdc_header_descriptor),
        .bDescriptorType = USB_DESC_CS_INTERFACE,
        .bDescriptorSubtype = HEADER_FUNC_DESC,
        .bcdCDC = sys_cpu_to_le16(USB_1_1),
    },
    .if0_cm = {
        .bFunctionLength = sizeof(struct cdc_cm_descriptor),
        .bDescriptorType = USB_DESC_CS_INTERFACE,
        .bDescriptorSubtype = CALL_MANAGEMENT_FUNC_DESC,
        .bmCapabilities = 0x02,
        .bDataInterface = 1,
    },
    .if0_acm = {
        .bFunctionLength = sizeof(struct cdc_acm_descriptor),
        .bDescriptorType = USB_DESC_CS_INTERFACE,
        .bDescriptorSubtype = ACM_FUNC_DESC,
        .bmCapabilities = 0x02,
    },
    .if0_union = {
        .bFunctionLength = sizeof(struct cdc_union_descriptor),
        .bDescriptorType = USB_DESC_CS_INTERFACE,
        .bDescriptorSubtype = UNION_FUNC_DESC,
        .bControlInterface = 0,
        .bSubordinateInterface0 = 1,
    },
    .if0_int_ep = {
        .bLength = sizeof(struct usb_ep_descriptor),
        .bDescriptorType = USB_DESC_ENDPOINT,
        .bEndpointAddress = CDC_ACM_ENDPOINT_IN,
        .bmAttributes = USB_DC_EP_INTERRUPT,
        .wMaxPacketSize = sys_cpu_to_le16(16),
        .bInterval = 10,
    },
    .if1 = {
        .bLength = sizeof(struct usb_if_descriptor),
        .bDescriptorType = USB_DESC_INTERFACE,
        .bInterfaceNumber = CDC_ACM_INTERFACE_1,
        .bAlternateSetting = 0,
        .bNumEndpoints = 2,
        .bInterfaceClass = USB_BCC_CDC_DATA,
        .bInterfaceSubClass = 0,
        .bInterfaceProtocol = 0,
        .iInterface = 0,
    },
    .if1_in_ep = {
        .bLength = sizeof(struct usb_ep_descriptor),
        .bDescriptorType = USB_DESC_ENDPOINT,
        .bEndpointAddress = CDC_ACM_ENDPOINT_IN,
        .bmAttributes = USB_DC_EP_BULK,
        .wMaxPacketSize = sys_cpu_to_le16(64),
        .bInterval = 0,
    },
    .if1_out_ep = {
        .bLength = sizeof(struct usb_ep_descriptor),
        .bDescriptorType = USB_DESC_ENDPOINT,
        .bEndpointAddress = CDC_ACM_ENDPOINT_OUT,
        .bmAttributes = USB_DC_EP_BULK,
        .wMaxPacketSize = sys_cpu_to_le16(64),
        .bInterval = 0,
    },
};
```

### 3.2 CDC-ACM 初始化和数据处理
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/class/usb_cdc.h>

static uint8_t cdc_acm_rx_buf[64];
static uint8_t cdc_acm_tx_buf[64];
static volatile bool cdc_acm_configured;

/* CDC ACM 数据接收回调 */
static void cdc_acm_rx_cb(uint8_t ep, int size, void *priv)
{
    if (size <= 0) {
        return;
    }

    /* 处理接收到的数据 */
    printk("CDC ACM received %d bytes\n", size);

    /* 回显数据 */
    memcpy(cdc_acm_tx_buf, cdc_acm_rx_buf, size);
    usb_transfer(CDC_ACM_ENDPOINT_IN, cdc_acm_tx_buf, size, USB_TRANS_WRITE, NULL, NULL);

    /* 准备接收更多数据 */
    usb_transfer(CDC_ACM_ENDPOINT_OUT, cdc_acm_rx_buf, sizeof(cdc_acm_rx_buf),
                USB_TRANS_READ, cdc_acm_rx_cb, NULL);
}

/* CDC ACM 配置回调 */
static void cdc_acm_configure(const struct device *dev)
{
    cdc_acm_configured = true;

    /* 启动接收 */
    usb_transfer(CDC_ACM_ENDPOINT_OUT, cdc_acm_rx_buf, sizeof(cdc_acm_rx_buf),
                USB_TRANS_READ, cdc_acm_rx_cb, NULL);
}

/* CDC ACM 回调结构体 */
static struct usb_cdc_acm_ops cdc_acm_ops = {
    .configured = cdc_acm_configure,
};

/* 初始化 CDC ACM */
int cdc_acm_init(void)
{
    int ret;

    /* 注册 CDC ACM 设备 */
    ret = usb_cdc_acm_register(&cdc_acm_cfg, &cdc_acm_ops);
    if (ret != 0) {
        printk("Failed to register CDC ACM device\n");
        return ret;
    }

    return 0;
}
```

## 4. HID 组件实现

### 4.1 HID 接口定义
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/class/usb_hid.h>

/* HID 设备配置 */
#define HID_INTERFACE      2
#define HID_ENDPOINT_IN    0x82

/* HID 报告描述符 - 键盘 */
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

static struct usb_hid_config hid_cfg = {
    .interface = {
        .bLength = sizeof(struct usb_if_descriptor),
        .bDescriptorType = USB_DESC_INTERFACE,
        .bInterfaceNumber = HID_INTERFACE,
        .bAlternateSetting = 0,
        .bNumEndpoints = 1,
        .bInterfaceClass = USB_BCC_HID,
        .bInterfaceSubClass = 0,
        .bInterfaceProtocol = 0,
        .iInterface = 0,
    },
    .hid_descriptor = {
        .bLength = sizeof(struct usb_hid_descriptor),
        .bDescriptorType = USB_DESC_HID,
        .bcdHID = sys_cpu_to_le16(USB_1_1),
        .bCountryCode = 0,
        .bNumDescriptors = 1,
        .subdesc[0] = {
            .bDescriptorType = USB_DESC_HID_REPORT,
            .wDescriptorLength = sys_cpu_to_le16(sizeof(hid_report_desc)),
        },
    },
    .ep_in = {
        .bLength = sizeof(struct usb_ep_descriptor),
        .bDescriptorType = USB_DESC_ENDPOINT,
        .bEndpointAddress = HID_ENDPOINT_IN,
        .bmAttributes = USB_DC_EP_INTERRUPT,
        .wMaxPacketSize = sys_cpu_to_le16(8),
        .bInterval = 10,
    },
};
```

### 4.2 HID 初始化和数据处理
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/class/usb_hid.h>

static bool hid_configured;
static struct k_work_delayable hid_work;

/* HID 中断就绪回调 */
static void hid_int_in_ready(const struct device *dev)
{
    printk("HID interrupt endpoint ready\n");
}

/* HID 获取报告回调 */
static int hid_get_report(const struct device *dev, struct usb_setup_packet *setup,
                         int32_t *len, uint8_t **data)
{
    printk("HID Get Report request\n");
    return 0;
}

/* HID 设置报告回调 */
static int hid_set_report(const struct device *dev, struct usb_setup_packet *setup,
                         int32_t *len, uint8_t **data)
{
    printk("HID Set Report request\n");
    return 0;
}

/* HID 回调结构体 */
static const struct hid_ops hid_ops = {
    .get_report = hid_get_report,
    .set_report = hid_set_report,
    .int_in_ready = hid_int_in_ready,
};

/* HID 工作处理函数 */
static void hid_work_handler(struct k_work *work)
{
    static uint8_t report[8];
    static uint8_t key = 0;

    /* 模拟按键 */
    report[0] = key++;
    if (key > 10) {
        key = 0;
    }

    /* 发送报告 */
    hid_int_ep_write(report, sizeof(report), NULL);

    /* 安排下一次发送 */
    k_work_schedule(&hid_work, K_MSEC(1000));
}

/* 初始化 HID */
int hid_init(void)
{
    int ret;

    /* 注册 HID 设备 */
    ret = usb_hid_register_device(hid_report_desc, sizeof(hid_report_desc),
                                 &hid_cfg, &hid_ops);
    if (ret != 0) {
        printk("Failed to register HID device\n");
        return ret;
    }

    /* 初始化工作队列 */
    k_work_init_delayable(&hid_work, hid_work_handler);

    /* 启动周期性发送 */
    k_work_schedule(&hid_work, K_MSEC(1000));

    return 0;
}
```

## 5. MSC 组件实现

### 5.1 MSC 接口定义
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>
#include <zephyr/usb/class/usb_msc.h>

/* MSC 设备配置 */
#define MSC_INTERFACE      3
#define MSC_ENDPOINT_IN    0x83
#define MSC_ENDPOINT_OUT   0x03

static struct usb_msc_config msc_cfg = {
    .interface = {
        .bLength = sizeof(struct usb_if_descriptor),
        .bDescriptorType = USB_DESC_INTERFACE,
        .bInterfaceNumber = MSC_INTERFACE,
        .bAlternateSetting = 0,
        .bNumEndpoints = 2,
        .bInterfaceClass = USB_BCC_MASS_STORAGE,
        .bInterfaceSubClass = SCSI_TRANSPARENT_SUBCLASS,
        .bInterfaceProtocol = BULK_ONLY_PROTOCOL,
        .iInterface = 0,
    },
    .ep_in = {
        .bLength = sizeof(struct usb_ep_descriptor),
        .bDescriptorType = USB_DESC_ENDPOINT,
        .bEndpointAddress = MSC_ENDPOINT_IN,
        .bmAttributes = USB_DC_EP_BULK,
        .wMaxPacketSize = sys_cpu_to_le16(64),
        .bInterval = 0,
    },
    .ep_out = {
        .bLength = sizeof(struct usb_ep_descriptor),
        .bDescriptorType = USB_DESC_ENDPOINT,
        .bEndpointAddress = MSC_ENDPOINT_OUT,
        .bmAttributes = USB_DC_EP_BULK,
        .wMaxPacketSize = sys_cpu_to_le16(64),
        .bInterval = 0,
    },
};
```

### 5.2 MSC 初始化和数据处理
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

/* 初始化 MSC */
int msc_init(void)
{
    int ret;

    /* 注册磁盘设备 */
    ret = disk_access_register("RAM", &disk_ops);
    if (ret != 0) {
        printk("Failed to register disk\n");
        return ret;
    }

    /* 注册 MSC 设备 */
    ret = usb_msc_register(&msc_cfg, "RAM");
    if (ret != 0) {
        printk("Failed to register MSC device\n");
        return ret;
    }

    return 0;
}
```

## 6. 复合设备测试

### 6.1 主程序集成
```c
#include <zephyr/kernel.h>

extern int composite_device_init(void);
extern int cdc_acm_init(void);
extern int hid_init(void);
extern int msc_init(void);

void main(void)
{
    printk("USB Composite Device Example\n");

    /* 初始化各个组件 */
    cdc_acm_init();
    hid_init();
    msc_init();

    /* 初始化复合设备 */
    composite_device_init();

    /* 主循环 */
    while (1) {
        k_sleep(K_SECONDS(1));
    }
}
```

### 6.2 测试验证
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>

void test_composite_device(void)
{
    /* 验证设备状态 */
    enum usb_dc_status_code status;
    usb_dc_status(&status);
    printk("USB Status: %d\n", status);

    /* 验证各个组件功能 */
    if (status == USB_DC_CONFIGURED) {
        printk("Testing CDC ACM...\n");
        /* 发送测试数据 */
        const char *test_str = "CDC ACM Test\r\n";
        usb_write(CDC_ACM_ENDPOINT_IN, test_str, strlen(test_str), NULL);

        printk("Testing HID...\n");
        /* 发送按键报告 */
        uint8_t report[8] = {0x04}; /* 'a' 键 */
        hid_int_ep_write(report, sizeof(report), NULL);

        printk("MSC should be accessible as a disk drive\n");
    }
}
```

## 7. 最佳实践

1. 接口编号分配：
   - 确保每个接口使用唯一的接口编号
   - 考虑使用 IAD (Interface Association Descriptor) 将相关接口组合在一起

2. 端点分配：
   - 合理分配端点资源，避免冲突
   - 考虑不同设备类的端点需求和优先级

3. 描述符设计：
   - 确保描述符结构正确且完整
   - 提供清晰的设备和接口标识信息

4. 资源管理：
   - 合理分配内存和缓冲区
   - 考虑多个设备类之间的资源共享

5. 错误处理：
   - 实现完整的错误检查和恢复机制
   - 考虑一个组件失败对整个设备的影响

6. 电源管理：
   - 实现正确的挂起/恢复处理
   - 考虑不同组件的电源需求

7. 测试验证：
   - 在不同主机系统上测试
   - 验证各个组件的功能和互操作性

8. 兼容性：
   - 遵循 USB 规范和各设备类规范
   - 考虑向后兼容性和不同主机系统的支持