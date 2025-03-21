---
abbrlink: 21
title: Zephyr DMA 子系统指南
date: '2025-03-21 20:49:56'
---
# Zephyr DMA 管理指南

## 1. DMA 概述

DMA（直接内存访问）是一种允许外设直接访问系统内存而无需 CPU 干预的技术，能显著提高数据传输效率。Zephyr RTOS 提供了统一的 DMA API，支持各种硬件平台的 DMA 控制器。

## 2. DMA 配置

### 2.1 基本配置 (prj.conf)

```plaintext
# DMA 支持
CONFIG_DMA=y
CONFIG_DMA_64BIT=y  # 如果需要 64 位 DMA 支持

# 特定控制器支持（根据硬件选择）
CONFIG_DMA_STM32=y  # STM32 DMA 控制器
# 或
CONFIG_DMA_NRFX=y   # Nordic nRF DMA 控制器
# 或
CONFIG_DMA_SAM0=y   # Atmel SAM0 DMA 控制器

# DMA 调试支持
CONFIG_DMA_LOG_LEVEL_DBG=y
```

### 2.2 设备树配置

```dts
&dma0 {
    status = "okay";
};

/* 为特定外设配置 DMA */
&uart0 {
    dmas = <&dma0 0 0>, <&dma0 1 0>;
    dma-names = "tx", "rx";
};
```

## 3. DMA 基本操作

### 3.1 DMA 初始化与配置

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/dma.h>

void dma_example(void)
{
    const struct device *dma_dev = DEVICE_DT_GET(DT_NODELABEL(dma0));
    struct dma_config dma_cfg = {0};
    struct dma_block_config dma_block_cfg = {0};
    int ret;

    if (!device_is_ready(dma_dev)) {
        printk("DMA controller device not ready\n");
        return;
    }

    /* 源数据和目标缓冲区 */
    uint8_t tx_data[1024] = {0};
    uint8_t rx_data[1024] = {0};

    /* 填充源数据 */
    for (int i = 0; i < sizeof(tx_data); i++) {
        tx_data[i] = i & 0xFF;
    }

    /* 配置 DMA 传输块 */
    dma_block_cfg.block_size = sizeof(tx_data);
    dma_block_cfg.source_address = (uint32_t)tx_data;
    dma_block_cfg.dest_address = (uint32_t)rx_data;

    /* 配置 DMA 通道 */
    dma_cfg.channel_direction = MEMORY_TO_MEMORY;
    dma_cfg.source_data_size = 1;
    dma_cfg.dest_data_size = 1;
    dma_cfg.source_burst_length = 1;
    dma_cfg.dest_burst_length = 1;
    dma_cfg.dma_callback = NULL;
    dma_cfg.user_data = NULL;
    dma_cfg.block_count = 1;
    dma_cfg.head_block = &dma_block_cfg;

    /* 配置 DMA 通道 */
    ret = dma_config(dma_dev, 0, &dma_cfg);
    if (ret < 0) {
        printk("Failed to configure DMA channel: %d\n", ret);
        return;
    }

    /* 启动 DMA 传输 */
    ret = dma_start(dma_dev, 0);
    if (ret < 0) {
        printk("Failed to start DMA transfer: %d\n", ret);
        return;
    }

    /* 等待传输完成 */
    k_sleep(K_MSEC(100));

    /* 验证数据 */
    for (int i = 0; i < sizeof(rx_data); i++) {
        if (rx_data[i] != tx_data[i]) {
            printk("Data mismatch at index %d: %u != %u\n",
                   i, rx_data[i], tx_data[i]);
            return;
        }
    }

    printk("DMA transfer completed successfully\n");
}
```

## 4. DMA 回调处理

### 4.1 DMA 回调函数

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/dma.h>

/* 定义信号量 */
K_SEM_DEFINE(dma_sem, 0, 1);

/* DMA 传输完成回调 */
static void dma_callback(const struct device *dma_dev, void *user_data,
                        uint32_t channel, int status)
{
    if (status < 0) {
        printk("DMA transfer error: %d\n", status);
    } else {
        printk("DMA transfer completed\n");
    }

    /* 释放信号量通知传输完成 */
    k_sem_give(&dma_sem);
}

void dma_with_callback(void)
{
    const struct device *dma_dev = DEVICE_DT_GET(DT_NODELABEL(dma0));
    struct dma_config dma_cfg = {0};
    struct dma_block_config dma_block_cfg = {0};
    int ret;

    if (!device_is_ready(dma_dev)) {
        printk("DMA controller device not ready\n");
        return;
    }

    /* 源数据和目标缓冲区 */
    uint8_t tx_data[1024] = {0};
    uint8_t rx_data[1024] = {0};

    /* 填充源数据 */
    for (int i = 0; i < sizeof(tx_data); i++) {
        tx_data[i] = i & 0xFF;
    }

    /* 配置 DMA 传输块 */
    dma_block_cfg.block_size = sizeof(tx_data);
    dma_block_cfg.source_address = (uint32_t)tx_data;
    dma_block_cfg.dest_address = (uint32_t)rx_data;

    /* 配置 DMA 通道 */
    dma_cfg.channel_direction = MEMORY_TO_MEMORY;
    dma_cfg.source_data_size = 1;
    dma_cfg.dest_data_size = 1;
    dma_cfg.source_burst_length = 1;
    dma_cfg.dest_burst_length = 1;
    dma_cfg.dma_callback = dma_callback;
    dma_cfg.user_data = NULL;
    dma_cfg.block_count = 1;
    dma_cfg.head_block = &dma_block_cfg;

    /* 配置 DMA 通道 */
    ret = dma_config(dma_dev, 0, &dma_cfg);
    if (ret < 0) {
        printk("Failed to configure DMA channel: %d\n", ret);
        return;
    }

    /* 启动 DMA 传输 */
    ret = dma_start(dma_dev, 0);
    if (ret < 0) {
        printk("Failed to start DMA transfer: %d\n", ret);
        return;
    }

    /* 等待传输完成 */
    if (k_sem_take(&dma_sem, K_MSEC(1000)) != 0) {
        printk("DMA transfer timed out\n");
        dma_stop(dma_dev, 0);
        return;
    }

    /* 验证数据 */
    for (int i = 0; i < sizeof(rx_data); i++) {
        if (rx_data[i] != tx_data[i]) {
            printk("Data mismatch at index %d: %u != %u\n",
                   i, rx_data[i], tx_data[i]);
            return;
        }
    }

    printk("DMA transfer completed successfully\n");
}
```

## 5. 高级 DMA 功能

### 5.1 链式 DMA 传输

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/dma.h>

void dma_chained_transfer(void)
{
    const struct device *dma_dev = DEVICE_DT_GET(DT_NODELABEL(dma0));
    struct dma_config dma_cfg = {0};
    struct dma_block_config dma_block_cfg[3] = {0};
    int ret;

    if (!device_is_ready(dma_dev)) {
        printk("DMA controller device not ready\n");
        return;
    }

    /* 源数据和目标缓冲区 */
    uint8_t tx_data1[256] = {0};
    uint8_t tx_data2[256] = {0};
    uint8_t tx_data3[256] = {0};
    uint8_t rx_data[768] = {0};

    /* 填充源数据 */
    for (int i = 0; i < sizeof(tx_data1); i++) {
        tx_data1[i] = i & 0xFF;
        tx_data2[i] = (i + 100) & 0xFF;
        tx_data3[i] = (i + 200) & 0xFF;
    }

    /* 配置第一个 DMA 传输块 */
    dma_block_cfg[0].block_size = sizeof(tx_data1);
    dma_block_cfg[0].source_address = (uint32_t)tx_data1;
    dma_block_cfg[0].dest_address = (uint32_t)rx_data;
    dma_block_cfg[0].next_block = &dma_block_cfg[1];

    /* 配置第二个 DMA 传输块 */
    dma_block_cfg[1].block_size = sizeof(tx_data2);
    dma_block_cfg[1].source_address = (uint32_t)tx_data2;
    dma_block_cfg[1].dest_address = (uint32_t)(rx_data + sizeof(tx_data1));
    dma_block_cfg[1].next_block = &dma_block_cfg[2];

    /* 配置第三个 DMA 传输块 */
    dma_block_cfg[2].block_size = sizeof(tx_data3);
    dma_block_cfg[2].source_address = (uint32_t)tx_data3;
    dma_block_cfg[2].dest_address = (uint32_t)(rx_data + sizeof(tx_data1) + sizeof(tx_data2));
    dma_block_cfg[2].next_block = NULL;

    /* 配置 DMA 通道 */
    dma_cfg.channel_direction = MEMORY_TO_MEMORY;
    dma_cfg.source_data_size = 1;
    dma_cfg.dest_data_size = 1;
    dma_cfg.source_burst_length = 1;
    dma_cfg.dest_burst_length = 1;
    dma_cfg.dma_callback = NULL;
    dma_cfg.user_data = NULL;
    dma_cfg.block_count = 3;
    dma_cfg.head_block = &dma_block_cfg[0];

    /* 配置 DMA 通道 */
    ret = dma_config(dma_dev, 0, &dma_cfg);
    if (ret < 0) {
        printk("Failed to configure DMA channel: %d\n", ret);
        return;
    }

    /* 启动 DMA 传输 */
    ret = dma_start(dma_dev, 0);
    if (ret < 0) {
        printk("Failed to start DMA transfer: %d\n", ret);
        return;
    }

    /* 等待传输完成 */
    k_sleep(K_MSEC(100));

    printk("Chained DMA transfer completed\n");
}
```

### 5.2 循环 DMA 传输

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/dma.h>

/* DMA 回调函数 */
static void dma_cyclic_callback(const struct device *dma_dev, void *user_data,
                              uint32_t channel, int status)
{
    static int count = 0;
    
    if (status < 0) {
        printk("DMA transfer error: %d\n", status);
        return;
    }
    
    count++;
    printk("DMA cyclic transfer completed: %d\n", count);
    
    /* 如果达到指定次数，停止 DMA */
    if (count >= 10) {
        dma_stop(dma_dev, channel);
    }
}

void dma_cyclic_transfer(void)
{
    const struct device *dma_dev = DEVICE_DT_GET(DT_NODELABEL(dma0));
    struct dma_config dma_cfg = {0};
    struct dma_block_config dma_block_cfg = {0};
    int ret;

    if (!device_is_ready(dma_dev)) {
        printk("DMA controller device not ready\n");
        return;
    }

    /* 源数据和目标缓冲区 */
    uint8_t tx_data[256] = {0};
    uint8_t rx_data[256] = {0};

    /* 填充源数据 */
    for (int i = 0; i < sizeof(tx_data); i++) {
        tx_data[i] = i & 0xFF;
    }

    /* 配置 DMA 传输块 */
    dma_block_cfg.block_size = sizeof(tx_data);
    dma_block_cfg.source_address = (uint32_t)tx_data;
    dma_block_cfg.dest_address = (uint32_t)rx_data;
    dma_block_cfg.source_reload_en = 1;  /* 启用源地址重载 */
    dma_block_cfg.dest_reload_en = 1;    /* 启用目标地址重载 */

    /* 配置 DMA 通道 */
    dma_cfg.channel_direction = MEMORY_TO_MEMORY;
    dma_cfg.source_data_size = 1;
    dma_cfg.dest_data_size = 1;
    dma_cfg.source_burst_length = 1;
    dma_cfg.dest_burst_length = 1;
    dma_cfg.dma_callback = dma_cyclic_callback;
    dma_cfg.user_data = NULL;
    dma_cfg.block_count = 1;
    dma_cfg.head_block = &dma_block_cfg;
    dma_cfg.cyclic = 1;  /* 启用循环模式 */

    /* 配置 DMA 通道 */
    ret = dma_config(dma_dev, 0, &dma_cfg);
    if (ret < 0) {
        printk("Failed to configure DMA channel: %d\n", ret);
        return;
    }

    /* 启动 DMA 传输 */
    ret = dma_start(dma_dev, 0);
    if (ret < 0) {
        printk("Failed to start DMA transfer: %d\n", ret);
        return;
    }

    /* 等待循环传输完成（由回调函数停止） */
    k_sleep(K_SECONDS(5));

    printk("Cyclic DMA transfer stopped\n");
}
```

## 6. 与外设集成

### 6.1 UART 与 DMA 集成

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/dma.h>
#include <zephyr/drivers/uart.h>

/* DMA 完成回调 */
static void uart_dma_callback(const struct device *dma_dev, void *user_data,
                            uint32_t channel, int status)
{
    printk("UART DMA transfer completed with status: %d\n", status);
}

void uart_dma_example(void)
{
    const struct device *uart_dev = DEVICE_DT_GET(DT_NODELABEL(uart0));
    const struct device *dma_dev = DEVICE_DT_GET(DT_NODELABEL(dma0));
    struct dma_config dma_cfg = {0};
    struct dma_block_config dma_block_cfg = {0};
    int ret;

    if (!device_is_ready(uart_dev) || !device_is_ready(dma_dev)) {
        printk("Devices not ready\n");
        return;
    }

    /* 要发送的数据 */
    char tx_data[] = "Hello, DMA UART!\r\n";

    /* 配置 DMA 传输块 */
    dma_block_cfg.block_size = sizeof(tx_data) - 1;
    dma_block_cfg.source_address = (uint32_t)tx_data;
    dma_block_cfg.dest_address = (uint32_t)UART0_BASE_ADDR + UART_TX_REG_OFFSET;

    /* 配置 DMA 通道 */
    dma_cfg.channel_direction = MEMORY_TO_PERIPHERAL;
    dma_cfg.source_data_size = 1;
    dma_cfg.dest_data_size = 1;
    dma_cfg.source_burst_length = 1;
    dma_cfg.dest_burst_length = 1;
    dma_cfg.dma_callback = uart_dma_callback;
    dma_cfg.user_data = NULL;
    dma_cfg.block_count = 1;
    dma_cfg.head_block = &dma_block_cfg;

    /* 配置 DMA 通道 */
    ret = dma_config(dma_dev, 0, &dma_cfg);
    if (ret < 0) {
        printk("Failed to configure DMA channel: %d\n", ret);
        return;
    }

    /* 启动 DMA 传输 */
    ret = dma_start(dma_dev, 0);
    if (ret < 0) {
        printk("Failed to start DMA transfer: %d\n", ret);
        return;
    }

    /* 等待传输完成 */
    k_sleep(K_MSEC(100));
}
```

## 7. DMA 最佳实践

1. 内存对齐：确保 DMA 缓冲区按照硬件要求对齐，通常为 4 或 8 字节边界。

2. 缓冲区位置：对于某些平台，DMA 缓冲区可能需要放置在特定的内存区域，如 SRAM 而非 Flash。

3. 缓存一致性：在使用 DMA 之前，确保清除或刷新 CPU 缓存，以避免数据不一致问题。

4. 错误处理：实现完善的错误处理机制，包括超时检测和错误恢复。

5. 资源管理：在不需要时释放 DMA 通道，以便其他组件使用。

6. 中断处理：使用回调函数处理 DMA 传输完成事件，避免轮询等待。

7. 功耗优化：在低功耗应用中，考虑 DMA 传输完成后进入低功耗模式。

8. 传输大小：根据硬件特性选择最优的传输大小，通常是 2 的幂次。

9. 并发传输：如果硬件支持，可以使用多个 DMA 通道并行处理不同的数据流。

10. 安全考虑：确保 DMA 不会访问受保护的内存区域，特别是在多任务系统中。
```