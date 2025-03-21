---
title: 【Zephyr】【一】Zephyr RTOS 示例代码集
tags: zephyr
abbrlink: 25034
date: 2025-03-20 22:09:02
categories: zephyr
---

# Zephyr RTOS 示例代码集

## 1. 基础示例

### 1.0 基础配置
每个示例都需要一个 `prj.conf` 文件来配置项目。以下是各个示例所需的配置：

#### 基础示例 prj.conf
```plaintext
# 控制台输出
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# 日志系统
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3

# 内核配置
CONFIG_KERNEL_BIN_NAME="app"
```

#### 线程示例 prj.conf
```plaintext
# 基础配置
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# 线程配置
CONFIG_THREAD_NAME=y
CONFIG_THREAD_MONITOR=y
CONFIG_THREAD_STACK_INFO=y
```

### 1.1 Hello World
```c
int main(void)
{
    const struct device *uart_dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));

    if (!device_is_ready(uart_dev)) {
        printk("UART device not ready\n");
        return -ENODEV;
    }

    char tx_buf[] = "Hello World!\r\n";
    for (int i = 0; i < sizeof(tx_buf); i++) {
        uart_poll_out(uart_dev, tx_buf[i]);
    }

    return 0;
}
        return ret;
    }

    while (1) {
        gpio_pin_set(dev, PIN, (int)led_is_on);
        led_is_on = !led_is_on;
        k_sleep(K_MSEC(1000));
    }

    return 0;
}
{
    while (1) {
        printk("Thread 1 running\n");
        k_msleep(1000);
    }
}

void thread2_entry(void *p1, void *p2, void *p3)
{
    while (1) {
        printk("Thread 2 running\n");
        k_msleep(1500);
    }
}

int main(void)
{
    k_thread_create(&thread1_data, thread1_stack,
                    STACK_SIZE, thread1_entry,
                    NULL, NULL, NULL,
                    THREAD_PRIORITY, 0, K_NO_WAIT);

    k_thread_create(&thread2_data, thread2_stack,
                    STACK_SIZE, thread2_entry,
                    NULL, NULL, NULL,
                    THREAD_PRIORITY, 0, K_NO_WAIT);

    return 0;
}
```

## 2. 通信示例

### 2.0 通信示例配置

#### 信号量示例 prj.conf
```plaintext
# 基础配置
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# 内核配置
CONFIG_KERNEL_BIN_NAME="app"

# 线程配置
CONFIG_THREAD_NAME=y
CONFIG_THREAD_MONITOR=y
CONFIG_THREAD_STACK_INFO=y
```

#### 消息队列示例 prj.conf
```plaintext
# 基础配置
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# 内核配置
CONFIG_KERNEL_BIN_NAME="app"

# 线程配置
CONFIG_THREAD_NAME=y
CONFIG_THREAD_MONITOR=y
CONFIG_THREAD_STACK_INFO=y

# 消息队列配置
CONFIG_HEAP_MEM_POOL_SIZE=256
```

### 2.1 信号量
```c
#include <zephyr/kernel.h>

#define STACK_SIZE 1024
#define THREAD_PRIORITY 7

K_SEM_DEFINE(my_sem, 0, 1);

K_THREAD_STACK_DEFINE(producer_stack, STACK_SIZE);
K_THREAD_STACK_DEFINE(consumer_stack, STACK_SIZE);

struct k_thread producer_thread_data;
struct k_thread consumer_thread_data;

void producer_entry(void *p1, void *p2, void *p3)
{
    while (1) {
        k_msleep(1000);
        printk("Giving semaphore\n");
        k_sem_give(&my_sem);
    }
}

void consumer_entry(void *p1, void *p2, void *p3)
{
    while (1) {
        printk("Taking semaphore\n");
        k_sem_take(&my_sem, K_FOREVER);
        printk("Got semaphore\n");
    }
}

void main(void)
{
    k_thread_create(&producer_thread_data, producer_stack,
                    STACK_SIZE, producer_entry,
                    NULL, NULL, NULL,
                    THREAD_PRIORITY, 0, K_NO_WAIT);

    k_thread_create(&consumer_thread_data, consumer_stack,
                    STACK_SIZE, consumer_entry,
                    NULL, NULL, NULL,
                    THREAD_PRIORITY, 0, K_NO_WAIT);
}
```

### 2.2 消息队列
```c
#include <zephyr/kernel.h>

#define STACK_SIZE 1024
#define THREAD_PRIORITY 7
#define MAX_MSG_SIZE 32

K_MSGQ_DEFINE(my_msgq, MAX_MSG_SIZE, 10, 4);

K_THREAD_STACK_DEFINE(producer_stack, STACK_SIZE);
K_THREAD_STACK_DEFINE(consumer_stack, STACK_SIZE);

struct k_thread producer_thread_data;
struct k_thread consumer_thread_data;

void producer_entry(void *p1, void *p2, void *p3)
{
    char tx_str[MAX_MSG_SIZE];
    int count = 0;

    while (1) {
        snprintf(tx_str, sizeof(tx_str), "message %d", count++);
        k_msgq_put(&my_msgq, tx_str, K_FOREVER);
        k_msleep(1000);
    }
}

void consumer_entry(void *p1, void *p2, void *p3)
{
    char rx_str[MAX_MSG_SIZE];

    while (1) {
        k_msgq_get(&my_msgq, &rx_str, K_FOREVER);
        printk("Received: %s\n", rx_str);
    }
}

void main(void)
{
    k_thread_create(&producer_thread_data, producer_stack,
                    STACK_SIZE, producer_entry,
                    NULL, NULL, NULL,
                    THREAD_PRIORITY, 0, K_NO_WAIT);

    k_thread_create(&consumer_thread_data, consumer_stack,
                    STACK_SIZE, consumer_entry,
                    NULL, NULL, NULL,
                    THREAD_PRIORITY, 0, K_NO_WAIT);
}
```

## 3. 硬件操作示例

### 3.0 硬件操作配置

#### GPIO示例 prj.conf
```plaintext
# 基础配置
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# GPIO配置
CONFIG_GPIO=y

# 设备树配置
CONFIG_GPIO_INIT_PRIORITY=40
```

#### UART示例 prj.conf
```plaintext
# 基础配置
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# UART配置
CONFIG_UART_INTERRUPT_DRIVEN=y
CONFIG_UART_LINE_CTRL=y
```

### 3.1 GPIO控制
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>

/* 获取LED设备树信息 */
static const struct gpio_dt_spec led = GPIO_DT_SPEC_GET(DT_ALIAS(led0), gpios);

void main(void)
{
    int ret;

    if (!device_is_ready(led.port)) {
        printk("Error: LED device %s is not ready\n", led.port->name);
        return;
    }

    ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT_ACTIVE);
    if (ret < 0) {
        printk("Error %d: Failed to configure LED pin\n", ret);
        return;
    }

    while (1) {
        gpio_pin_toggle_dt(&led);
        k_msleep(1000);
    }
}
```

### 3.2 UART通信
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/uart.h>

void main(void)
{
    const struct device *const uart_dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));

    if (!device_is_ready(uart_dev)) {
        printk("UART device not ready\n");
        return;
    }

    char tx_buf[] = "Hello World!\r\n";
    for (int i = 0; i < sizeof(tx_buf); i++) {
        uart_poll_out(uart_dev, tx_buf[i]);
    }
}
```

## 4. 传感器示例

### 4.0 传感器配置

#### 温度传感器示例 prj.conf
```plaintext
# 基础配置
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# I2C配置（如果传感器使用I2C）
CONFIG_I2C=y

# 传感器配置
CONFIG_SENSOR=y
CONFIG_BME280=y
CONFIG_BME280_MODE_FORCED=y
```

### 4.1 温度传感器
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>

int main(void)
{
    const struct device *const dev = DEVICE_DT_GET_ANY(bosch_bme280);

    if (!device_is_ready(dev)) {
        printk("Device %s is not ready\n", dev->name);
        return -ENODEV;
    }

    while (1) {
        struct sensor_value temp;
        
        int ret = sensor_sample_fetch(dev);
        if (ret < 0) {
            printk("sensor_sample_fetch failed: %d\n", ret);
            continue;
        }

        ret = sensor_channel_get(dev, SENSOR_CHAN_AMBIENT_TEMP, &temp);
        if (ret < 0) {
            printk("sensor_channel_get failed: %d\n", ret);
            continue;
        }

        printk("Temperature: %.2f °C\n", sensor_value_to_double(&temp));
        k_msleep(1000);
    }

    return 0;
}
```

## 5. 网络示例

### 5.0 网络配置

#### TCP Echo服务器示例 prj.conf
```plaintext
# 基础配置
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# 网络配置
CONFIG_NETWORKING=y
CONFIG_NET_IPV4=y
CONFIG_NET_IPV6=y
CONFIG_NET_TCP=y
CONFIG_NET_SOCKETS=y
CONFIG_NET_SOCKETS_POSIX_NAMES=y

# 网络缓冲区配置
CONFIG_NET_PKT_RX_COUNT=16
CONFIG_NET_PKT_TX_COUNT=16
CONFIG_NET_BUF_RX_COUNT=64
CONFIG_NET_BUF_TX_COUNT=64

# 网络shell（可选，用于调试）
CONFIG_NET_SHELL=y

# DHCP客户端（可选）
CONFIG_NET_DHCPV4=y

# 日志配置
CONFIG_NET_LOG=y
CONFIG_NET_SOCKETS_LOG_LEVEL_DBG=y
```

### 5.1 TCP Echo服务器
```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>
#include <zephyr/net/net_ip.h>

#define PORT 4242
#define BUFFER_SIZE 1024

int main(void)
{
    int serv_sock, client_sock;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    char rx_buf[BUFFER_SIZE];
    int ret;

    /* 创建socket */
    serv_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (serv_sock < 0) {
        printk("Failed to create socket: %d\n", errno);
        return -errno;
    }

    /* 配置服务器地址 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(PORT);

    ret = bind(serv_sock, (struct sockaddr *)&server_addr,
               sizeof(server_addr));
    if (ret < 0) {
        printk("Failed to bind: %d\n", errno);
        close(serv_sock);
        return -errno;
    }

    ret = listen(serv_sock, 5);
    if (ret < 0) {
        printk("Failed to listen: %d\n", errno);
        close(serv_sock);
        return -errno;
    }

    printk("TCP server listening on port %d\n", PORT);

    while (1) {
        client_sock = accept(serv_sock, (struct sockaddr *)&client_addr,
                           &client_addr_len);
        if (client_sock < 0) {
            printk("Accept failed: %d\n", errno);
            continue;
        }

        printk("Client connected\n");

        while (1) {
            ret = recv(client_sock, rx_buf, sizeof(rx_buf), 0);
            if (ret < 0) {
                printk("Receive failed: %d\n", errno);
                break;
            }
            if (ret == 0) {
                printk("Client disconnected\n");
                break;
            }

            ret = send(client_sock, rx_buf, ret, 0);
            if (ret < 0) {
                printk("Send failed: %d\n", errno);
                break;
            }
        }

        close(client_sock);
    }

    close(serv_sock);
    return 0;
}
```

## 6. 蓝牙示例

### 6.0 蓝牙配置

#### BLE广播示例 prj.conf
```plaintext
# 基础配置
CONFIG_SERIAL=y
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y

# 蓝牙配置
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_DEVICE_NAME="Zephyr"
CONFIG_BT_DEVICE_NAME_DYNAMIC=n
CONFIG_BT_DEVICE_APPEARANCE=0
CONFIG_BT_MAX_CONN=1

# 蓝牙服务配置
CONFIG_BT_DIS=y
CONFIG_BT_DIS_PNP=n
CONFIG_BT_DIS_MODEL="Zephyr Model"
CONFIG_BT_DIS_MANUF="Zephyr Manufacturer"

# 调试配置（可选）
#CONFIG_BT_DEBUG_LOG=y
```

### 6.1 BLE广播

#### prj.conf
```plaintext
# 基础配置
CONFIG_SERIAL=y
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y

# 蓝牙配置
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_DEVICE_NAME="Zephyr Test"

# 调试配置（可选）
CONFIG_BT_DEBUG_LOG=y
```

#### src/main.c
```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>

static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA_BYTES(BT_DATA_NAME_COMPLETE, 'Z', 'e', 'p', 'h', 'y', 'r', ' ', 'T', 'e', 's', 't'),
};

void start_advertising(void)
{
    int err;

    err = bt_le_adv_start(BT_LE_ADV_NCONN, ad, ARRAY_SIZE(ad), NULL, 0);
    if (err) {
        printk("Advertising failed to start (err %d)\n", err);
        return;
    }

    printk("Advertising successfully started\n");
}

int main(void)
{
    int err;

    printk("Starting Bluetooth Peripheral example\n");

    /* 初始化蓝牙 */
    err = bt_enable(NULL);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return err;
    }

    printk("Bluetooth initialized\n");

    /* 开始广播 */
    start_advertising();

    return 0;
}
```
CONFIG_BT_DIS=y
CONFIG_BT_DIS_PNP=n
CONFIG_BT_DIS_MODEL="Zephyr Model"
CONFIG_BT_DIS_MANUF="Zephyr Manufacturer"

# 调试配置（可选）
CONFIG_BT_DEBUG_LOG=y
```

#### src/main.c
```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
#include <zephyr/bluetooth/conn.h>
#include <zephyr/bluetooth/uuid.h>
#include <zephyr/bluetooth/gatt.h>

static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR),
    BT_DATA_BYTES(BT_DATA_UUID16_ALL, BT_UUID_16_ENCODE(BT_UUID_DIS_VAL)),
};

static const struct bt_data sd[] = {
    BT_DATA(BT_DATA_NAME_COMPLETE, CONFIG_BT_DEVICE_NAME, sizeof(CONFIG_BT_DEVICE_NAME) - 1),
};

static void bt_ready(int err)
{
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return;
    }

    printk("Bluetooth initialized\n");

    /* 开始广播 */
    err = bt_le_adv_start(BT_LE_ADV_CONN, ad, ARRAY_SIZE(ad), NULL, 0);
    if (err) {
        printk("Advertising failed to start (err %d)\n", err);
        return;
    }

    printk("Advertising successfully started\n");
}

static void connected(struct bt_conn *conn, uint8_t err)
{
    if (err) {
        printk("Connection failed (err 0x%02x)\n", err);
    } else {
        printk("Connected\n");
    }
}

static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    printk("Disconnected (reason 0x%02x)\n", reason);
}

BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
};

int main(void)
{
    int err;

    printk("Starting Bluetooth Peripheral example\n");

    /* 初始化蓝牙 */
    err = bt_enable(bt_ready);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return err;
    }

    /* 等待事件 */
    while (1) {
        k_msleep(1000);
    }

    return 0;
}
```

### 6.2 BLE GATT服务

#### prj.conf
```plaintext
# 基础配置
CONFIG_SERIAL=y
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y

# 蓝牙配置
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_DEVICE_NAME="Zephyr GATT"
CONFIG_BT_DEVICE_APPEARANCE=0
CONFIG_BT_MAX_CONN=1

# GATT配置
CONFIG_BT_GATT_DYNAMIC_DB=y
CONFIG_BT_GATT_SERVICE_CHANGED=y

# 调试配置
CONFIG_BT_DEBUG_LOG=y
```

#### src/main.c
```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
#include <zephyr/bluetooth/conn.h>
#include <zephyr/bluetooth/uuid.h>
#include <zephyr/bluetooth/gatt.h>

/* 自定义服务 UUID */
#define BT_UUID_CUSTOM_SERVICE_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef0)

/* 自定义特征 UUID */
#define BT_UUID_CUSTOM_CHRC_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef1)

static struct bt_uuid_128 custom_service_uuid = BT_UUID_INIT_128(
    BT_UUID_CUSTOM_SERVICE_VAL);
static struct bt_uuid_128 custom_characteristic_uuid = BT_UUID_INIT_128(
    BT_UUID_CUSTOM_CHRC_VAL);

static uint8_t custom_value[] = { 0x00 };

static ssize_t read_custom_characteristic(struct bt_conn *conn,
                                        const struct bt_gatt_attr *attr,
                                        void *buf,
                                        uint16_t len,
                                        uint16_t offset)
{
    return bt_gatt_attr_read(conn, attr, buf, len, offset,
                            custom_value, sizeof(custom_value));
}

static ssize_t write_custom_characteristic(struct bt_conn *conn,
                                         const struct bt_gatt_attr *attr,
                                         const void *buf,
                                         uint16_t len,
                                         uint16_t offset,
                                         uint8_t flags)
{
    if (offset + len > sizeof(custom_value)) {
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_OFFSET);
    }

    memcpy(custom_value + offset, buf, len);
    printk("Received data: 0x%02x\n", custom_value[0]);

    return len;
}

/* GATT服务定义 */
BT_GATT_SERVICE_DEFINE(custom_svc,
    BT_GATT_PRIMARY_SERVICE(&custom_service_uuid),
    BT_GATT_CHARACTERISTIC(&custom_characteristic_uuid.uuid,
                          BT_GATT_CHRC_READ | BT_GATT_CHRC_WRITE,
                          BT_GATT_PERM_READ | BT_GATT_PERM_WRITE,
                          read_custom_characteristic,
                          write_custom_characteristic,
                          custom_value),
);

static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA(BT_DATA_NAME_COMPLETE, CONFIG_BT_DEVICE_NAME, sizeof(CONFIG_BT_DEVICE_NAME) - 1),
    BT_DATA_BYTES(BT_DATA_UUID128_ALL, BT_UUID_CUSTOM_SERVICE_VAL),
};

static void connected(struct bt_conn *conn, uint8_t err)
{
    if (err) {
        printk("Connection failed (err 0x%02x)\n", err);
    } else {
        printk("Connected\n");
    }
}

static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    printk("Disconnected (reason 0x%02x)\n", reason);
}

BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
};

static void bt_ready(int err)
{
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return;
    }

    printk("Bluetooth initialized\n");

    /* 开始广播 */
    err = bt_le_adv_start(BT_LE_ADV_CONN, ad, ARRAY_SIZE(ad), sd, ARRAY_SIZE(sd));
    if (err) {
        printk("Advertising failed to start (err %d)\n", err);
        return;
    }

    printk("Advertising successfully started\n");
}

int main(void)
{
    int err;

    printk("Starting Bluetooth GATT Service example\n");

    /* 初始化蓝牙 */
    err = bt_enable(bt_ready);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return err;
    }

    /* 等待事件 */
    while (1) {
        k_msleep(1000);
    }

    return 0;
}
```

#### prj.conf
```plaintext
# 基础配置
CONFIG_SERIAL=y
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y

# 蓝牙配置
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_DEVICE_NAME="Zephyr"
CONFIG_BT_DEVICE_APPEARANCE=0
CONFIG_BT_MAX_CONN=1

# 调试配置（可选）
CONFIG_BT_DEBUG_LOG=y
```

#### src/main.c
```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
#include <zephyr/bluetooth/conn.h>
#include <zephyr/bluetooth/uuid.h>
#include <zephyr/bluetooth/gatt.h>

#define DEVICE_NAME     CONFIG_BT_DEVICE_NAME
#define DEVICE_NAME_LEN (sizeof(DEVICE_NAME) - 1)

/* 广播数据 */
static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA(BT_DATA_NAME_COMPLETE, DEVICE_NAME, DEVICE_NAME_LEN)
};

/* 扫描响应数据 */
static const struct bt_data sd[] = {
    BT_DATA_BYTES(BT_DATA_UUID16_ALL, BT_UUID_16_ENCODE(BT_UUID_DIS_VAL))
};

static void bt_ready(int err)
{
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return;
    }

    printk("Bluetooth initialized\n");

    /* 开始广播 */
    err = bt_le_adv_start(BT_LE_ADV_CONN_NAME, ad, ARRAY_SIZE(ad),
                         sd, ARRAY_SIZE(sd));
    if (err) {
        printk("Advertising failed to start (err %d)\n", err);
        return;
    }

    printk("Advertising successfully started\n");
}

static void connected(struct bt_conn *conn, uint8_t err)
{
    if (err) {
        printk("Connection failed (err %u)\n", err);
        return;
    }

    printk("Connected\n");
}

static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    printk("Disconnected (reason %u)\n", reason);
}

BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
};

int main(void)
{
    int err;

    printk("Starting Bluetooth Peripheral example\n");

    /* 初始化蓝牙 */
    err = bt_enable(bt_ready);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return err;
    }

    /* 无限循环等待事件 */
    while (1) {
        k_msleep(1000);
    }

    return 0;
}
```

#### CMakeLists.txt
```cmake
# SPDX-License-Identifier: Apache-2.0

cmake_minimum_required(VERSION 3.20.0)
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(bluetooth_peripheral)

target_sources(app PRIVATE src/main.c)
```

### 6.2 BLE服务示例

#### src/main.c
```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
#include <zephyr/bluetooth/conn.h>
#include <zephyr/bluetooth/uuid.h>
#include <zephyr/bluetooth/gatt.h>

#define DEVICE_NAME     CONFIG_BT_DEVICE_NAME
#define DEVICE_NAME_LEN (sizeof(DEVICE_NAME) - 1)

/* 自定义服务 UUID */
#define BT_UUID_CUSTOM_SERVICE_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef0)

#define BT_UUID_CUSTOM_CHAR_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef1)

static struct bt_uuid_128 custom_service_uuid = BT_UUID_INIT_128(
    BT_UUID_CUSTOM_SERVICE_VAL);
static struct bt_uuid_128 custom_char_uuid = BT_UUID_INIT_128(
    BT_UUID_CUSTOM_CHAR_VAL);

static uint8_t custom_value[] = { 0x00 };

/* 广播数据 */
static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA(BT_DATA_NAME_COMPLETE, DEVICE_NAME, DEVICE_NAME_LEN),
    BT_DATA_BYTES(BT_DATA_UUID128_ALL, BT_UUID_CUSTOM_SERVICE_VAL)
};

/* 特征值读取回调 */
static ssize_t read_custom_value(struct bt_conn *conn,
                               const struct bt_gatt_attr *attr,
                               void *buf, uint16_t len, uint16_t offset)
{
    return bt_gatt_attr_read(conn, attr, buf, len, offset,
                            custom_value, sizeof(custom_value));
}

/* 特征值写入回调 */
static ssize_t write_custom_value(struct bt_conn *conn,
                                const struct bt_gatt_attr *attr,
                                const void *buf, uint16_t len,
                                uint16_t offset, uint8_t flags)
{
    if (offset + len > sizeof(custom_value)) {
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_OFFSET);
    }

    memcpy(custom_value + offset, buf, len);
    printk("Value updated: %02x\n", custom_value[0]);

    return len;
}

/* 定义GATT服务 */
BT_GATT_SERVICE_DEFINE(custom_svc,
    BT_GATT_PRIMARY_SERVICE(&custom_service_uuid),
    BT_GATT_CHARACTERISTIC(&custom_char_uuid.uuid,
                          BT_GATT_CHRC_READ | BT_GATT_CHRC_WRITE,
                          BT_GATT_PERM_READ | BT_GATT_PERM_WRITE,
                          read_custom_value, write_custom_value,
                          custom_value),
);

static void connected(struct bt_conn *conn, uint8_t err)
{
    if (err) {
        printk("Connection failed (err %u)\n", err);
        return;
    }

    printk("Connected\n");
}

static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    printk("Disconnected (reason %u)\n", reason);
}

BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
};

static void bt_ready(int err)
{
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return;
    }

    printk("Bluetooth initialized\n");

    /* 开始广播 */
    err = bt_le_adv_start(BT_LE_ADV_CONN_NAME, ad, ARRAY_SIZE(ad),
                         NULL, 0);
    if (err) {
        printk("Advertising failed to start (err %d)\n", err);
        return;
    }

    printk("Advertising successfully started\n");
}

int main(void)
{
    int err;

    printk("Starting Bluetooth GATT Service example\n");

    /* 初始化蓝牙 */
    err = bt_enable(bt_ready);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return err;
    }

    /* 无限循环等待事件 */
    while (1) {
        k_msleep(1000);
    }

    return 0;
}
```

## 7. 文件系统示例

### 7.0 文件系统配置

#### 文件操作示例 prj.conf
```plaintext
# 基础配置
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# 文件系统配置
CONFIG_FILE_SYSTEM=y
CONFIG_FILE_SYSTEM_LITTLEFS=y

# Flash驱动配置
CONFIG_FLASH=y
CONFIG_FLASH_MAP=y
CONFIG_FLASH_PAGE_LAYOUT=y

# 日志配置
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3

# 文件系统分区配置示例（需要在设备树中定义具体分区）
CONFIG_FLASH_MAP=y
```

#### 文件系统分区示例 (boards/xxx.overlay)
```dts
/ {
    fstab {
        compatible = "zephyr,fstab";
        lfs1: lfs1 {
            compatible = "zephyr,fstab,littlefs";
            mount-point = "/lfs";
            partition = <&lfs1_part>;
            read-size = <16>;
            prog-size = <16>;
            cache-size = <64>;
            lookahead-size = <32>;
            block-cycles = <512>;
        };
    };
};

&flash0 {
    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        lfs1_part: partition@70000 {
            label = "storage";
            reg = <0x70000 0x10000>;
        };
    };
};
```

### 7.1 文件操作
```c
#include <zephyr/kernel.h>
#include <zephyr/fs/fs.h>

int main(void)
{
    struct fs_file_t file;
    char buf[128];
    ssize_t written, read;
    int ret;

    /* 初始化文件对象 */
    fs_file_t_init(&file);

    /* 写文件 */
    ret = fs_open(&file, "/lfs/test.txt", FS_O_CREATE | FS_O_WRITE);
    if (ret < 0) {
        printk("Failed to open file for writing: %d\n", ret);
        return ret;
    }

    written = fs_write(&file, "Hello, Zephyr!", 13);
    if (written < 0) {
        printk("Failed to write file: %d\n", written);
        fs_close(&file);
        return written;
    }

    ret = fs_close(&file);
    if (ret < 0) {
        printk("Failed to close file: %d\n", ret);
        return ret;
    }

    /* 读文件 */
    ret = fs_open(&file, "/lfs/test.txt", FS_O_READ);
    if (ret < 0) {
        printk("Failed to open file for reading: %d\n", ret);
        return ret;
    }

    read = fs_read(&file, buf, sizeof(buf));
    if (read < 0) {
        printk("Failed to read file: %d\n", read);
        fs_close(&file);
        return read;
    }

    if (read > 0) {
        buf[read] = '\0';
        printk("Read from file: %s\n", buf);
    }

    ret = fs_close(&file);
    if (ret < 0) {
        printk("Failed to close file: %d\n", ret);
        return ret;
    }

    return 0;
}
```