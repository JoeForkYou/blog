---
abbrlink: 26
title: bluetooth
date: '2025-03-21 20:49:56'
---
# Zephyr 蓝牙示例

本文档提供了 Zephyr RTOS 中使用蓝牙功能的示例代码，包括 BLE 广播、GATT 服务、BLE 中心设备、BLE 外围设备和蓝牙 Mesh 等内容。

## BLE 广播

这个示例展示了如何实现一个 BLE 广播设备。

### 源代码

```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>

/* 定义广播数据 */
static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA_BYTES(BT_DATA_NAME_COMPLETE, 'Z', 'e', 'p', 'h', 'y', 'r'),
};

/* 定义扫描响应数据 */
static const struct bt_data sd[] = {
    BT_DATA_BYTES(BT_DATA_UUID16_ALL, 0x0d, 0x18), /* 0x180d 心率服务 */
};

void main(void)
{
    int err;

    /* 初始化蓝牙协议栈 */
    err = bt_enable(NULL);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return;
    }

    printk("Bluetooth initialized\n");

    /* 启动广播 */
    err = bt_le_adv_start(BT_LE_ADV_CONN_NAME, ad, ARRAY_SIZE(ad),
                         sd, ARRAY_SIZE(sd));
    if (err) {
        printk("Advertising failed to start (err %d)\n", err);
        return;
    }

    printk("Advertising started\n");

    /* 主循环 */
    while (1) {
        k_sleep(K_SECONDS(1));
    }
}
```

### 配置文件

**prj.conf**:
```
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_DEVICE_NAME="Zephyr"
```

## GATT 服务

这个示例展示了如何实现一个自定义的 GATT 服务。

### 源代码

```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
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

/* 特征值 */
static uint8_t custom_value[] = { 0x00 };

/* 读取回调 */
static ssize_t read_custom(struct bt_conn *conn,
                          const struct bt_gatt_attr *attr,
                          void *buf, uint16_t len, uint16_t offset)
{
    const uint8_t *value = attr->user_data;

    return bt_gatt_attr_read(conn, attr, buf, len, offset, value,
                            sizeof(custom_value));
}

/* 写入回调 */
static ssize_t write_custom(struct bt_conn *conn,
                           const struct bt_gatt_attr *attr,
                           const void *buf, uint16_t len,
                           uint16_t offset, uint8_t flags)
{
    uint8_t *value = attr->user_data;

    if (offset + len > sizeof(custom_value)) {
        return BT_GATT_ERR(BT_ATT_ERR_INVALID_OFFSET);
    }

    memcpy(value + offset, buf, len);
    printk("Value updated: %u\n", *value);

    return len;
}

/* 定义 GATT 服务 */
BT_GATT_SERVICE_DEFINE(custom_svc,
    BT_GATT_PRIMARY_SERVICE(&custom_service_uuid),
    BT_GATT_CHARACTERISTIC(&custom_characteristic_uuid.uuid,
                          BT_GATT_CHRC_READ | BT_GATT_CHRC_WRITE,
                          BT_GATT_PERM_READ | BT_GATT_PERM_WRITE,
                          read_custom, write_custom, custom_value),
);

/* 连接回调 */
static void connected(struct bt_conn *conn, uint8_t err)
{
    if (err) {
        printk("Connection failed (err 0x%02x)\n", err);
    } else {
        printk("Connected\n");
    }
}

/* 断开连接回调 */
static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    printk("Disconnected (reason 0x%02x)\n", reason);
}

/* 连接回调结构体 */
BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
};

void main(void)
{
    int err;

    /* 初始化蓝牙协议栈 */
    err = bt_enable(NULL);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return;
    }

    printk("Bluetooth initialized\n");

    /* 启动广播 */
    err = bt_le_adv_start(BT_LE_ADV_CONN, NULL, 0, NULL, 0);
    if (err) {
        printk("Advertising failed to start (err %d)\n", err);
        return;
    }

    printk("Advertising started\n");

    /* 主循环 */
    while (1) {
        k_sleep(K_SECONDS(1));
    }
}
```

### 配置文件

**prj.conf**:
```
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_DEVICE_NAME="Zephyr GATT"
CONFIG_BT_DEVICE_APPEARANCE=0
CONFIG_BT_MAX_CONN=1
```

## BLE 中心设备

这个示例展示了如何实现一个 BLE 中心设备，扫描并连接到外围设备。

### 源代码

```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
#include <zephyr/bluetooth/conn.h>
#include <zephyr/bluetooth/uuid.h>
#include <zephyr/bluetooth/gatt.h>

/* 连接句柄 */
static struct bt_conn *default_conn;

/* 扫描参数 */
static struct bt_le_scan_param scan_param = {
    .type = BT_LE_SCAN_TYPE_ACTIVE,
    .options = BT_LE_SCAN_OPT_NONE,
    .interval = BT_GAP_SCAN_FAST_INTERVAL,
    .window = BT_GAP_SCAN_FAST_WINDOW,
};

/* 扫描回调 */
static void device_found(const bt_addr_le_t *addr, int8_t rssi, uint8_t type,
                        struct net_buf_simple *ad)
{
    char addr_str[BT_ADDR_LE_STR_LEN];
    int err;

    /* 将蓝牙地址转换为字符串 */
    bt_addr_le_to_str(addr, addr_str, sizeof(addr_str));

    /* 检查是否已连接 */
    if (default_conn) {
        return;
    }

    /* 停止扫描 */
    err = bt_le_scan_stop();
    if (err) {
        printk("Failed to stop scanning (err %d)\n", err);
        return;
    }

    /* 连接到设备 */
    err = bt_conn_le_create(addr, BT_CONN_LE_CREATE_CONN,
                          BT_LE_CONN_PARAM_DEFAULT, &default_conn);
    if (err) {
        printk("Failed to create connection (err %d)\n", err);
        /* 重新开始扫描 */
        bt_le_scan_start(&scan_param, device_found);
    }
}

/* 连接回调 */
static void connected(struct bt_conn *conn, uint8_t err)
{
    char addr[BT_ADDR_LE_STR_LEN];

    bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));

    if (err) {
        printk("Failed to connect to %s (err %u)\n", addr, err);

        if (conn == default_conn) {
            bt_conn_unref(default_conn);
            default_conn = NULL;

            /* 重新开始扫描 */
            bt_le_scan_start(&scan_param, device_found);
        }

        return;
    }

    printk("Connected to %s\n", addr);

    if (conn == default_conn) {
        /* 发现服务 */
        err = bt_gatt_discover(conn, NULL);
        if (err) {
            printk("Failed to start discovery (err %d)\n", err);
        }
    }
}

/* 断开连接回调 */
static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    char addr[BT_ADDR_LE_STR_LEN];

    bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));

    printk("Disconnected from %s (reason 0x%02x)\n", addr, reason);

    if (conn == default_conn) {
        bt_conn_unref(default_conn);
        default_conn = NULL;

        /* 重新开始扫描 */
        bt_le_scan_start(&scan_param, device_found);
    }
}

/* 连接回调结构体 */
BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
};

void main(void)
{
    int err;

    /* 初始化蓝牙协议栈 */
    err = bt_enable(NULL);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return;
    }

    printk("Bluetooth initialized\n");

    /* 启动扫描 */
    err = bt_le_scan_start(&scan_param, device_found);
    if (err) {
        printk("Scanning failed to start (err %d)\n", err);
        return;
    }

    printk("Scanning started\n");

    /* 主循环 */
    while (1) {
        k_sleep(K_SECONDS(1));
    }
}
```

### 配置文件

**prj.conf**:
```
CONFIG_BT=y
CONFIG_BT_CENTRAL=y
CONFIG_BT_GATT_CLIENT=y
```

## BLE 外围设备

这个示例展示了如何实现一个更复杂的 BLE 外围设备，提供心率服务。

### 源代码

```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/hci.h>
#include <zephyr/bluetooth/conn.h>
#include <zephyr/bluetooth/uuid.h>
#include <zephyr/bluetooth/gatt.h>

/* 心率服务 UUID */
static struct bt_uuid_16 hrs_uuid = BT_UUID_INIT_16(0x180D);

/* 心率测量特征 UUID */
static struct bt_uuid_16 hrm_uuid = BT_UUID_INIT_16(0x2A37);

/* 心率值 */
static uint8_t hrm_value[2] = { 0x06, 0x40 };

/* 定时器 */
static struct k_work_delayable hrm_work;

/* 当前连接 */
static struct bt_conn *current_conn;

/* 读取回调 */
static ssize_t read_hrs(struct bt_conn *conn,
                       const struct bt_gatt_attr *attr,
                       void *buf, uint16_t len, uint16_t offset)
{
    return bt_gatt_attr_read(conn, attr, buf, len, offset, hrm_value,
                            sizeof(hrm_value));
}

/* 心率服务定义 */
BT_GATT_SERVICE_DEFINE(hrs_svc,
    BT_GATT_PRIMARY_SERVICE(&hrs_uuid),
    BT_GATT_CHARACTERISTIC(&hrm_uuid.uuid,
                          BT_GATT_CHRC_READ | BT_GATT_CHRC_NOTIFY,
                          BT_GATT_PERM_READ, read_hrs, NULL, hrm_value),
    BT_GATT_CCC(NULL, BT_GATT_PERM_READ | BT_GATT_PERM_WRITE),
);

/* 更新心率值 */
static void hrm_update(struct k_work *work)
{
    /* 模拟心率变化 */
    hrm_value[1]++;
    if (hrm_value[1] > 160) {
        hrm_value[1] = 60;
    }

    /* 通知连接的设备 */
    bt_gatt_notify(NULL, &hrs_svc.attrs[1], hrm_value, sizeof(hrm_value));

    /* 重新调度工作 */
    k_work_schedule(&hrm_work, K_SECONDS(1));
}

/* 连接回调 */
static void connected(struct bt_conn *conn, uint8_t err)
{
    if (err) {
        printk("Connection failed (err 0x%02x)\n", err);
    } else {
        printk("Connected\n");
        current_conn = bt_conn_ref(conn);
    }
}

/* 断开连接回调 */
static void disconnected(struct bt_conn *conn, uint8_t reason)
{
    printk("Disconnected (reason 0x%02x)\n", reason);

    if (current_conn) {
        bt_conn_unref(current_conn);
        current_conn = NULL;
    }
}

/* 连接回调结构体 */
BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
    .disconnected = disconnected,
};

/* 广播数据 */
static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA_BYTES(BT_DATA_UUID16_ALL, 0x0d, 0x18), /* 心率服务 */
};

void main(void)
{
    int err;

    /* 初始化工作队列 */
    k_work_init_delayable(&hrm_work, hrm_update);

    /* 初始化蓝牙协议栈 */
    err = bt_enable(NULL);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return;
    }

    printk("Bluetooth initialized\n");

    /* 启动广播 */
    err = bt_le_adv_start(BT_LE_ADV_CONN, ad, ARRAY_SIZE(ad), NULL, 0);
    if (err) {
        printk("Advertising failed to start (err %d)\n", err);
        return;
    }

    printk("Advertising started\n");

    /* 启动心率更新 */
    k_work_schedule(&hrm_work, K_SECONDS(1));

    /* 主循环 */
    while (1) {
        k_sleep(K_SECONDS(1));
    }
}
```

### 配置文件

**prj.conf**:
```
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_DEVICE_NAME="Zephyr HRS"
CONFIG_BT_DEVICE_APPEARANCE=833
CONFIG_BT_MAX_CONN=1
```

## 编译和运行

```bash
# 编译 BLE 广播示例
west build -b <board> samples/bluetooth/beacon

# 编译 GATT 服务示例
west build -b <board> samples/bluetooth/peripheral

# 编译 BLE 中心设备示例
west build -b <board> samples/bluetooth/central

# 编译 BLE 外围设备示例
west build -b <board> samples/bluetooth/peripheral_hr

# 烧录到开发板
west flash
```

## 蓝牙 Mesh

这个示例展示了如何实现一个基本的蓝牙 Mesh 节点，包括配置服务器模型和通用开关服务器模型。

### 源代码

```c
#include <zephyr/kernel.h>
#include <zephyr/bluetooth/bluetooth.h>
#include <zephyr/bluetooth/mesh.h>

/* 定义模型操作码 */
#define BT_MESH_MODEL_OP_GEN_ONOFF_GET BT_MESH_MODEL_OP_2(0x82, 0x01)
#define BT_MESH_MODEL_OP_GEN_ONOFF_SET BT_MESH_MODEL_OP_2(0x82, 0x02)
#define BT_MESH_MODEL_OP_GEN_ONOFF_SET_UNACK BT_MESH_MODEL_OP_2(0x82, 0x03)
#define BT_MESH_MODEL_OP_GEN_ONOFF_STATUS BT_MESH_MODEL_OP_2(0x82, 0x04)

/* LED GPIO 设备 */
static const struct device *led_dev = DEVICE_DT_GET(DT_ALIAS(led0));
static uint8_t led_state = 0;

/* 通用开关服务器模型操作回调 */
static int gen_onoff_get(struct bt_mesh_model *model,
                        struct bt_mesh_msg_ctx *ctx,
                        struct net_buf_simple *buf)
{
    NET_BUF_SIMPLE_DEFINE(msg, 2 + 1 + 4);

    bt_mesh_model_msg_init(&msg, BT_MESH_MODEL_OP_GEN_ONOFF_STATUS);
    net_buf_simple_add_u8(&msg, led_state);

    bt_mesh_model_send(model, ctx, &msg, NULL, NULL);

    return 0;
}

static int gen_onoff_set_unack(struct bt_mesh_model *model,
                              struct bt_mesh_msg_ctx *ctx,
                              struct net_buf_simple *buf)
{
    uint8_t state = net_buf_simple_pull_u8(buf);

    led_state = state;
    gpio_pin_set(led_dev, DT_ALIAS_LED0_GPIOS_PIN, led_state);

    return 0;
}

static int gen_onoff_set(struct bt_mesh_model *model,
                        struct bt_mesh_msg_ctx *ctx,
                        struct net_buf_simple *buf)
{
    gen_onoff_set_unack(model, ctx, buf);
    gen_onoff_get(model, ctx, buf);

    return 0;
}

/* 通用开关服务器模型操作 */
static const struct bt_mesh_model_op gen_onoff_op[] = {
    { BT_MESH_MODEL_OP_GEN_ONOFF_GET, 0, gen_onoff_get },
    { BT_MESH_MODEL_OP_GEN_ONOFF_SET, 2, gen_onoff_set },
    { BT_MESH_MODEL_OP_GEN_ONOFF_SET_UNACK, 2, gen_onoff_set_unack },
    BT_MESH_MODEL_OP_END,
};

/* 根元素模型 */
static struct bt_mesh_model root_models[] = {
    BT_MESH_MODEL_CFG_SRV,
    BT_MESH_MODEL(BT_MESH_MODEL_ID_GEN_ONOFF_SRV, gen_onoff_op, NULL, NULL),
};

/* 根元素 */
static struct bt_mesh_elem elements[] = {
    BT_MESH_ELEM(0, root_models, BT_MESH_MODEL_NONE),
};

/* 节点组成 */
static const struct bt_mesh_comp comp = {
    .cid = CONFIG_BT_COMPANY_ID,
    .elem = elements,
    .elem_count = ARRAY_SIZE(elements),
};

/* 配置完成回调 */
static void prov_complete(uint16_t net_idx, uint16_t addr)
{
    printk("Provisioning completed\n");
}

/* 重置回调 */
static void prov_reset(void)
{
    bt_mesh_prov_enable(BT_MESH_PROV_ADV | BT_MESH_PROV_GATT);
}

/* 配置参数 */
static const struct bt_mesh_prov prov = {
    .uuid = dev_uuid,
    .output_size = 4,
    .output_actions = BT_MESH_DISPLAY_NUMBER,
    .output_number = output_number,
    .complete = prov_complete,
    .reset = prov_reset,
};

void main(void)
{
    int err;

    /* 初始化 LED */
    if (!device_is_ready(led_dev)) {
        printk("LED device not ready\n");
        return;
    }
    gpio_pin_configure(led_dev, DT_ALIAS_LED0_GPIOS_PIN, GPIO_OUTPUT_ACTIVE | DT_ALIAS_LED0_GPIOS_FLAGS);

    /* 初始化蓝牙协议栈 */
    err = bt_enable(NULL);
    if (err) {
        printk("Bluetooth init failed (err %d)\n", err);
        return;
    }

    printk("Bluetooth initialized\n");

    /* 初始化 Mesh */
    err = bt_mesh_init(&prov, &comp);
    if (err) {
        printk("Initializing mesh failed (err %d)\n", err);
        return;
    }

    /* 启用配置 */
    bt_mesh_prov_enable(BT_MESH_PROV_ADV | BT_MESH_PROV_GATT);

    printk("Mesh initialized\n");

    /* 主循环 */
    while (1) {
        k_sleep(K_SECONDS(1));
    }
}
```

### 配置文件

**prj.conf**:
```
CONFIG_BT=y
CONFIG_BT_MESH=y
CONFIG_BT_MESH_RELAY=y
CONFIG_BT_MESH_PB_ADV=y
CONFIG_BT_MESH_PB_GATT=y
CONFIG_BT_MESH_LOW_POWER=n
CONFIG_BT_MESH_FRIEND=n
CONFIG_BT_MESH_GATT_PROXY=y

CONFIG_BT_MESH_MODEL_GROUP_COUNT=1
CONFIG_BT_MESH_MODEL_SUBSCRIBE_GROUP_COUNT=1
CONFIG_BT_MESH_MODEL_EXTENSIONS=y
CONFIG_BT_MESH_SUBNET_COUNT=1
CONFIG_BT_MESH_APP_KEY_COUNT=1
CONFIG_BT_MESH_IV_UPDATE_TEST=y

CONFIG_GPIO=y
```

## 编译和运行

```bash
# 编译 BLE 广播示例
west build -b <board> samples/bluetooth/beacon

# 编译 GATT 服务示例
west build -b <board> samples/bluetooth/peripheral

# 编译 BLE 中心设备示例
west build -b <board> samples/bluetooth/central

# 编译 BLE 外围设备示例
west build -b <board> samples/bluetooth/peripheral_hr

# 编译蓝牙 Mesh 示例
west build -b <board> samples/bluetooth/mesh/onoff_server

# 烧录到开发板
west flash
```

## 最佳实践

1. **蓝牙协议栈初始化**
   - 始终检查 `bt_enable()` 的返回值
   - 在使用任何蓝牙功能之前确保协议栈已初始化

2. **错误处理**
   - 对所有 API 调用进行错误检查
   - 实现适当的错误恢复机制

3. **连接管理**
   - 正确处理连接和断开连接事件
   - 在断开连接时释放资源

4. **GATT 服务设计**
   - 遵循蓝牙 SIG 定义的标准服务和特征
   - 为自定义服务使用唯一的 UUID

5. **广播数据**
   - 包含必要的信息以便于发现
   - 遵循蓝牙规范中的广播数据格式

6. **电源管理**
   - 使用适当的连接参数
   - 在不需要时禁用蓝牙功能

7. **安全性**
   - 使用加密和身份验证
   - 实现适当的配对机制

8. **Mesh 网络**
   - 正确配置节点功能（中继、低功耗、好友节点等）
   - 实现适当的配置服务器模型

## 常见问题

1. **设备不可见**
   - 检查广播数据和参数
   - 验证蓝牙控制器是否正常工作

2. **连接失败**
   - 检查安全要求
   - 验证连接参数

3. **GATT 操作失败**
   - 检查权限设置
   - 验证特征属性

4. **Mesh 节点配置失败**
   - 检查配置服务器模型实现
   - 验证配置过程中的安全性

5. **性能问题**
   - 优化连接间隔和广播间隔
   - 减少不必要的 GATT 操作

## 总结

这些蓝牙示例展示了 Zephyr RTOS 中蓝牙功能的多个方面，包括：

1. 基本的 BLE 广播
2. GATT 服务的实现
3. BLE 中心和外围设备的角色
4. 蓝牙 Mesh 网络的基本实现

通过这些示例，您可以学习如何：

- 初始化和配置蓝牙协议栈
- 实现 BLE 广播和扫描
- 创建和使用 GATT 服务
- 管理 BLE 连接
- 设置基本的蓝牙 Mesh 节点

这些示例可以作为开发蓝牙应用的起点，您可以根据需要修改和扩展它们。记住要根据您的硬件和应用需求调整配置，并始终关注安全性和性能优化。