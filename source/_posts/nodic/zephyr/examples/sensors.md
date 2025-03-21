---
abbrlink: 29
title: sensors
date: '2025-03-21 20:49:56'
---
# Zephyr 传感器示例

本文档提供了 Zephyr RTOS 中使用各种传感器的示例代码，包括温度传感器、加速度传感器、压力传感器、光线传感器以及传感器数据融合等内容。

## 温度传感器

这个示例展示了如何读取温度传感器数据。

### 源代码

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>

void main(void)
{
    const struct device *dev = DEVICE_DT_GET_ANY(ti_tmp116);
    struct sensor_value temp;

    if (!device_is_ready(dev)) {
        printk("Device %s is not ready\n", dev->name);
        return;
    }

    while (1) {
        /* 获取温度数据 */
        if (sensor_sample_fetch(dev) < 0) {
            printk("Failed to fetch sample\n");
            continue;
        }

        if (sensor_channel_get(dev, SENSOR_CHAN_AMBIENT_TEMP, &temp) < 0) {
            printk("Failed to get temperature\n");
            continue;
        }

        /* 打印温度值 */
        printk("Temperature: %.2f °C\n",
               sensor_value_to_double(&temp));

        /* 延时 1 秒 */
        k_sleep(K_SECONDS(1));
    }
}
```

### 配置文件

**prj.conf**:
```
CONFIG_SENSOR=y
CONFIG_TMP116=y
```

**overlay-tmp116.conf**:
```dts
/ {
    aliases {
        tempsensor0 = &tmp116;
    };
};

&i2c0 {
    status = "okay";
    tmp116: tmp116@48 {
        compatible = "ti,tmp116";
        reg = <0x48>;
        status = "okay";
    };
};
```

## 加速度传感器

这个示例展示了如何读取加速度传感器数据。

### 源代码

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>

/* 触发回调函数 */
static void trigger_handler(const struct device *dev,
                          const struct sensor_trigger *trig)
{
    struct sensor_value accel[3];

    /* 获取加速度数据 */
    if (sensor_sample_fetch(dev) < 0) {
        printk("Failed to fetch sample\n");
        return;
    }

    if (sensor_channel_get(dev, SENSOR_CHAN_ACCEL_XYZ, accel) < 0) {
        printk("Failed to get acceleration\n");
        return;
    }

    /* 打印加速度值 */
    printk("x: %.2f , y: %.2f , z: %.2f (m/s^2)\n",
           sensor_value_to_double(&accel[0]),
           sensor_value_to_double(&accel[1]),
           sensor_value_to_double(&accel[2]));
}

void main(void)
{
    const struct device *dev = DEVICE_DT_GET_ANY(st_lis2dh);
    struct sensor_trigger trig = {
        .type = SENSOR_TRIG_DATA_READY,
        .chan = SENSOR_CHAN_ACCEL_XYZ,
    };

    if (!device_is_ready(dev)) {
        printk("Device %s is not ready\n", dev->name);
        return;
    }

    /* 配置触发器 */
    if (sensor_trigger_set(dev, &trig, trigger_handler) < 0) {
        printk("Failed to set trigger\n");
        return;
    }

    while (1) {
        k_sleep(K_SECONDS(1));
    }
}
```

### 配置文件

**prj.conf**:
```
CONFIG_SENSOR=y
CONFIG_LIS2DH=y
```

**overlay-lis2dh.conf**:
```dts
/ {
    aliases {
        accel0 = &lis2dh;
    };
};

&i2c0 {
    status = "okay";
    lis2dh: lis2dh@18 {
        compatible = "st,lis2dh";
        reg = <0x18>;
        irq-gpios = <&gpio0 3 GPIO_ACTIVE_HIGH>;
        status = "okay";
    };
};
```

## 压力传感器

这个示例展示了如何读取压力传感器数据。

### 源代码

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>

void main(void)
{
    const struct device *dev = DEVICE_DT_GET_ANY(bosch_bmp280);
    struct sensor_value pressure, temp;

    if (!device_is_ready(dev)) {
        printk("Device %s is not ready\n", dev->name);
        return;
    }

    while (1) {
        /* 获取压力和温度数据 */
        if (sensor_sample_fetch(dev) < 0) {
            printk("Failed to fetch sample\n");
            continue;
        }

        if (sensor_channel_get(dev, SENSOR_CHAN_PRESS, &pressure) < 0) {
            printk("Failed to get pressure\n");
            continue;
        }

        if (sensor_channel_get(dev, SENSOR_CHAN_AMBIENT_TEMP, &temp) < 0) {
            printk("Failed to get temperature\n");
            continue;
        }

        /* 打印数据 */
        printk("Pressure: %.2f kPa\n",
               sensor_value_to_double(&pressure));
        printk("Temperature: %.2f °C\n",
               sensor_value_to_double(&temp));

        k_sleep(K_SECONDS(1));
    }
}
```

### 配置文件

**prj.conf**:
```
CONFIG_SENSOR=y
CONFIG_BMP280=y
```

**overlay-bmp280.conf**:
```dts
/ {
    aliases {
        pressure0 = &bmp280;
    };
};

&i2c0 {
    status = "okay";
    bmp280: bmp280@76 {
        compatible = "bosch,bmp280";
        reg = <0x76>;
        status = "okay";
    };
};
```

## 光线传感器

这个示例展示了如何读取光线传感器数据。

### 源代码

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>

void main(void)
{
    const struct device *dev = DEVICE_DT_GET_ANY(maxim_max44009);
    struct sensor_value light;

    if (!device_is_ready(dev)) {
        printk("Device %s is not ready\n", dev->name);
        return;
    }

    while (1) {
        /* 获取光线数据 */
        if (sensor_sample_fetch(dev) < 0) {
            printk("Failed to fetch sample\n");
            continue;
        }

        if (sensor_channel_get(dev, SENSOR_CHAN_LIGHT, &light) < 0) {
            printk("Failed to get light level\n");
            continue;
        }

        /* 打印光线值 */
        printk("Light level: %.2f lux\n",
               sensor_value_to_double(&light));

        k_sleep(K_SECONDS(1));
    }
}
```

### 配置文件

**prj.conf**:
```
CONFIG_SENSOR=y
CONFIG_MAX44009=y
```

**overlay-max44009.conf**:
```dts
/ {
    aliases {
        light0 = &max44009;
    };
};

&i2c0 {
    status = "okay";
    max44009: max44009@4a {
        compatible = "maxim,max44009";
        reg = <0x4a>;
        status = "okay";
    };
};
```

## 传感器数据融合

这个示例展示了如何组合多个传感器的数据。

### 源代码

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>

/* 定义传感器数据结构 */
struct sensor_data {
    struct sensor_value temperature;
    struct sensor_value pressure;
    struct sensor_value humidity;
    struct sensor_value accel[3];
};

/* 读取所有传感器数据 */
static int read_sensor_data(const struct device *temp_dev,
                          const struct device *press_dev,
                          const struct device *humid_dev,
                          const struct device *accel_dev,
                          struct sensor_data *data)
{
    /* 读取温度 */
    if (sensor_sample_fetch(temp_dev) < 0) {
        return -1;
    }
    if (sensor_channel_get(temp_dev, SENSOR_CHAN_AMBIENT_TEMP,
                          &data->temperature) < 0) {
        return -1;
    }

    /* 读取压力 */
    if (sensor_sample_fetch(press_dev) < 0) {
        return -1;
    }
    if (sensor_channel_get(press_dev, SENSOR_CHAN_PRESS,
                          &data->pressure) < 0) {
        return -1;
    }

    /* 读取湿度 */
    if (sensor_sample_fetch(humid_dev) < 0) {
        return -1;
    }
    if (sensor_channel_get(humid_dev, SENSOR_CHAN_HUMIDITY,
                          &data->humidity) < 0) {
        return -1;
    }

    /* 读取加速度 */
    if (sensor_sample_fetch(accel_dev) < 0) {
        return -1;
    }
    if (sensor_channel_get(accel_dev, SENSOR_CHAN_ACCEL_XYZ,
                          data->accel) < 0) {
        return -1;
    }

    return 0;
}

/* 处理传感器数据 */
static void process_sensor_data(const struct sensor_data *data)
{
    /* 计算海拔（使用温度和压力） */
    double temperature = sensor_value_to_double(&data->temperature);
    double pressure = sensor_value_to_double(&data->pressure);
    double altitude = 44330.0 * (1.0 - pow(pressure / 101.325,
                                          1.0 / 5.255));

    /* 计算露点（使用温度和湿度） */
    double humidity = sensor_value_to_double(&data->humidity);
    double a = 17.27;
    double b = 237.7;
    double alpha = ((a * temperature) / (b + temperature)) +
                   log(humidity / 100.0);
    double dew_point = (b * alpha) / (a - alpha);

    /* 计算倾斜角度（使用加速度） */
    double ax = sensor_value_to_double(&data->accel[0]);
    double ay = sensor_value_to_double(&data->accel[1]);
    double az = sensor_value_to_double(&data->accel[2]);
    double pitch = atan2(ax, sqrt(ay * ay + az * az)) * 180.0 / M_PI;
    double roll = atan2(ay, sqrt(ax * ax + az * az)) * 180.0 / M_PI;

    /* 打印结果 */
    printk("Environmental Data:\n");
    printk("  Temperature: %.2f °C\n", temperature);
    printk("  Pressure: %.2f kPa\n", pressure);
    printk("  Humidity: %.2f %%\n", humidity);
    printk("  Altitude: %.2f m\n", altitude);
    printk("  Dew Point: %.2f °C\n", dew_point);
    printk("Motion Data:\n");
    printk("  Pitch: %.2f degrees\n", pitch);
    printk("  Roll: %.2f degrees\n", roll);
}

void main(void)
{
    const struct device *temp_dev = DEVICE_DT_GET_ANY(ti_tmp116);
    const struct device *press_dev = DEVICE_DT_GET_ANY(bosch_bmp280);
    const struct device *humid_dev = DEVICE_DT_GET_ANY(ti_hdc1080);
    const struct device *accel_dev = DEVICE_DT_GET_ANY(st_lis2dh);
    struct sensor_data data;

    /* 检查设备是否就绪 */
    if (!device_is_ready(temp_dev) || !device_is_ready(press_dev) ||
        !device_is_ready(humid_dev) || !device_is_ready(accel_dev)) {
        printk("One or more devices are not ready\n");
        return;
    }

    while (1) {
        /* 读取所有传感器数据 */
        if (read_sensor_data(temp_dev, press_dev, humid_dev,
                           accel_dev, &data) < 0) {
            printk("Failed to read sensor data\n");
            continue;
        }

        /* 处理数据 */
        process_sensor_data(&data);

        /* 延时 */
        k_sleep(K_SECONDS(5));
    }
}
```

### 配置文件

**prj.conf**:
```
CONFIG_SENSOR=y
CONFIG_TMP116=y
CONFIG_BMP280=y
CONFIG_HDC1080=y
CONFIG_LIS2DH=y
CONFIG_NEWLIB_LIBC=y
```

**overlay-sensors.conf**:
```dts
/ {
    aliases {
        tempsensor0 = &tmp116;
        pressure0 = &bmp280;
        humidity0 = &hdc1080;
        accel0 = &lis2dh;
    };
};

&i2c0 {
    status = "okay";
    
    tmp116: tmp116@48 {
        compatible = "ti,tmp116";
        reg = <0x48>;
        status = "okay";
    };

    bmp280: bmp280@76 {
        compatible = "bosch,bmp280";
        reg = <0x76>;
        status = "okay";
    };

    hdc1080: hdc1080@40 {
        compatible = "ti,hdc1080";
        reg = <0x40>;
        status = "okay";
    };

    lis2dh: lis2dh@18 {
        compatible = "st,lis2dh";
        reg = <0x18>;
        irq-gpios = <&gpio0 3 GPIO_ACTIVE_HIGH>;
        status = "okay";
    };
};
```

## 编译和运行

```bash
# 编译温度传感器示例
west build -b <board> -- -DOVERLAY_CONFIG=overlay-tmp116.conf

# 编译加速度传感器示例
west build -b <board> -- -DOVERLAY_CONFIG=overlay-lis2dh.conf

# 编译压力传感器示例
west build -b <board> -- -DOVERLAY_CONFIG=overlay-bmp280.conf

# 编译光线传感器示例
west build -b <board> -- -DOVERLAY_CONFIG=overlay-max44009.conf

# 编译传感器融合示例
west build -b <board> -- -DOVERLAY_CONFIG=overlay-sensors.conf

# 烧录到开发板
west flash
```

## 最佳实践

1. **初始化检查**
   - 始终检查设备是否就绪
   - 验证传感器配置是否正确
   - 处理初始化错误

2. **数据采集**
   - 使用适当的采样率
   - 处理采样错误
   - 验证数据有效性

3. **数据处理**
   - 实现数据滤波
   - 校准传感器
   - 处理异常值

4. **错误处理**
   - 实现错误恢复机制
   - 记录错误信息
   - 保持系统稳定

5. **电源管理**
   - 优化采样频率
   - 使用低功耗模式
   - 管理传感器电源状态

## 常见问题

1. **传感器不响应**
   - 检查 I2C/SPI 配置
   - 验证设备地址
   - 检查电源连接

2. **数据异常**
   - 检查传感器位置
   - 验证采样配置
   - 实现数据过滤

3. **性能问题**
   - 优化采样频率
   - 减少处理开销
   - 使用中断而不是轮询

4. **功耗问题**
   - 使用低功耗模式
   - 优化采样策略
   - 关闭不需要的功能

## 总结

这些传感器示例展示了如何在 Zephyr RTOS 中使用各种传感器，从基本的数据采集到复杂的数据融合。通过这些示例，您可以学习：

1. 如何初始化和配置不同类型的传感器
2. 如何读取和处理传感器数据
3. 如何实现传感器数据融合
4. 如何处理错误和异常情况

这些示例可以作为开发传感器应用的起点，您可以根据需要修改和扩展它们。记住要根据您的硬件配置正确的设备树覆盖文件，并确保所有必要的驱动程序都已启用。