---
abbrlink: 2
title: drivers
date: '2025-03-21 20:49:56'
---
# Zephyr 驱动系统

Zephyr RTOS 提供了丰富的驱动系统，用于支持各种硬件外设。本文档将详细介绍 Zephyr 驱动系统的使用方法和常见外设驱动。

## 驱动框架

### 设备模型

Zephyr 的设备模型基于以下概念：

1. **设备对象**：表示一个硬件设备
2. **驱动 API**：定义设备操作接口
3. **设备树**：描述硬件配置

### 获取设备实例

```c
// 通过设备树标签获取设备
const struct device *dev = DEVICE_DT_GET(DT_NODELABEL(uart0));

// 检查设备是否就绪
if (!device_is_ready(dev)) {
    // 处理错误
    return;
}
```

### 设备初始化

设备初始化过程：

1. 系统启动时，按照初始化级别顺序初始化设备
2. 每个设备的初始化函数被调用
3. 设备初始化完成后标记为就绪

初始化级别：
- `PRE_KERNEL_1`：基础硬件初始化
- `PRE_KERNEL_2`：设备和驱动初始化
- `POST_KERNEL`：需要内核服务的设备
- `APPLICATION`：应用级设备

## 常用外设驱动

### GPIO 驱动

1. **配置 GPIO**

```c
#include <zephyr/drivers/gpio.h>

// 获取 GPIO 设备
const struct device *gpio = DEVICE_DT_GET(DT_NODELABEL(gpio0));

// 配置 GPIO 引脚
gpio_pin_configure(gpio, PIN, GPIO_OUTPUT_ACTIVE);
```

2. **控制 GPIO**

```c
// 设置引脚电平
gpio_pin_set(gpio, PIN, 1);  // 高电平
gpio_pin_set(gpio, PIN, 0);  // 低电平

// 读取引脚电平
int val = gpio_pin_get(gpio, PIN);
```

3. **GPIO 中断**

```c
// 中断回调函数
void gpio_callback(const struct device *dev, struct gpio_callback *cb,
                  uint32_t pins)
{
    // 处理中断
}

// 配置中断
struct gpio_callback gpio_cb;
gpio_init_callback(&gpio_cb, gpio_callback, BIT(PIN));
gpio_add_callback(gpio, &gpio_cb);
gpio_pin_interrupt_configure(gpio, PIN, GPIO_INT_EDGE_RISING);
```

### UART 驱动

1. **配置 UART**

```c
#include <zephyr/drivers/uart.h>

const struct device *uart = DEVICE_DT_GET(DT_NODELABEL(uart0));
```

2. **发送数据**

```c
// 发送单个字节
uart_poll_out(uart, 'A');

// 发送数据块
const char *data = "Hello, Zephyr!";
for (int i = 0; i < strlen(data); i++) {
    uart_poll_out(uart, data[i]);
}
```

3. **接收数据**

```c
// 接收单个字节
uint8_t c;
int ret = uart_poll_in(uart, &c);
if (ret == 0) {
    // 处理接收到的字符
}
```

4. **异步 UART**

```c
// 回调函数
static void uart_cb(const struct device *dev, struct uart_event *evt, void *user_data)
{
    switch (evt->type) {
    case UART_RX_RDY:
        // 处理接收到的数据
        break;
    case UART_TX_DONE:
        // 发送完成
        break;
    // 其他事件处理
    }
}

// 配置异步 UART
uart_callback_set(uart, uart_cb, NULL);
uart_rx_enable(uart, rx_buf, sizeof(rx_buf), 100);
uart_tx(uart, tx_buf, len, SYS_FOREVER_MS);
```

### SPI 驱动

1. **配置 SPI**

```c
#include <zephyr/drivers/spi.h>

const struct device *spi = DEVICE_DT_GET(DT_NODELABEL(spi0));

struct spi_config spi_cfg = {
    .frequency = 1000000,
    .operation = SPI_OP_MODE_MASTER | SPI_WORD_SET(8) | SPI_TRANSFER_MSB,
};
```

2. **SPI 传输**

```c
// 准备数据
uint8_t tx_buffer[] = {0x01, 0x02, 0x03};
uint8_t rx_buffer[sizeof(tx_buffer)];

struct spi_buf tx_buf = {
    .buf = tx_buffer,
    .len = sizeof(tx_buffer)
};
struct spi_buf rx_buf = {
    .buf = rx_buffer,
    .len = sizeof(rx_buffer)
};

struct spi_buf_set tx = {
    .buffers = &tx_buf,
    .count = 1
};
struct spi_buf_set rx = {
    .buffers = &rx_buf,
    .count = 1
};

// 执行传输
int ret = spi_transceive(spi, &spi_cfg, &tx, &rx);
if (ret == 0) {
    // 传输成功
}
```

### I2C 驱动

1. **配置 I2C**

```c
#include <zephyr/drivers/i2c.h>

const struct device *i2c = DEVICE_DT_GET(DT_NODELABEL(i2c0));
```

2. **I2C 读写**

```c
// 写数据
uint8_t data[] = {0x01, 0x02};
int ret = i2c_write(i2c, data, sizeof(data), SLAVE_ADDR);

// 读数据
uint8_t buffer[2];
ret = i2c_read(i2c, buffer, sizeof(buffer), SLAVE_ADDR);

// 写后读
uint8_t reg = 0x00;
ret = i2c_write_read(i2c, SLAVE_ADDR, &reg, 1, buffer, sizeof(buffer));
```

## 传感器子系统

### 传感器 API

```c
#include <zephyr/drivers/sensor.h>

const struct device *sensor = DEVICE_DT_GET(DT_NODELABEL(temp_sensor));

// 获取传感器数据
struct sensor_value temp;
sensor_sample_fetch(sensor);
sensor_channel_get(sensor, SENSOR_CHAN_AMBIENT_TEMP, &temp);

// 转换为浮点数
double temperature = sensor_value_to_double(&temp);
```

### 常见传感器类型

1. **温度传感器**
   - 环境温度 (`SENSOR_CHAN_AMBIENT_TEMP`)
   - 对象温度 (`SENSOR_CHAN_OBJ_TEMP`)

2. **加速度传感器**
   - X 轴加速度 (`SENSOR_CHAN_ACCEL_X`)
   - Y 轴加速度 (`SENSOR_CHAN_ACCEL_Y`)
   - Z 轴加速度 (`SENSOR_CHAN_ACCEL_Z`)
   - XYZ 轴加速度 (`SENSOR_CHAN_ACCEL_XYZ`)

3. **陀螺仪**
   - X 轴角速度 (`SENSOR_CHAN_GYRO_X`)
   - Y 轴角速度 (`SENSOR_CHAN_GYRO_Y`)
   - Z 轴角速度 (`SENSOR_CHAN_GYRO_Z`)
   - XYZ 轴角速度 (`SENSOR_CHAN_GYRO_XYZ`)

4. **其他传感器**
   - 气压传感器 (`SENSOR_CHAN_PRESS`)
   - 湿度传感器 (`SENSOR_CHAN_HUMIDITY`)
   - 光线传感器 (`SENSOR_CHAN_LIGHT`)

### 传感器触发

```c
// 触发回调函数
static void sensor_trigger_handler(const struct device *dev,
                                  const struct sensor_trigger *trigger)
{
    // 处理传感器触发事件
    struct sensor_value data;
    sensor_sample_fetch(dev);
    sensor_channel_get(dev, SENSOR_CHAN_AMBIENT_TEMP, &data);
}

// 配置触发
struct sensor_trigger trigger = {
    .type = SENSOR_TRIG_THRESHOLD,
    .chan = SENSOR_CHAN_AMBIENT_TEMP,
};
sensor_trigger_set(sensor, &trigger, sensor_trigger_handler);
```

## 存储驱动

### Flash 存储

```c
#include <zephyr/drivers/flash.h>

const struct device *flash_dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_flash_controller));

// 读取 Flash
uint8_t buffer[256];
flash_read(flash_dev, offset, buffer, sizeof(buffer));

// 擦除 Flash
flash_erase(flash_dev, offset, FLASH_SECTOR_SIZE);

// 写入 Flash
uint8_t data[] = {0x01, 0x02, 0x03, 0x04};
flash_write(flash_dev, offset, data, sizeof(data));
```

### EEPROM

```c
#include <zephyr/drivers/eeprom.h>

const struct device *eeprom = DEVICE_DT_GET(DT_NODELABEL(eeprom0));

// 读取 EEPROM
uint8_t buffer[16];
eeprom_read(eeprom, offset, buffer, sizeof(buffer));

// 写入 EEPROM
uint8_t data[] = {0x01, 0x02, 0x03, 0x04};
eeprom_write(eeprom, offset, data, sizeof(data));
```

## 电源管理

### 设备电源管理

```c
#include <zephyr/pm/device.h>

// 设置设备电源状态
pm_device_state_set(dev, PM_DEVICE_STATE_LOW_POWER);

// 获取设备电源状态
enum pm_device_state state;
pm_device_state_get(dev, &state);
```

## 最佳实践

1. **设备树配置**
   - 使用设备树配置硬件
   - 避免硬编码硬件参数
   - 利用设备树覆盖文件定制配置

2. **错误处理**
   - 检查设备是否就绪
   - 处理所有驱动 API 返回的错误码
   - 实现适当的错误恢复机制

3. **资源管理**
   - 合理使用中断和 DMA
   - 避免长时间阻塞
   - 使用异步 API 提高效率

4. **电源优化**
   - 不使用时禁用外设
   - 使用低功耗模式
   - 优化数据传输批量处理

## 常见问题

1. **设备未就绪**
   - 检查设备树配置
   - 确认驱动已启用
   - 验证硬件连接

2. **通信错误**
   - 检查通信参数（波特率、极性等）
   - 验证设备地址
   - 检查时序要求

3. **中断问题**
   - 确认中断配置正确
   - 检查中断优先级
   - 避免中断处理函数中的长时间操作

4. **DMA 传输失败**
   - 检查内存对齐
   - 验证 DMA 通道配置
   - 确保缓冲区在传输期间有效

## 总结

Zephyr 驱动系统提供了丰富的硬件抽象层，简化了与外设的交互。通过使用标准化的驱动 API，可以开发出可移植、可维护的嵌入式应用。深入理解这些驱动接口对于开发高质量的 Zephyr 应用至关重要。