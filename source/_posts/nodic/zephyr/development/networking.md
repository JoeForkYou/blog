---
abbrlink: 17
title: networking
date: '2025-03-21 20:49:56'
---
# Zephyr 网络开发

本文档详细介绍了 Zephyr RTOS 的网络开发功能，包括网络协议栈、网络接口、协议实现以及网络应用开发。

## 网络协议栈

### 基本配置

1. **Kconfig 配置**
```
# 启用网络功能
CONFIG_NETWORKING=y

# IPv4 支持
CONFIG_NET_IPV4=y
CONFIG_NET_IPV4_AUTO_LOCAL_ADDR_SET=y

# IPv6 支持
CONFIG_NET_IPV6=y
CONFIG_NET_IPV6_AUTO_LOCAL_ADDR_SET=y

# TCP 支持
CONFIG_NET_TCP=y
CONFIG_NET_TCP_TIME_WAIT=y

# UDP 支持
CONFIG_NET_UDP=y

# DNS 支持
CONFIG_DNS_RESOLVER=y
CONFIG_DNS_SERVER_IP_ADDRESSES=y
```

2. **网络接口配置**
```c
/* 网络接口数据 */
struct net_if_config {
    struct net_if *iface;
    struct in_addr addr;
    struct in_addr netmask;
    struct in_addr gw;
};

/* 配置网络接口 */
static void setup_network(void)
{
    struct net_if *iface;
    struct net_if_config cfg;

    iface = net_if_get_default();
    if (!iface) {
        return;
    }

    /* 配置 IPv4 地址 */
    net_if_ipv4_addr_add(iface, &cfg.addr,
                         NET_ADDR_MANUAL, 0);

    /* 配置网关 */
    net_if_ipv4_set_gw(iface, &cfg.gw);

    /* 配置子网掩码 */
    net_if_ipv4_set_netmask(iface, &cfg.netmask);
}
```

### 协议栈初始化

```c
/* 网络初始化 */
void init_network(void)
{
    /* 等待网络就绪 */
    struct net_if *iface = net_if_get_default();
    if (!iface) {
        return;
    }

    /* 等待地址配置完成 */
    k_sem_take(&wait_for_addr, K_FOREVER);

    /* 配置 DNS */
    struct dns_resolve_context *ctx = dns_resolve_get_default();
    if (ctx) {
        dns_resolve_init(ctx);
    }
}
```

## 网络接口

### 以太网接口

1. **配置以太网**
```c
/* 以太网配置 */
static void ethernet_init(struct net_if *iface)
{
    /* MAC 地址配置 */
    uint8_t mac[6] = {0x00, 0x11, 0x22, 0x33, 0x44, 0x55};
    net_if_set_link_addr(iface, mac, sizeof(mac),
                        NET_LINK_ETHERNET);

    /* 以太网配置 */
    ethernet_configure(iface);
}

NET_DEVICE_INIT(eth_driver, "ETH_0",
                ethernet_init, NULL,
                &eth_data, &eth_config,
                CONFIG_ETH_INIT_PRIORITY, &eth_api,
                ETHERNET_L2, NET_L2_GET_CTX_TYPE(ETHERNET_L2),
                NET_ETH_MTU);
```

2. **使用以太网**
```c
/* 发送数据 */
static int ethernet_send(struct net_if *iface,
                        struct net_pkt *pkt)
{
    /* 准备数据包 */
    net_pkt_set_iface(pkt, iface);

    /* 发送数据 */
    return eth_tx(iface, pkt);
}

/* 接收数据 */
static void ethernet_recv(struct net_if *iface,
                         struct net_pkt *pkt)
{
    /* 处理接收到的数据 */
    if (net_recv_data(iface, pkt) < 0) {
        net_pkt_unref(pkt);
    }
}
```

### 无线接口

1. **配置 Wi-Fi**
```c
/* Wi-Fi 配置 */
static const struct wifi_config wifi_cfg = {
    .ssid = "MyNetwork",
    .psk = "MyPassword",
    .security = WIFI_SECURITY_TYPE_PSK,
};

/* 初始化 Wi-Fi */
static void wifi_init(struct net_if *iface)
{
    /* 配置 Wi-Fi */
    wifi_connect(iface, &wifi_cfg);
}
```

2. **使用 Wi-Fi**
```c
/* 连接回调 */
static void wifi_connect_cb(struct net_if *iface,
                           int status,
                           void *user_data)
{
    if (status == 0) {
        /* 连接成功 */
        k_sem_give(&wifi_connected);
    }
}

/* 扫描回调 */
static void wifi_scan_cb(struct net_if *iface,
                        int status,
                        struct wifi_scan_result *entry,
                        void *user_data)
{
    if (status == 0 && entry) {
        /* 处理扫描结果 */
        LOG_INF("Found SSID: %s", entry->ssid);
    }
}
```

## 协议实现

### TCP 协议

1. **TCP 服务器**
```c
/* TCP 服务器配置 */
static void tcp_server(void)
{
    struct sockaddr_in addr;
    int sock, client;

    /* 创建套接字 */
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (sock < 0) {
        return;
    }

    /* 绑定地址 */
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        close(sock);
        return;
    }

    /* 监听连接 */
    listen(sock, 5);

    while (1) {
        client = accept(sock, NULL, NULL);
        if (client < 0) {
            continue;
        }

        /* 处理客户端连接 */
        handle_client(client);
        close(client);
    }
}
```

2. **TCP 客户端**
```c
/* TCP 客户端配置 */
static void tcp_client(void)
{
    struct sockaddr_in addr;
    int sock;

    /* 创建套接字 */
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (sock < 0) {
        return;
    }

    /* 连接服务器 */
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    inet_pton(AF_INET, SERVER_ADDR, &addr.sin_addr);

    if (connect(sock, (struct sockaddr *)&addr,
                sizeof(addr)) < 0) {
        close(sock);
        return;
    }

    /* 发送和接收数据 */
    communicate(sock);
    close(sock);
}
```

### UDP 协议

1. **UDP 服务器**
```c
/* UDP 服务器配置 */
static void udp_server(void)
{
    struct sockaddr_in addr;
    int sock;

    /* 创建套接字 */
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        return;
    }

    /* 绑定地址 */
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = INADDR_ANY;

    if (bind(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        close(sock);
        return;
    }

    /* 接收数据 */
    while (1) {
        handle_udp_data(sock);
    }
}
```

2. **UDP 客户端**
```c
/* UDP 客户端配置 */
static void udp_client(void)
{
    struct sockaddr_in addr;
    int sock;

    /* 创建套接字 */
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        return;
    }

    /* 设置目标地址 */
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    inet_pton(AF_INET, SERVER_ADDR, &addr.sin_addr);

    /* 发送数据 */
    send_udp_data(sock, &addr);
    close(sock);
}
```

## 网络应用

### HTTP 客户端

1. **基本配置**
```c
/* HTTP 客户端配置 */
static struct http_client_request req = {
    .method = HTTP_GET,
    .url = "/api/data",
    .host = "example.com",
    .protocol = "HTTP/1.1",
    .header_fields = NULL,
};
```

2. **发送请求**
```c
/* 发送 HTTP 请求 */
static void http_get(void)
{
    struct http_client_ctx ctx;
    int ret;

    /* 初始化上下文 */
    http_client_init(&ctx);

    /* 发送请求 */
    ret = http_client_send_req(&ctx, &req, response_cb, NULL, 0);
    if (ret < 0) {
        LOG_ERR("Failed to send HTTP request");
    }
}
```

### MQTT 客户端

1. **基本配置**
```c
/* MQTT 客户端配置 */
static struct mqtt_client client = {
    .client_id = "zephyr_mqtt",
    .broker = &broker,
    .evt_cb = mqtt_evt_handler,
    .protocol_version = MQTT_VERSION_3_1_1,
};
```

2. **连接和发布**
```c
/* MQTT 事件处理 */
static void mqtt_evt_handler(struct mqtt_client *client,
                           const struct mqtt_evt *evt)
{
    switch (evt->type) {
    case MQTT_EVT_CONNACK:
        if (evt->result == 0) {
            /* 连接成功 */
            subscribe_topics();
        }
        break;

    case MQTT_EVT_PUBLISH:
        /* 处理接收到的消息 */
        handle_publish(evt);
        break;
    }
}

/* 发布消息 */
static void publish_message(void)
{
    struct mqtt_publish_param param = {
        .message.topic.qos = MQTT_QOS_1_AT_LEAST_ONCE,
        .message.topic.topic.utf8 = "test/topic",
        .message.topic.topic.size = 10,
        .message.payload.data = "Hello MQTT",
        .message.payload.len = 10,
    };

    mqtt_publish(&client, &param);
}
```

### CoAP 客户端

1. **基本配置**
```c
/* CoAP 客户端配置 */
static struct coap_client client;
static struct coap_packet request;
```

2. **发送请求**
```c
/* 发送 CoAP 请求 */
static void coap_get(void)
{
    uint8_t *data;
    uint16_t id;
    int r;

    /* 准备请求 */
    data = (uint8_t *)k_malloc(MAX_COAP_MSG_LEN);
    if (!data) {
        return;
    }

    r = coap_packet_init(&request, data, MAX_COAP_MSG_LEN,
                         1, COAP_TYPE_CON, 8, coap_next_token(),
                         COAP_METHOD_GET, id);
    if (r < 0) {
        k_free(data);
        return;
    }

    /* 发送请求 */
    r = coap_packet_send(&request, sock, &server_addr);
    k_free(data);
}
```

## 调试技巧

### 1. 网络调试

```c
/* 启用网络日志 */
#define NET_LOG_ENABLED 1
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(net_app, LOG_LEVEL_DBG);

/* 数据包跟踪 */
NET_PKT_DATA_ACCESS_DEFINE(pkt_data);
struct net_pkt *pkt = net_pkt_alloc_with_buffer(iface,
                                               size,
                                               AF_INET,
                                               IPPROTO_TCP,
                                               K_NO_WAIT);
```

### 2. 抓包分析

```bash
# 使用 Wireshark 分析
sudo ip link set dev zeth up
sudo wireshark -i zeth

# 使用 tcpdump 抓包
sudo tcpdump -i zeth -w capture.pcap
```

### 3. 性能分析

```c
/* 网络性能统计 */
struct net_stats stats;
net_mgmt(NET_REQUEST_STATS_GET_ALL, iface, &stats, sizeof(stats));
```

## 最佳实践

### 1. 网络安全

- 使用安全协议
- 验证证书
- 加密敏感数据
- 实现访问控制

### 2. 错误处理

- 检查返回值
- 实现超时机制
- 处理断开连接
- 实现重连逻辑

### 3. 资源管理

- 释放套接字
- 管理内存使用
- 限制并发连接
- 实现清理机制

### 4. 性能优化

- 使用缓冲池
- 优化数据包大小
- 实现数据压缩
- 减少数据复制

## 常见问题

### 1. 连接问题

**问题**：无法建立网络连接

**解决方案**：
- 检查网络配置
- 验证防火墙设置
- 确认路由配置
- 测试网络可达性

### 2. 性能问题

**问题**：网络性能不佳

**解决方案**：
- 优化缓冲区大小
- 减少数据拷贝
- 使用零拷贝技术
- 实现数据批处理

### 3. 内存问题

**问题**：内存使用过高

**解决方案**：
- 使用内存池
- 限制缓冲区大小
- 及时释放资源
- 监控内存使用

### 4. 稳定性问题

**问题**：连接不稳定

**解决方案**：
- 实现重连机制
- 添加心跳检测
- 处理超时情况
- 实现错误恢复

## 总结

Zephyr RTOS 提供了完整的网络开发支持，包括多种协议实现和网络应用框架。通过正确使用这些功能，可以开发出稳定、高效的网络应用。本文档提供了详细的指导和实例，帮助开发者更好地理解和使用 Zephyr 的网络功能。