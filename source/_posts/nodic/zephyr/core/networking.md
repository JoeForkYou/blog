---
abbrlink: 6
title: networking
date: '2025-03-21 20:49:56'
---
# Zephyr 网络协议栈

Zephyr RTOS 提供了功能丰富的网络协议栈，支持多种网络技术和协议。本文档将详细介绍 Zephyr 网络子系统的架构和使用方法。

## 网络协议栈概述

### 支持的网络技术

Zephyr 网络协议栈支持多种网络技术：

1. **有线网络**
   - 以太网
   - CAN 总线
   - USB 网络

2. **无线网络**
   - Wi-Fi
   - 蓝牙
   - 蓝牙低功耗 (BLE)
   - IEEE 802.15.4
   - LoRaWAN
   - NB-IoT/LTE-M

### 网络协议

Zephyr 支持多种网络协议：

1. **IPv4/IPv6**
   - 双栈支持
   - 地址自动配置
   - 邻居发现

2. **传输层协议**
   - TCP
   - UDP
   - DTLS

3. **应用层协议**
   - HTTP
   - CoAP
   - MQTT
   - LwM2M
   - SNTP

## 网络配置

### 基本配置

在 `prj.conf` 中启用网络功能：

```
# 启用网络功能
CONFIG_NETWORKING=y

# IPv4 支持
CONFIG_NET_IPV4=y
CONFIG_NET_IPV4_AUTO=y

# IPv6 支持
CONFIG_NET_IPV6=y
CONFIG_NET_IPV6_AUTO_PREFIX=y

# 协议支持
CONFIG_NET_TCP=y
CONFIG_NET_UDP=y

# 网络缓冲区配置
CONFIG_NET_BUF_RX_COUNT=16
CONFIG_NET_BUF_TX_COUNT=16
CONFIG_NET_PKT_RX_COUNT=16
CONFIG_NET_PKT_TX_COUNT=16
```

### 网络接口配置

通过设备树配置网络接口：

```dts
&eth0 {
    status = "okay";
    local-mac-address = [00 00 00 01 02 03];
};
```

## TCP/IP 协议栈

### 套接字 API

Zephyr 提供了兼容 BSD 套接字的 API：

```c
#include <zephyr/net/socket.h>

// 创建套接字
int sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);

// 连接到服务器
struct sockaddr_in addr;
addr.sin_family = AF_INET;
addr.sin_port = htons(80);
inet_pton(AF_INET, "192.168.1.1", &addr.sin_addr);
connect(sock, (struct sockaddr *)&addr, sizeof(addr));

// 发送数据
send(sock, data, len, 0);

// 接收数据
recv(sock, buffer, sizeof(buffer), 0);

// 关闭套接字
close(sock);
```

### TCP 客户端

```c
#include <zephyr/net/socket.h>

void tcp_client(void)
{
    int sock;
    struct sockaddr_in addr;

    // 创建套接字
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (sock < 0) {
        printk("Failed to create socket\n");
        return;
    }

    // 设置服务器地址
    addr.sin_family = AF_INET;
    addr.sin_port = htons(80);
    inet_pton(AF_INET, "192.168.1.1", &addr.sin_addr);

    // 连接服务器
    if (connect(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        printk("Failed to connect\n");
        close(sock);
        return;
    }

    // 发送 HTTP 请求
    const char *request = "GET / HTTP/1.1\r\nHost: 192.168.1.1\r\n\r\n";
    send(sock, request, strlen(request), 0);

    // 接收响应
    char buffer[1024];
    int len = recv(sock, buffer, sizeof(buffer) - 1, 0);
    if (len > 0) {
        buffer[len] = '\0';
        printk("Received: %s\n", buffer);
    }

    // 关闭连接
    close(sock);
}
```

### TCP 服务器

```c
#include <zephyr/net/socket.h>

void tcp_server(void)
{
    int server_sock, client_sock;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);

    // 创建服务器套接字
    server_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (server_sock < 0) {
        printk("Failed to create server socket\n");
        return;
    }

    // 绑定地址
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(8080);
    if (bind(server_sock, (struct sockaddr *)&server_addr,
             sizeof(server_addr)) < 0) {
        printk("Failed to bind server socket\n");
        close(server_sock);
        return;
    }

    // 监听连接
    if (listen(server_sock, 5) < 0) {
        printk("Failed to listen\n");
        close(server_sock);
        return;
    }

    printk("TCP server listening on port 8080\n");

    // 接受连接
    client_sock = accept(server_sock, (struct sockaddr *)&client_addr,
                         &client_addr_len);
    if (client_sock < 0) {
        printk("Failed to accept connection\n");
        close(server_sock);
        return;
    }

    // 处理客户端请求
    char buffer[1024];
    int len = recv(client_sock, buffer, sizeof(buffer) - 1, 0);
    if (len > 0) {
        buffer[len] = '\0';
        printk("Received: %s\n", buffer);

        // 发送响应
        const char *response = "HTTP/1.1 200 OK\r\n\r\nHello from Zephyr!\r\n";
        send(client_sock, response, strlen(response), 0);
    }

    // 关闭连接
    close(client_sock);
    close(server_sock);
}
```

### UDP 通信

```c
#include <zephyr/net/socket.h>

void udp_example(void)
{
    int sock;
    struct sockaddr_in local_addr, remote_addr;
    socklen_t remote_addr_len = sizeof(remote_addr);

    // 创建 UDP 套接字
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        printk("Failed to create UDP socket\n");
        return;
    }

    // 绑定本地地址
    local_addr.sin_family = AF_INET;
    local_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    local_addr.sin_port = htons(5683);
    if (bind(sock, (struct sockaddr *)&local_addr, sizeof(local_addr)) < 0) {
        printk("Failed to bind UDP socket\n");
        close(sock);
        return;
    }

    // 接收数据
    char buffer[1024];
    int len = recvfrom(sock, buffer, sizeof(buffer) - 1, 0,
                       (struct sockaddr *)&remote_addr, &remote_addr_len);
    if (len > 0) {
        buffer[len] = '\0';
        printk("Received: %s\n", buffer);

        // 发送响应
        const char *response = "ACK";
        sendto(sock, response, strlen(response), 0,
               (struct sockaddr *)&remote_addr, remote_addr_len);
    }

    // 关闭套接字
    close(sock);
}
```

## 蓝牙支持

### 蓝牙配置

在 `prj.conf` 中启用蓝牙功能：

```
# 蓝牙核心功能
CONFIG_BT=y

# 蓝牙角色
CONFIG_BT_PERIPHERAL=y  # 外设角色
CONFIG_BT_CENTRAL=y     # 中心角色

# 蓝牙 LE 功能
CONFIG_BT_DEVICE_NAME="Zephyr BLE Device"
CONFIG_BT_DEVICE_APPEARANCE=0
CONFIG_BT_MAX_CONN=5
```

### 蓝牙 LE 外设

```c
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
#include <zephyr/bluetooth/conn.h>
#include <zephyr/bluetooth/uuid.h>
#include <zephyr/bluetooth/gatt.h>

// 广播数据
static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA_BYTES(BT_DATA_UUID16_ALL, 0x0a, 0x18),  // 设备信息服务
};

// 连接回调
static void connected(struct bt_conn *conn, uint8_t err)
{
    if (err) {
        printk("Connection failed (err %u)\n", err);
        return;
    }

    printk("Connected\n");
}

// 断开连接回调
static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    printk("Disconnected (reason %u)\n", reason);
}

// 连接回调结构体
static struct bt_conn_cb conn_callbacks = {
    .connected = connected,
    .disconnected = disconnected,
};

// 初始化蓝牙
void ble_init(void)
{
    int err;

    // 初始化蓝牙
    err = bt_enable(NULL);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return;
    }

    printk("Bluetooth initialized\n");

    // 注册连接回调
    bt_conn_cb_register(&conn_callbacks);

    // 开始广播
    err = bt_le_adv_start(BT_LE_ADV_CONN, ad, ARRAY_SIZE(ad), NULL, 0);
    if (err) {
        printk("Advertising failed to start (err %d)\n", err);
        return;
    }

    printk("Advertising started\n");
}
```

### GATT 服务

```c
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/gatt.h>

// 自定义服务 UUID
#define BT_UUID_CUSTOM_SERVICE_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef0)
#define BT_UUID_CUSTOM_SERVICE BT_UUID_DECLARE_128(BT_UUID_CUSTOM_SERVICE_VAL)

// 特征 UUID
#define BT_UUID_CUSTOM_CHRC_VAL \
    BT_UUID_128_ENCODE(0x12345678, 0x1234, 0x5678, 0x1234, 0x56789abcdef1)
#define BT_UUID_CUSTOM_CHRC BT_UUID_DECLARE_128(BT_UUID_CUSTOM_CHRC_VAL)

// 特征值
static uint8_t custom_value[] = {0x00};

// 读取回调
static ssize_t read_custom(struct bt_conn *conn,
                          const struct bt_gatt_attr *attr,
                          void *buf, uint16_t len, uint16_t offset)
{
    const uint8_t *value = attr->user_data;

    return bt_gatt_attr_read(conn, attr, buf, len, offset, value,
                            sizeof(custom_value));
}

// 写入回调
static ssize_t write_custom(struct bt_conn *conn,
                           const struct bt_gatt_attr *attr,
                           const void *buf, uint16_t len, uint16_t offset,
                           uint8_t flags)
{
    uint8_t *value = attr->user_data;

    if (offset + len > sizeof(custom_value)) {
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_OFFSET);
    }

    memcpy(value + offset, buf, len);
    printk("Value updated: %u\n", *value);

    return len;
}

// 定义 GATT 服务
BT_GATT_SERVICE_DEFINE(custom_svc,
    BT_GATT_PRIMARY_SERVICE(BT_UUID_CUSTOM_SERVICE),
    BT_GATT_CHARACTERISTIC(BT_UUID_CUSTOM_CHRC,
                         BT_GATT_CHRC_READ | BT_GATT_CHRC_WRITE,
                         BT_GATT_PERM_READ | BT_GATT_PERM_WRITE,
                         read_custom, write_custom, custom_value),
);
```

## IEEE 802.15.4

### 配置 802.15.4

在 `prj.conf` 中启用 IEEE 802.15.4 功能：

```
# 启用 IEEE 802.15.4
CONFIG_NET_L2_IEEE802154=y
CONFIG_NET_L2_IEEE802154_SHELL=y

# 配置 IEEE 802.15.4 参数
CONFIG_IEEE802154_CHANNEL=26
CONFIG_IEEE802154_PAN_ID=0xabcd
```

### 使用 802.15.4

```c
#include <zephyr/net/ieee802154_mgmt.h>

// 配置 IEEE 802.15.4 接口
void configure_ieee802154(void)
{
    struct net_if *iface;
    struct ieee802154_req_params params;

    // 获取 IEEE 802.15.4 接口
    iface = net_if_get_first_by_type(&NET_L2_GET_NAME(IEEE802154));
    if (!iface) {
        printk("No IEEE 802.15.4 interface found\n");
        return;
    }

    // 设置 PAN ID
    params.pan_id = 0xabcd;
    net_mgmt(NET_REQUEST_IEEE802154_SET_PAN_ID, iface, &params, sizeof(params));

    // 设置信道
    params.channel = 26;
    net_mgmt(NET_REQUEST_IEEE802154_SET_CHANNEL, iface, &params, sizeof(params));

    // 启动接口
    net_if_up(iface);
}
```

## LoRaWAN

### 配置 LoRaWAN

在 `prj.conf` 中启用 LoRaWAN 功能：

```
# 启用 LoRaWAN
CONFIG_LORA=y
CONFIG_LORAWAN=y

# LoRaWAN 配置
CONFIG_LORAWAN_REGION_EU868=y
```

### 使用 LoRaWAN

```c
#include <zephyr/lorawan/lorawan.h>

// LoRaWAN 参数
static uint8_t dev_eui[] = {0x00, 0x01, 0x02, 0x03, 0x04, 0x05, 0x06, 0x07};
static uint8_t app_eui[] = {0x01, 0x01, 0x01, 0x01, 0x01, 0x01, 0x01, 0x01};
static uint8_t app_key[] = {0x2B, 0x7E, 0x15, 0x16, 0x28, 0xAE, 0xD2, 0xA6,
                           0xAB, 0xF7, 0x15, 0x88, 0x09, 0xCF, 0x4F, 0x3C};

// LoRaWAN 回调
static void dl_callback(uint8_t port, bool data_pending,
                       uint8_t *data, uint16_t len)
{
    printk("LoRaWAN downlink: port %d, pending %d, len %d\n",
           port, data_pending, len);
}

// 初始化 LoRaWAN
void lorawan_init(void)
{
    int ret;
    struct lorawan_join_config join_cfg;

    // 配置入网参数
    join_cfg.mode = LORAWAN_ACT_OTAA;
    join_cfg.dev_eui = dev_eui;
    join_cfg.otaa.join_eui = app_eui;
    join_cfg.otaa.app_key = app_key;
    join_cfg.otaa.nwk_key = app_key;

    // 初始化 LoRaWAN
    ret = lorawan_start();
    if (ret < 0) {
        printk("LoRaWAN start failed: %d\n", ret);
        return;
    }

    // 注册回调
    lorawan_register_downlink_callback(dl_callback);

    // 入网
    ret = lorawan_join(&join_cfg);
    if (ret < 0) {
        printk("LoRaWAN join failed: %d\n", ret);
        return;
    }

    printk("LoRaWAN join requested\n");
}

// 发送 LoRaWAN 数据
void lorawan_send(void)
{
    uint8_t data[] = {0x01, 0x02, 0x03};
    int ret;

    ret = lorawan_send(2, data, sizeof(data), LORAWAN_MSG_CONFIRMED);
    if (ret < 0) {
        printk("LoRaWAN send failed: %d\n", ret);
    }
}
```

## 网络管理

### 网络接口管理

```c
#include <zephyr/net/net_if.h>
#include <zephyr/net/net_mgmt.h>

// 获取网络接口
struct net_if *iface = net_if_get_default();

// 启用接口
net_if_up(iface);

// 禁用接口
net_if_down(iface);

// 获取 IPv4 地址
char addr_str[NET_IPV4_ADDR_LEN];
struct net_if_addr *if_addr = net_if_ipv4_addr_add(iface, &addr, NET_ADDR_MANUAL, 0);
if (if_addr) {
    net_addr_ntop(AF_INET, &addr, addr_str, sizeof(addr_str));
    printk("IPv4 address: %s\n", addr_str);
}
```

### 网络事件监听

```c
#include <zephyr/net/net_mgmt.h>
#include <zephyr/net/net_event.h>

// 定义事件处理函数
static void net_event_handler(struct net_mgmt_event_callback *cb,
                             uint32_t mgmt_event, struct net_if *iface)
{
    if (mgmt_event == NET_EVENT_IPV4_ADDR_ADD) {
        printk("IPv4 address added\n");
    } else if (mgmt_event == NET_EVENT_IPV4_ADDR_DEL) {
        printk("IPv4 address removed\n");
    }
}

// 注册事件回调
NET_MGMT_REGISTER_EVENT_HANDLER(net_event_cb, net_event_handler,
                              NET_EVENT_IPV4_ADDR_ADD | NET_EVENT_IPV4_ADDR_DEL);
```

## 最佳实践

1. **网络配置**
   - 合理配置网络缓冲区大小
   - 根据应用需求选择合适的协议
   - 优化网络参数以降低功耗

2. **安全性**
   - 使用 TLS/DTLS 保护通信
   - 实施适当的认证机制
   - 定期更新安全凭证

3. **错误处理**
   - 处理网络连接错误
   - 实现重连机制
   - 监控网络状态变化

4. **资源优化**
   - 减少网络流量
   - 使用异步通信
   - 批量处理数据传输

## 常见问题

1. **连接问题**
   - 检查网络配置
   - 验证网络接口状态
   - 确认 IP 地址配置

2. **性能问题**
   - 增加网络缓冲区大小
   - 优化数据传输批量处理
   - 减少不必要的网络请求

3. **功耗问题**
   - 使用低功耗网络模式
   - 减少网络唤醒频率
   - 优化数据传输批量处理

4. **协议兼容性**
   - 确保协议版本兼容
   - 验证协议实现是否完整
   - 测试与不同设备的互操作性

## 总结

Zephyr 网络协议栈提供了丰富的网络功能，支持多种网络技术和协议。通过合理配置和使用这些功能，可以开发出高效、可靠的联网应用。深入理解这些网络接口对于开发高质量的 Zephyr 网络应用至关重要。