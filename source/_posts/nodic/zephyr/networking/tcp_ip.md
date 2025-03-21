---
abbrlink: 35
title: tcp_ip
date: '2025-03-21 20:49:56'
---
# Zephyr TCP/IP 网络协议栈指南

## 1. 基础配置

### 1.1 网络配置 (prj.conf)
```plaintext
# 基础网络支持
CONFIG_NETWORKING=y
CONFIG_NET_IPV4=y
CONFIG_NET_IPV6=y
CONFIG_NET_TCP=y
CONFIG_NET_UDP=y

# 网络缓冲区配置
CONFIG_NET_PKT_RX_COUNT=16
CONFIG_NET_PKT_TX_COUNT=16
CONFIG_NET_BUF_RX_COUNT=64
CONFIG_NET_BUF_TX_COUNT=64
CONFIG_NET_BUF_DATA_SIZE=128

# DHCP客户端
CONFIG_NET_DHCPV4=y

# DNS解析
CONFIG_DNS_RESOLVER=y
CONFIG_DNS_SERVER_IP_ADDRESSES=y
CONFIG_DNS_SERVER1="8.8.8.8"

# 网络日志
CONFIG_NET_LOG=y
CONFIG_NET_LOG_LEVEL_DBG=y
```

### 1.2 网络接口配置
```c
#include <zephyr/kernel.h>
#include <zephyr/net/net_if.h>
#include <zephyr/net/net_core.h>
#include <zephyr/net/net_context.h>
#include <zephyr/net/net_mgmt.h>

static struct net_if *iface;

static void net_iface_init(void)
{
    /* 获取默认网络接口 */
    iface = net_if_get_default();
    if (!iface) {
        printk("No network interface available\n");
        return;
    }

    /* 配置IPv4地址 */
    struct in_addr addr4;
    net_addr_pton(AF_INET, "192.168.1.100", &addr4);
    net_if_ipv4_addr_add(iface, &addr4, NET_ADDR_MANUAL, 0);

    /* 配置IPv6地址 */
    struct in6_addr addr6;
    net_addr_pton(AF_INET6, "2001:db8::1", &addr6);
    net_if_ipv6_addr_add(iface, &addr6, NET_ADDR_MANUAL, 0);
}
```

## 2. TCP通信

### 2.1 TCP服务器
```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>

#define TCP_PORT 4242
#define BUFFER_SIZE 1024

int tcp_server_example(void)
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
    server_addr.sin_port = htons(TCP_PORT);

    /* 绑定地址 */
    ret = bind(serv_sock, (struct sockaddr *)&server_addr,
               sizeof(server_addr));
    if (ret < 0) {
        printk("Failed to bind: %d\n", errno);
        close(serv_sock);
        return -errno;
    }

    /* 监听连接 */
    ret = listen(serv_sock, 5);
    if (ret < 0) {
        printk("Failed to listen: %d\n", errno);
        close(serv_sock);
        return -errno;
    }

    while (1) {
        /* 接受连接 */
        client_sock = accept(serv_sock,
                           (struct sockaddr *)&client_addr,
                           &client_addr_len);
        if (client_sock < 0) {
            printk("Accept failed: %d\n", errno);
            continue;
        }

        /* 处理连接 */
        while (1) {
            ret = recv(client_sock, rx_buf, sizeof(rx_buf), 0);
            if (ret < 0) {
                printk("Receive failed: %d\n", errno);
                break;
            }
            if (ret == 0) {
                printk("Connection closed\n");
                break;
            }

            /* 发送响应 */
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

### 2.2 TCP客户端
```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>

#define SERVER_PORT 4242
#define BUFFER_SIZE 1024

int tcp_client_example(void)
{
    int sock;
    struct sockaddr_in server_addr;
    char tx_buf[] = "Hello, Server!";
    char rx_buf[BUFFER_SIZE];
    int ret;

    /* 创建socket */
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (sock < 0) {
        printk("Failed to create socket: %d\n", errno);
        return -errno;
    }

    /* 配置服务器地址 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    inet_pton(AF_INET, "192.168.1.100", &server_addr.sin_addr);

    /* 连接服务器 */
    ret = connect(sock, (struct sockaddr *)&server_addr,
                 sizeof(server_addr));
    if (ret < 0) {
        printk("Failed to connect: %d\n", errno);
        close(sock);
        return -errno;
    }

    /* 发送数据 */
    ret = send(sock, tx_buf, strlen(tx_buf), 0);
    if (ret < 0) {
        printk("Failed to send data: %d\n", errno);
        close(sock);
        return -errno;
    }

    /* 接收响应 */
    ret = recv(sock, rx_buf, sizeof(rx_buf), 0);
    if (ret < 0) {
        printk("Failed to receive: %d\n", errno);
        close(sock);
        return -errno;
    }

    rx_buf[ret] = '\0';
    printk("Received: %s\n", rx_buf);

    close(sock);
    return 0;
}
```

## 3. UDP通信

### 3.1 UDP服务器
```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>

#define UDP_PORT 5678
#define BUFFER_SIZE 1024

int udp_server_example(void)
{
    int sock;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    char rx_buf[BUFFER_SIZE];
    int ret;

    /* 创建socket */
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        printk("Failed to create socket: %d\n", errno);
        return -errno;
    }

    /* 配置服务器地址 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(UDP_PORT);

    /* 绑定地址 */
    ret = bind(sock, (struct sockaddr *)&server_addr,
               sizeof(server_addr));
    if (ret < 0) {
        printk("Failed to bind: %d\n", errno);
        close(sock);
        return -errno;
    }

    while (1) {
        /* 接收数据 */
        ret = recvfrom(sock, rx_buf, sizeof(rx_buf), 0,
                      (struct sockaddr *)&client_addr,
                      &client_addr_len);
        if (ret < 0) {
            printk("Failed to receive: %d\n", errno);
            continue;
        }

        /* 发送响应 */
        ret = sendto(sock, rx_buf, ret, 0,
                    (struct sockaddr *)&client_addr,
                    client_addr_len);
        if (ret < 0) {
            printk("Failed to send: %d\n", errno);
            continue;
        }
    }

    close(sock);
    return 0;
}
```

### 3.2 UDP客户端
```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>

#define SERVER_PORT 5678
#define BUFFER_SIZE 1024

int udp_client_example(void)
{
    int sock;
    struct sockaddr_in server_addr;
    char tx_buf[] = "Hello, UDP Server!";
    char rx_buf[BUFFER_SIZE];
    int ret;

    /* 创建socket */
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        printk("Failed to create socket: %d\n", errno);
        return -errno;
    }

    /* 配置服务器地址 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    inet_pton(AF_INET, "192.168.1.100", &server_addr.sin_addr);

    /* 发送数据 */
    ret = sendto(sock, tx_buf, strlen(tx_buf), 0,
                (struct sockaddr *)&server_addr,
                sizeof(server_addr));
    if (ret < 0) {
        printk("Failed to send data: %d\n", errno);
        close(sock);
        return -errno;
    }

    /* 接收响应 */
    ret = recv(sock, rx_buf, sizeof(rx_buf), 0);
    if (ret < 0) {
        printk("Failed to receive: %d\n", errno);
        close(sock);
        return -errno;
    }

    rx_buf[ret] = '\0';
    printk("Received: %s\n", rx_buf);

    close(sock);
    return 0;
}
```

## 4. 网络管理

### 4.1 网络事件监听
```c
#include <zephyr/kernel.h>
#include <zephyr/net/net_mgmt.h>
#include <zephyr/net/net_event.h>

static struct net_mgmt_event_callback mgmt_cb;

/* 网络事件处理函数 */
static void net_event_handler(struct net_mgmt_event_callback *cb,
                            uint32_t mgmt_event,
                            struct net_if *iface)
{
    switch (mgmt_event) {
    case NET_EVENT_IPV4_ADDR_ADD:
        printk("IPv4 address added\n");
        break;
    case NET_EVENT_IPV4_ADDR_DEL:
        printk("IPv4 address removed\n");
        break;
    case NET_EVENT_IPV6_ADDR_ADD:
        printk("IPv6 address added\n");
        break;
    case NET_EVENT_IPV6_ADDR_DEL:
        printk("IPv6 address removed\n");
        break;
    }
}

void net_mgmt_init(void)
{
    /* 注册网络事件回调 */
    net_mgmt_init_event_callback(&mgmt_cb,
                               net_event_handler,
                               NET_EVENT_IPV4_ADDR_ADD |
                               NET_EVENT_IPV4_ADDR_DEL |
                               NET_EVENT_IPV6_ADDR_ADD |
                               NET_EVENT_IPV6_ADDR_DEL);

    net_mgmt_add_event_callback(&mgmt_cb);
}
```

### 4.2 DHCP客户端
```c
#include <zephyr/kernel.h>
#include <zephyr/net/net_if.h>
#include <zephyr/net/net_mgmt.h>
#include <zephyr/net/dhcpv4.h>

static struct net_mgmt_event_callback dhcp_cb;

/* DHCP事件处理函数 */
static void dhcp_handler(struct net_mgmt_event_callback *cb,
                        uint32_t mgmt_event,
                        struct net_if *iface)
{
    switch (mgmt_event) {
    case NET_EVENT_IPV4_DHCP_BOUND:
        {
            char addr_str[NET_IPV4_ADDR_LEN];
            struct net_if_dhcpv4 *dhcpv4 = iface->config.dhcpv4;
            
            net_addr_ntop(AF_INET,
                         &dhcpv4->requested_ip,
                         addr_str,
                         sizeof(addr_str));
            printk("DHCP bound to address: %s\n", addr_str);
        }
        break;
    case NET_EVENT_IPV4_DHCP_STOP:
        printk("DHCP stopped\n");
        break;
    }
}

void dhcp_init(void)
{
    struct net_if *iface = net_if_get_default();

    /* 注册DHCP事件回调 */
    net_mgmt_init_event_callback(&dhcp_cb,
                               dhcp_handler,
                               NET_EVENT_IPV4_DHCP_BOUND |
                               NET_EVENT_IPV4_DHCP_STOP);

    net_mgmt_add_event_callback(&dhcp_cb);

    /* 启动DHCP客户端 */
    net_dhcpv4_start(iface);
}
```

## 5. DNS解析

### 5.1 DNS客户端
```c
#include <zephyr/kernel.h>
#include <zephyr/net/dns_resolve.h>

#define DNS_TIMEOUT K_MSEC(2000)

void dns_resolve_example(void)
{
    uint8_t ip_addr[NET_IPV4_ADDR_LEN];
    int ret;

    /* 解析域名 */
    ret = dns_resolve_name(DNS_RESOLVE_DEFAULT_CTX,
                          "example.com",
                          DNS_QUERY_TYPE_A,
                          DNS_TIMEOUT,
                          ip_addr,
                          sizeof(ip_addr));
    if (ret < 0) {
        printk("Failed to resolve name: %d\n", ret);
        return;
    }

    /* 打印解析结果 */
    printk("Resolved IP: %d.%d.%d.%d\n",
           ip_addr[0], ip_addr[1],
           ip_addr[2], ip_addr[3]);
}
```

## 6. 网络安全

### 6.1 TLS配置
```plaintext
# TLS配置 (prj.conf)
CONFIG_MBEDTLS=y
CONFIG_MBEDTLS_BUILTIN=y
CONFIG_MBEDTLS_SSL_MAX_CONTENT_LEN=1500
CONFIG_MBEDTLS_ENABLE_HEAP=y
CONFIG_MBEDTLS_HEAP_SIZE=10240
```

### 6.2 TLS客户端
```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>
#include <zephyr/net/tls_credentials.h>

#define TLS_PSK_ID "Client_identity"
#define TLS_PSK_KEY "123456"

void tls_client_example(void)
{
    int sock, ret;
    struct sockaddr_in addr;

    /* 加载TLS证书 */
    ret = tls_credential_add(1, TLS_CREDENTIAL_PSK_ID,
                           TLS_PSK_ID, strlen(TLS_PSK_ID));
    if (ret < 0) {
        printk("Failed to add PSK ID: %d\n", ret);
        return;
    }

    ret = tls_credential_add(1, TLS_CREDENTIAL_PSK,
                           TLS_PSK_KEY, strlen(TLS_PSK_KEY));
    if (ret < 0) {
        printk("Failed to add PSK: %d\n", ret);
        return;
    }

    /* 创建TLS socket */
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TLS_1_2);
    if (sock < 0) {
        printk("Failed to create socket: %d\n", errno);
        return;
    }

    /* 配置TLS选项 */
    sec_tag_t sec_tag_list[] = {1};
    ret = setsockopt(sock, SOL_TLS, TLS_SEC_TAG_LIST,
                     sec_tag_list, sizeof(sec_tag_list));
    if (ret < 0) {
        printk("Failed to set TLS_SEC_TAG_LIST: %d\n", errno);
        close(sock);
        return;
    }

    /* 连接服务器 */
    addr.sin_family = AF_INET;
    addr.sin_port = htons(443);
    inet_pton(AF_INET, "192.168.1.100", &addr.sin_addr);

    ret = connect(sock, (struct sockaddr *)&addr, sizeof(addr));
    if (ret < 0) {
        printk("Failed to connect: %d\n", errno);
        close(sock);
        return;
    }

    /* 使用安全连接 */
    // ... 发送和接收数据 ...

    close(sock);
}
```