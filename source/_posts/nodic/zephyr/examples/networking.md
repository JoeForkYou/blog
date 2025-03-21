---
abbrlink: 28
title: networking
date: '2025-03-21 20:49:56'
---
# Zephyr 网络示例

本文档提供了 Zephyr RTOS 的网络编程示例，包括 TCP/IP 通信、UDP 通信、HTTP 客户端/服务器、MQTT 客户端和 CoAP 通信等内容。

## TCP/IP 通信

### TCP 服务器示例

这个示例展示了如何创建一个简单的 TCP 服务器。

```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>
#include <zephyr/net/net_ip.h>

#define PORT 4242
#define BUFFER_SIZE 1024

void main(void)
{
    int serv_sock, client_sock;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    char rx_buffer[BUFFER_SIZE];
    int received;

    /* 创建套接字 */
    serv_sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (serv_sock < 0) {
        printk("Failed to create socket: %d\n", errno);
        return;
    }

    /* 配置服务器地址 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(PORT);

    /* 绑定地址 */
    if (bind(serv_sock, (struct sockaddr *)&server_addr,
             sizeof(server_addr)) < 0) {
        printk("Failed to bind: %d\n", errno);
        close(serv_sock);
        return;
    }

    /* 监听连接 */
    if (listen(serv_sock, 1) < 0) {
        printk("Failed to listen: %d\n", errno);
        close(serv_sock);
        return;
    }

    printk("TCP server listening on port %d\n", PORT);

    while (1) {
        /* 接受客户端连接 */
        client_sock = accept(serv_sock, (struct sockaddr *)&client_addr,
                           &client_addr_len);
        if (client_sock < 0) {
            printk("Failed to accept: %d\n", errno);
            continue;
        }

        printk("Client connected\n");

        /* 接收数据 */
        received = recv(client_sock, rx_buffer, sizeof(rx_buffer) - 1, 0);
        if (received > 0) {
            rx_buffer[received] = '\0';
            printk("Received: %s\n", rx_buffer);

            /* 发送响应 */
            send(client_sock, "Message received\n", 16, 0);
        }

        close(client_sock);
    }
}
```

### TCP 客户端示例

这个示例展示了如何创建一个 TCP 客户端连接服务器。

```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>
#include <zephyr/net/net_ip.h>

#define SERVER_PORT 4242
#define SERVER_ADDR "192.168.1.100"
#define BUFFER_SIZE 1024

void main(void)
{
    int sock;
    struct sockaddr_in server_addr;
    char tx_buffer[] = "Hello from Zephyr TCP client!";
    char rx_buffer[BUFFER_SIZE];
    int received;

    /* 创建套接字 */
    sock = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
    if (sock < 0) {
        printk("Failed to create socket: %d\n", errno);
        return;
    }

    /* 配置服务器地址 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    inet_pton(AF_INET, SERVER_ADDR, &server_addr.sin_addr);

    /* 连接服务器 */
    if (connect(sock, (struct sockaddr *)&server_addr,
                sizeof(server_addr)) < 0) {
        printk("Failed to connect: %d\n", errno);
        close(sock);
        return;
    }

    printk("Connected to server\n");

    /* 发送数据 */
    if (send(sock, tx_buffer, strlen(tx_buffer), 0) < 0) {
        printk("Failed to send: %d\n", errno);
        close(sock);
        return;
    }

    /* 接收响应 */
    received = recv(sock, rx_buffer, sizeof(rx_buffer) - 1, 0);
    if (received > 0) {
        rx_buffer[received] = '\0';
        printk("Received: %s\n", rx_buffer);
    }

    close(sock);
}
```

## UDP 通信

### UDP 服务器示例

这个示例展示了如何创建一个 UDP 服务器接收数据。

```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>
#include <zephyr/net/net_ip.h>

#define PORT 5683
#define BUFFER_SIZE 1024

void main(void)
{
    int sock;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_addr_len = sizeof(client_addr);
    char rx_buffer[BUFFER_SIZE];
    int received;

    /* 创建 UDP 套接字 */
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        printk("Failed to create socket: %d\n", errno);
        return;
    }

    /* 配置服务器地址 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(PORT);

    /* 绑定地址 */
    if (bind(sock, (struct sockaddr *)&server_addr,
             sizeof(server_addr)) < 0) {
        printk("Failed to bind: %d\n", errno);
        close(sock);
        return;
    }

    printk("UDP server listening on port %d\n", PORT);

    while (1) {
        /* 接收数据 */
        received = recvfrom(sock, rx_buffer, sizeof(rx_buffer) - 1, 0,
                          (struct sockaddr *)&client_addr, &client_addr_len);
        if (received > 0) {
            rx_buffer[received] = '\0';
            printk("Received from client: %s\n", rx_buffer);

            /* 发送响应 */
            sendto(sock, "ACK", 3, 0,
                   (struct sockaddr *)&client_addr, client_addr_len);
        }
    }
}
```

### UDP 客户端示例

这个示例展示了如何创建一个 UDP 客户端发送数据。

```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>
#include <zephyr/net/net_ip.h>

#define SERVER_PORT 5683
#define SERVER_ADDR "192.168.1.100"
#define BUFFER_SIZE 1024

void main(void)
{
    int sock;
    struct sockaddr_in server_addr;
    char tx_buffer[] = "Hello from Zephyr UDP client!";
    char rx_buffer[BUFFER_SIZE];
    int received;
    socklen_t server_addr_len = sizeof(server_addr);

    /* 创建 UDP 套接字 */
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        printk("Failed to create socket: %d\n", errno);
        return;
    }

    /* 配置服务器地址 */
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERVER_PORT);
    inet_pton(AF_INET, SERVER_ADDR, &server_addr.sin_addr);

    /* 发送数据 */
    if (sendto(sock, tx_buffer, strlen(tx_buffer), 0,
               (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        printk("Failed to send: %d\n", errno);
        close(sock);
        return;
    }

    /* 接收响应 */
    received = recvfrom(sock, rx_buffer, sizeof(rx_buffer) - 1, 0,
                       (struct sockaddr *)&server_addr, &server_addr_len);
    if (received > 0) {
        rx_buffer[received] = '\0';
        printk("Server response: %s\n", rx_buffer);
    }

    close(sock);
}
```

## HTTP 客户端

这个示例展示了如何创建一个 HTTP 客户端发送请求。

```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>
#include <zephyr/net/http/client.h>

#define HTTP_PORT 80
#define BUFFER_SIZE 1024

/* HTTP 请求回调 */
static void http_response_cb(struct http_response *rsp,
                           enum http_final_call final_data,
                           void *user_data)
{
    if (final_data == HTTP_DATA_MORE) {
        printk("%.*s", rsp->data_len, (char *)rsp->recv_buf);
    }
}

void main(void)
{
    int ret;
    struct http_request req;
    char response[BUFFER_SIZE];

    /* 配置 HTTP 请求 */
    memset(&req, 0, sizeof(req));
    req.method = HTTP_GET;
    req.url = "http://example.com/";
    req.host = "example.com";
    req.protocol = "HTTP/1.1";
    req.response = response;
    req.response_size = sizeof(response);
    req.header_fields = NULL;
    req.payload = NULL;
    req.payload_len = 0;
    req.recv_buf = response;
    req.recv_buf_len = sizeof(response);

    /* 发送 HTTP 请求 */
    ret = http_client_req(&req, http_response_cb, 5000, NULL);
    if (ret < 0) {
        printk("Failed to send HTTP request: %d\n", ret);
        return;
    }
}
```

## MQTT 客户端

这个示例展示了如何创建一个 MQTT 客户端连接到代理服务器。

```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>
#include <zephyr/net/mqtt.h>

#define MQTT_BROKER_ADDR "mqtt.example.com"
#define MQTT_BROKER_PORT 1883
#define MQTT_CLIENT_ID "zephyr_mqtt_client"
#define MQTT_TOPIC "test/topic"

/* MQTT 客户端上下文 */
static struct mqtt_client client;
static uint8_t rx_buffer[256];
static uint8_t tx_buffer[256];

/* MQTT 事件回调 */
static void mqtt_evt_handler(struct mqtt_client *client,
                           const struct mqtt_evt *evt)
{
    switch (evt->type) {
    case MQTT_EVT_CONNACK:
        if (evt->result == 0) {
            printk("MQTT client connected\n");
        }
        break;

    case MQTT_EVT_DISCONNECT:
        printk("MQTT client disconnected\n");
        break;

    case MQTT_EVT_PUBLISH:
        printk("MQTT PUBLISH received\n");
        break;

    default:
        printk("MQTT event: %d\n", evt->type);
        break;
    }
}

void main(void)
{
    int ret;
    struct sockaddr_in broker;

    /* 配置代理服务器地址 */
    broker.sin_family = AF_INET;
    broker.sin_port = htons(MQTT_BROKER_PORT);
    inet_pton(AF_INET, MQTT_BROKER_ADDR, &broker.sin_addr);

    /* 初始化 MQTT 客户端 */
    mqtt_client_init(&client);

    /* 配置 MQTT 客户端 */
    client.broker = (struct sockaddr *)&broker;
    client.evt_cb = mqtt_evt_handler;
    client.client_id.utf8 = (uint8_t *)MQTT_CLIENT_ID;
    client.client_id.size = strlen(MQTT_CLIENT_ID);
    client.password = NULL;
    client.user_name = NULL;
    client.protocol_version = MQTT_VERSION_3_1_1;
    client.rx_buf = rx_buffer;
    client.rx_buf_size = sizeof(rx_buffer);
    client.tx_buf = tx_buffer;
    client.tx_buf_size = sizeof(tx_buffer);

    /* 连接到代理服务器 */
    ret = mqtt_connect(&client);
    if (ret < 0) {
        printk("Failed to connect to MQTT broker: %d\n", ret);
        return;
    }

    /* 发布消息 */
    struct mqtt_publish_param param = {
        .message.topic.qos = MQTT_QOS_1_AT_LEAST_ONCE,
        .message.topic.topic.utf8 = (uint8_t *)MQTT_TOPIC,
        .message.topic.topic.size = strlen(MQTT_TOPIC),
        .message.payload.data = "Hello from Zephyr MQTT client!",
        .message.payload.len = 29,
        .message_id = 1,
        .dup_flag = 0,
        .retain_flag = 0
    };

    ret = mqtt_publish(&client, &param);
    if (ret < 0) {
        printk("Failed to publish message: %d\n", ret);
        return;
    }

    /* 主循环 */
    while (1) {
        k_sleep(K_SECONDS(1));
    }
}
```

## CoAP 通信

这个示例展示了如何使用 CoAP 协议进行通信。

```c
#include <zephyr/kernel.h>
#include <zephyr/net/socket.h>
#include <zephyr/net/coap.h>

#define COAP_PORT 5683
#define MAX_COAP_MSG_LEN 256

/* CoAP 资源处理函数 */
static int handle_get_temperature(struct coap_resource *resource,
                                struct coap_packet *request,
                                struct sockaddr *addr, socklen_t addr_len)
{
    uint8_t payload[] = "23.5";
    struct coap_packet response;
    uint8_t *data;
    int r;

    data = (uint8_t *)k_malloc(MAX_COAP_MSG_LEN);
    if (!data) {
        return -ENOMEM;
    }

    r = coap_packet_init(&response, data, MAX_COAP_MSG_LEN,
                        COAP_VERSION_1, COAP_TYPE_ACK,
                        coap_header_get_token_len(request),
                        coap_header_get_token(request),
                        COAP_RESPONSE_CODE_CONTENT, 
                        coap_header_get_id(request));
    if (r < 0) {
        goto end;
    }

    r = coap_packet_append_payload_marker(&response);
    if (r < 0) {
        goto end;
    }

    r = coap_packet_append_payload(&response, payload, sizeof(payload) - 1);
    if (r < 0) {
        goto end;
    }

    r = sendto(resource->sock, data, response.offset, 0, addr, addr_len);

end:
    k_free(data);
    return r;
}

/* CoAP 资源定义 */
static const char * const temperature_path[] = { "temperature", NULL };
static struct coap_resource temperature_resource = {
    .get = handle_get_temperature,
    .path = temperature_path
};

void main(void)
{
    int sock;
    struct sockaddr_in addr;
    int received;
    uint8_t buffer[MAX_COAP_MSG_LEN];

    /* 创建 UDP 套接字 */
    sock = socket(AF_INET, SOCK_DGRAM, IPPROTO_UDP);
    if (sock < 0) {
        printk("Failed to create socket: %d\n", errno);
        return;
    }

    /* 配置地址 */
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY);
    addr.sin_port = htons(COAP_PORT);

    /* 绑定地址 */
    if (bind(sock, (struct sockaddr *)&addr, sizeof(addr)) < 0) {
        printk("Failed to bind: %d\n", errno);
        close(sock);
        return;
    }

    /* 设置资源的套接字 */
    temperature_resource.sock = sock;

    printk("CoAP server started on port %d\n", COAP_PORT);

    while (1) {
        struct sockaddr client_addr;
        socklen_t client_addr_len = sizeof(client_addr);
        struct coap_packet request;
        struct coap_option options[16];
        uint8_t opt_num = 16U;

        received = recvfrom(sock, buffer, sizeof(buffer), 0,
                          &client_addr, &client_addr_len);
        if (received < 0) {
            continue;
        }

        if (coap_packet_parse(&request, buffer, received, options, opt_num) < 0) {
            continue;
        }

        /* 处理请求 */
        if (coap_handle_request(&request, &temperature_resource,
                              &client_addr, client_addr_len) < 0) {
            printk("Failed to handle CoAP request\n");
        }
    }
}
```

## 配置说明

要运行这些网络示例，需要在 `prj.conf` 中添加相应的配置：

```
# 网络基础配置
CONFIG_NETWORKING=y
CONFIG_NET_IPV4=y
CONFIG_NET_TCP=y
CONFIG_NET_UDP=y

# 网络应用配置
CONFIG_NET_SOCKETS=y
CONFIG_NET_SOCKETS_POSIX_NAMES=y

# HTTP 客户端配置
CONFIG_HTTP_CLIENT=y
CONFIG_HTTP_PARSER=y

# MQTT 配置
CONFIG_MQTT_LIB=y
CONFIG_MQTT_LIB_TLS=n

# CoAP 配置
CONFIG_COAP=y

# 网络缓冲区配置
CONFIG_NET_BUF_RX_COUNT=16
CONFIG_NET_BUF_TX_COUNT=16
CONFIG_NET_PKT_RX_COUNT=16
CONFIG_NET_PKT_TX_COUNT=16

# 日志配置
CONFIG_NET_LOG=y
CONFIG_NET_SHELL=y
```

## 编译和运行

```bash
# 编译 TCP 示例
west build -b <board> samples/net/sockets/tcp

# 编译 UDP 示例
west build -b <board> samples/net/sockets/udp

# 编译 HTTP 客户端示例
west build -b <board> samples/net/http_client

# 编译 MQTT 客户端示例
west build -b <board> samples/net/mqtt_publisher

# 编译 CoAP 示例
west build -b <board> samples/net/coap_server

# 烧录到开发板
west flash
```

## 总结

这些网络示例展示了 Zephyr RTOS 中不同网络协议的使用方法。通过这些示例，您可以学习如何：

1. 使用套接字 API 进行 TCP/UDP 通信
2. 实现 HTTP 客户端发送请求
3. 创建 MQTT 客户端与代理服务器通信
4. 使用 CoAP 协议进行物联网通信

这些示例可以作为开发网络应用的起点，您可以根据需要修改和扩展它们。记住要根据您的网络环境配置正确的网络参数，并确保网络连接可用。