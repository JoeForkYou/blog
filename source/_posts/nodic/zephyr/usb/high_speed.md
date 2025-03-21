---
abbrlink: 50
title: high_speed
date: '2025-03-21 20:49:56'
---
# Zephyr USB 高速设备指南

## 1. 高速设备基础

### 1.1 高速设备配置 (prj.conf)
```plaintext
# USB 高速设备支持
CONFIG_USB_DEVICE_STACK=y
CONFIG_USB_DEVICE_HIGH_SPEED=y

# DMA 支持（用于高速传输）
CONFIG_DMA=y
CONFIG_USB_DMA=y

# 缓冲区配置
CONFIG_USB_REQUEST_BUFFER_SIZE=512
CONFIG_USB_MAX_NUM_ENDPOINTS=16
CONFIG_USB_DEVICE_BUF_NUM_HE=8
CONFIG_USB_DEVICE_BUF_NUM_LE=8
```

### 1.2 设备描述符
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>

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

/* 设备限定符描述符 */
static const struct usb_device_qualifier_descriptor dev_qualifier = {
    .bLength = sizeof(struct usb_device_qualifier_descriptor),
    .bDescriptorType = USB_DESC_DEVICE_QUALIFIER,
    .bcdUSB = sys_cpu_to_le16(USB_2_0),
    .bDeviceClass = 0,
    .bDeviceSubClass = 0,
    .bDeviceProtocol = 0,
    .bMaxPacketSize0 = USB_MAX_PACKET_SIZE,
    .bNumConfigurations = 1,
    .bReserved = 0,
};
```

## 2. 高速端点配置

### 2.1 高速端点描述符
```c
/* 高速配置描述符 */
static const struct usb_config_descriptor hs_cfg_desc = {
    .bLength = sizeof(struct usb_config_descriptor),
    .bDescriptorType = USB_DESC_CONFIGURATION,
    .wTotalLength = 0, /* 运行时计算 */
    .bNumInterfaces = 1,
    .bConfigurationValue = 1,
    .iConfiguration = 0,
    .bmAttributes = USB_CONFIG_ATT_ONE | USB_CONFIG_ATT_SELF_POWERED,
    .bMaxPower = 50,
};

/* 高速端点描述符 */
static const struct usb_ep_descriptor hs_ep_desc[] = {
    {
        .bLength = sizeof(struct usb_ep_descriptor),
        .bDescriptorType = USB_DESC_ENDPOINT,
        .bEndpointAddress = 0x81,
        .bmAttributes = USB_DC_EP_BULK,
        .wMaxPacketSize = sys_cpu_to_le16(512), /* 高速批量端点 */
        .bInterval = 0,
    },
    {
        .bLength = sizeof(struct usb_ep_descriptor),
        .bDescriptorType = USB_DESC_ENDPOINT,
        .bEndpointAddress = 0x01,
        .bmAttributes = USB_DC_EP_BULK,
        .wMaxPacketSize = sys_cpu_to_le16(512),
        .bInterval = 0,
    },
};
```

### 2.2 DMA 配置
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/dma.h>

/* DMA 配置结构体 */
struct dma_config dma_cfg = {
    .channel_direction = MEMORY_TO_PERIPHERAL,
    .source_data_size = 4,
    .dest_data_size = 4,
    .source_burst_length = 4,
    .dest_burst_length = 4,
    .dma_callback = NULL,
    .block_count = 1,
};

/* DMA 传输块配置 */
struct dma_block_config dma_block = {
    .source_gather_en = 0,
    .dest_scatter_en = 0,
    .source_addr_adj = DMA_ADDR_ADJ_INCREMENT,
    .dest_addr_adj = DMA_ADDR_ADJ_NO_CHANGE,
};
```

## 3. 高速传输实现

### 3.1 批量传输
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>

#define BULK_EP_IN   0x81
#define BULK_EP_OUT  0x01
#define MAX_PACKET   512

static uint8_t bulk_buf[MAX_PACKET];

/* 批量传输回调 */
static void bulk_out_cb(uint8_t ep, enum usb_dc_ep_cb_status_code status)
{
    if (status != USB_DC_EP_DATA_OUT) {
        return;
    }

    /* 读取数据 */
    int bytes_read = usb_read(ep, bulk_buf, sizeof(bulk_buf));
    if (bytes_read < 0) {
        return;
    }

    /* 处理数据 */
    process_bulk_data(bulk_buf, bytes_read);

    /* 准备接收下一个包 */
    usb_dc_ep_read_continue(ep);
}

/* 初始化批量传输 */
void init_bulk_transfer(void)
{
    /* 配置端点 */
    usb_dc_ep_configure(BULK_EP_IN, USB_DC_EP_BULK, MAX_PACKET);
    usb_dc_ep_configure(BULK_EP_OUT, USB_DC_EP_BULK, MAX_PACKET);

    /* 使能端点 */
    usb_dc_ep_enable(BULK_EP_IN);
    usb_dc_ep_enable(BULK_EP_OUT);

    /* 设置回调 */
    usb_dc_ep_callback_set(BULK_EP_OUT, bulk_out_cb);

    /* 启动接收 */
    usb_dc_ep_read_continue(BULK_EP_OUT);
}

/* 发送批量数据 */
int send_bulk_data(const uint8_t *data, size_t len)
{
    size_t sent = 0;

    while (sent < len) {
        size_t chunk = MIN(len - sent, MAX_PACKET);
        int ret = usb_write(BULK_EP_IN, data + sent, chunk);
        if (ret < 0) {
            return ret;
        }
        sent += chunk;
    }

    return 0;
}
```

### 3.2 DMA 传输
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/dma.h>
#include <zephyr/usb/usb_device.h>

/* DMA 传输完成回调 */
static void dma_callback(const struct device *dma_dev, void *user_data,
                        uint32_t channel, int status)
{
    if (status != 0) {
        return;
    }

    /* 通知传输完成 */
    k_sem_give((struct k_sem *)user_data);
}

/* 使用 DMA 发送数据 */
int send_data_with_dma(const uint8_t *data, size_t len)
{
    const struct device *dma_dev = DEVICE_DT_GET(DT_NODELABEL(dma0));
    struct k_sem dma_sem;
    int ret;

    k_sem_init(&dma_sem, 0, 1);

    /* 配置 DMA */
    dma_cfg.dma_callback = dma_callback;
    dma_cfg.user_data = &dma_sem;
    dma_block.block_size = len;
    dma_block.source_address = (uint32_t)data;
    dma_block.dest_address = USB_EP_FIFO_ADDR(BULK_EP_IN);
    dma_cfg.head_block = &dma_block;

    /* 启动 DMA 传输 */
    ret = dma_config(dma_dev, 0, &dma_cfg);
    if (ret != 0) {
        return ret;
    }

    ret = dma_start(dma_dev, 0);
    if (ret != 0) {
        return ret;
    }

    /* 等待传输完成 */
    ret = k_sem_take(&dma_sem, K_MSEC(1000));
    if (ret != 0) {
        dma_stop(dma_dev, 0);
        return ret;
    }

    return 0;
}
```

## 4. 高速传输优化

### 4.1 缓冲区管理
```c
#include <zephyr/kernel.h>

/* 缓冲区配置 */
#define NUM_BUFFERS 8
#define BUFFER_SIZE 512

/* 缓冲区池 */
static uint8_t buffer_pool[NUM_BUFFERS][BUFFER_SIZE] __aligned(32);
static ATOMIC_DEFINE(buffer_map, NUM_BUFFERS);

/* 获取缓冲区 */
static uint8_t *get_buffer(void)
{
    for (int i = 0; i < NUM_BUFFERS; i++) {
        if (!atomic_test_and_set_bit(buffer_map, i)) {
            return buffer_pool[i];
        }
    }
    return NULL;
}

/* 释放缓冲区 */
static void release_buffer(uint8_t *buffer)
{
    int index = (buffer - (uint8_t *)buffer_pool) / BUFFER_SIZE;
    atomic_clear_bit(buffer_map, index);
}

/* 缓冲区管理示例 */
void buffer_management_example(void)
{
    uint8_t *buf = get_buffer();
    if (buf != NULL) {
        /* 使用缓冲区 */
        memset(buf, 0, BUFFER_SIZE);
        /* ... */
        release_buffer(buf);
    }
}
```

### 4.2 性能优化
```c
#include <zephyr/kernel.h>
#include <zephyr/usb/usb_device.h>

/* 双缓冲传输 */
struct transfer_buffer {
    uint8_t *buffer;
    size_t length;
    bool in_use;
};

static struct transfer_buffer tx_buffers[2];
static int active_buffer;

/* 初始化双缓冲 */
void init_double_buffering(void)
{
    for (int i = 0; i < 2; i++) {
        tx_buffers[i].buffer = get_buffer();
        tx_buffers[i].length = 0;
        tx_buffers[i].in_use = false;
    }
    active_buffer = 0;
}

/* 双缓冲传输 */
int send_data_double_buffered(const uint8_t *data, size_t len)
{
    struct transfer_buffer *curr = &tx_buffers[active_buffer];
    int ret;

    /* 等待当前缓冲区可用 */
    while (curr->in_use) {
        k_yield();
    }

    /* 准备数据 */
    memcpy(curr->buffer, data, len);
    curr->length = len;
    curr->in_use = true;

    /* 发送数据 */
    ret = usb_write(BULK_EP_IN, curr->buffer, curr->length);
    if (ret < 0) {
        curr->in_use = false;
        return ret;
    }

    /* 切换缓冲区 */
    active_buffer = (active_buffer + 1) % 2;

    return 0;
}
```

## 5. 高速设备测试

### 5.1 传输速度测试
```c
#include <zephyr/kernel.h>
#include <zephyr/timing/timing.h>

/* 速度测试参数 */
#define TEST_SIZE (1024 * 1024)  /* 1MB */
#define CHUNK_SIZE 512

/* 测试数据传输速度 */
void test_transfer_speed(void)
{
    uint8_t *test_data = k_malloc(TEST_SIZE);
    uint32_t start_time, end_time;
    int ret;

    if (test_data == NULL) {
        return;
    }

    /* 准备测试数据 */
    for (int i = 0; i < TEST_SIZE; i++) {
        test_data[i] = i & 0xFF;
    }

    /* 开始计时 */
    start_time = k_uptime_get_32();

    /* 发送数据 */
    for (size_t offset = 0; offset < TEST_SIZE; offset += CHUNK_SIZE) {
        size_t chunk = MIN(CHUNK_SIZE, TEST_SIZE - offset);
        ret = send_bulk_data(test_data + offset, chunk);
        if (ret < 0) {
            break;
        }
    }

    /* 结束计时 */
    end_time = k_uptime_get_32();

    /* 计算传输速度 */
    uint32_t duration_ms = end_time - start_time;
    float speed_mbps = (float)TEST_SIZE / duration_ms * 1000 / (1024 * 1024);

    printk("Transfer speed: %.2f MB/s\n", speed_mbps);

    k_free(test_data);
}
```

### 5.2 稳定性测试
```c
#include <zephyr/kernel.h>
#include <zephyr/random/rand32.h>

/* 稳定性测试参数 */
#define TEST_DURATION_SEC 3600  /* 1小时 */
#define MAX_PACKET_SIZE 512
#define MIN_PACKET_SIZE 64

/* 随机数据传输测试 */
void test_stability(void)
{
    uint8_t *test_data = k_malloc(MAX_PACKET_SIZE);
    uint32_t start_time = k_uptime_get_32();
    uint32_t packets_sent = 0;
    uint32_t errors = 0;

    if (test_data == NULL) {
        return;
    }

    while (k_uptime_get_32() - start_time < (TEST_DURATION_SEC * 1000)) {
        /* 生成随机数据大小 */
        size_t size = (sys_rand32_get() %
                      (MAX_PACKET_SIZE - MIN_PACKET_SIZE + 1)) + MIN_PACKET_SIZE;

        /* 生成随机数据 */
        for (size_t i = 0; i < size; i++) {
            test_data[i] = sys_rand32_get() & 0xFF;
        }

        /* 发送数据 */
        int ret = send_bulk_data(test_data, size);
        if (ret < 0) {
            errors++;
        }

        packets_sent++;

        /* 定期打印状态 */
        if (packets_sent % 1000 == 0) {
            printk("Packets sent: %u, Errors: %u\n",
                   packets_sent, errors);
        }

        k_sleep(K_MSEC(1));
    }

    printk("Stability test completed:\n");
    printk("Total packets: %u\n", packets_sent);
    printk("Total errors: %u\n", errors);
    printk("Error rate: %.2f%%\n",
           (float)errors / packets_sent * 100);

    k_free(test_data);
}
```

## 6. 高速设备最佳实践

1. 缓冲区管理：
   - 使用对齐的缓冲区以优化 DMA 传输
   - 实现缓冲区池以避免频繁分配
   - 使用双缓冲或环形缓冲区提高吞吐量

2. DMA 使用：
   - 对大数据传输使用 DMA
   - 正确配置 DMA 通道和优先级
   - 实现适当的错误处理和超时机制

3. 端点配置：
   - 使用最大包大小（512字节）
   - 为批量传输启用双缓冲
   - 适当配置端点类型和属性

4. 性能优化：
   - 最小化中断处理时间
   - 使用零拷贝技术
   - 实现有效的数据包分段和重组

5. 错误处理：
   - 实现完善的错误恢复机制
   - 监控传输错误和重试
   - 提供详细的错误诊断信息

6. 测试验证：
   - 进行长期稳定性测试
   - 测试不同数据大小和传输模式
   - 验证在各种主机系统上的性能

7. 功耗管理：
   - 在空闲时进入低功耗状态
   - 优化 DMA 传输以减少功耗
   - 实现适当的唤醒机制

8. 调试支持：
   - 添加性能计数器
   - 实现详细的日志记录
   - 提供诊断接口