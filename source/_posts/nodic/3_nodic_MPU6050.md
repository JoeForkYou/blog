---
title: nRF52832开发入门【三】MPU6050六轴传感器应用
tags: nodic
categories: nodic
abbrlink: 36538
date: 2025-03-14 20:11:26
---

# 1 说明

MPU6050是一款广泛使用的6轴运动跟踪设备，由InvenSense公司开发。它集成了3轴加速度计和3轴陀螺仪，能够精确测量物体的加速度和角速度。

在Zephyr RTOS中的实现：

- 驱动文件位置：`zephyr/drivers/sensor/tdk/mpu6050/`
- 主要文件：
  - `mpu6050.c`：主驱动实现
  - `mpu6050.h`：头文件定义
  - `mpu6050_trigger.c`：中断触发相关功能
  - `Kconfig`：驱动配置选项

官方有MPU6050的示例demo，路径如下:

C:\ncs\v2.9.1\zephyr\samples\sensor\mpu6050

拿到这个代码，在这个代码的基础上创建创建自己的工程文件。

代码结构很简单,其中多一个boards文件夹，下面有个nrf52dk_nrf52832.overlay文件，方便我们对自己板子的内容进行重构.

目标是剥离出6050的相关驱动，使得main.c中仅仅保留干净的代码

# 2 引脚配置

官方默认的dts配置是

| i2c0 |       |
| ---- | ----- |
| SDA  | P0.26 |
| SCL  | P0.27 |

详细的dts路劲和内容如下:

![](https://s3.bmp.ovh/imgs/2025/03/14/13f20d95ff33c829.png)

根据实际上自己板子上的外设连接的GPIO脚进行修改后保存

![](https://s3.bmp.ovh/imgs/2025/03/14/75c2423959e4eabd.png)

对应的dts文件

nrf52dk_nrf52832.overlay 也会新增相关的内容如下:

```dts
/*
 * Copyright (c) 2019 Nordic Semiconductor ASA
 *
 * SPDX-License-Identifier: Apache-2.0
 */

&i2c0 {
	mpu6050@68 {
		compatible = "invensense,mpu6050";
		reg = <0x68>;
		status = "okay";
		int-gpios = <&gpio0 11 GPIO_ACTIVE_HIGH>;
	};
};

&i2c0_default {
    group1 {
        psels = <NRF_PSEL(TWIM_SDA, 0, 27)>, <NRF_PSEL(TWIM_SCL, 0, 25)>;
    };
};
```

因为我的外设连接SDA是P027,SCL是P025,所以做如上的修改.

# 3 项目配置

我只是做个demo看数据，先把触发模式关掉

```
CONFIG_MPU6050_TRIGGER_NONE=y
```

# 4 项目说明

官方demo是直接放在

process_mpu6050 函数中获取不同通道的数据.

直接看我重构后的代码

mpu6050.c

```c
#include "mpu6050.h"

const char *now_str(void)
{
    static char buf[16]; /* ...HH:MM:SS.MMM */
    uint32_t now = k_uptime_get_32();
    unsigned int ms = now % MSEC_PER_SEC;
    unsigned int s;
    unsigned int min;
    unsigned int h;

    now /= MSEC_PER_SEC;
    s = now % 60U;
    now /= 60U;
    min = now % 60U;
    now /= 60U;
    h = now;

    snprintf(buf, sizeof(buf), "%u:%02u:%02u.%03u",
             h, min, s, ms);
    return buf;
}

int process_mpu6050(const struct device *dev)
{
    struct sensor_value temperature;
    struct sensor_value accel[3];
    struct sensor_value gyro[3];
    int rc = sensor_sample_fetch(dev);

    if (rc == 0)
    {
        rc = sensor_channel_get(dev, SENSOR_CHAN_ACCEL_XYZ,
                                accel);
    }
    if (rc == 0)
    {
        rc = sensor_channel_get(dev, SENSOR_CHAN_GYRO_XYZ,
                                gyro);
    }
    if (rc == 0)
    {
        rc = sensor_channel_get(dev, SENSOR_CHAN_DIE_TEMP,
                                &temperature);
    }
    if (rc == 0)
    {
        printf("[%s]:%g Cel\n"
               "  accel %f %f %f m/s/s\n"
               "  gyro  %f %f %f rad/s\n",
               now_str(),
               sensor_value_to_double(&temperature),
               sensor_value_to_double(&accel[0]),
               sensor_value_to_double(&accel[1]),
               sensor_value_to_double(&accel[2]),
               sensor_value_to_double(&gyro[0]),
               sensor_value_to_double(&gyro[1]),
               sensor_value_to_double(&gyro[2]));
    }
    else
    {
        printf("sample fetch/get failed: %d\n", rc);
    }

    return rc;
}

#ifdef CONFIG_MPU6050_TRIGGER
struct sensor_trigger trigger;
void handle_mpu6050_drdy(const struct device *dev,
                         const struct sensor_trigger *trig)
{
    int rc = process_mpu6050(dev);

    if (rc != 0)
    {
        printf("cancelling trigger due to failure: %d\n", rc);
        (void)sensor_trigger_set(dev, trig, NULL);
        return;
    }
}
#endif /* CONFIG_MPU6050_TRIGGER */
```

mpu6050.h

```
#ifndef __MPU6050_H__
#define __MPU6050_H__

#include <zephyr/types.h>
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>
#include <stdio.h>

const char *now_str(void);
int process_mpu6050(const struct device *dev);
#ifdef CONFIG_MPU6050_TRIGGER
extern struct sensor_trigger trigger;
void handle_mpu6050_drdy(const struct device *dev,const struct sensor_trigger *trig);
#endif
#endif /* __MPU6050_H__ */
```

main.c

```c
/*
 * Copyright (c) 2019 Nordic Semiconductor ASA
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>
#include <stdio.h>
#include "mpu6050.h"


int main(void)
{
    const struct device *const mpu6050 = DEVICE_DT_GET_ONE(invensense_mpu6050);

    if (!device_is_ready(mpu6050))
    {
        printf("Device %s is not ready\n", mpu6050->name);
        return 0;
    }

#ifdef CONFIG_MPU6050_TRIGGER
    trigger = (struct sensor_trigger){
        .type = SENSOR_TRIG_DATA_READY,
        .chan = SENSOR_CHAN_ALL,
    };
    if (sensor_trigger_set(mpu6050, &trigger,
                           handle_mpu6050_drdy) < 0)
    {
        printf("Cannot configure trigger\n");
        return 0;
    }
    printk("Configured for triggered sampling.\n");
#endif

    while (!IS_ENABLED(CONFIG_MPU6050_TRIGGER))
    {
        int rc = process_mpu6050(mpu6050);

        if (rc != 0)
        {
            break;
        }
        k_sleep(K_SECONDS(2));
    }

    /* triggered runs with its own thread after exit */
    return 0;
}

```

# 5 驱动说明

```c
LOG_MODULE_REGISTER(MPU6050, CONFIG_SENSOR_LOG_LEVEL);
```



这是官方内部配置的驱动.

```c
int mpu6050_init(const struct device *dev)
{
	struct mpu6050_data *drv_data = dev->data;
	const struct mpu6050_config *cfg = dev->config;
	uint8_t id, i;

	if (!device_is_ready(cfg->i2c.bus)) {
		LOG_ERR("Bus device is not ready");
		return -ENODEV;
	}

	/* check chip ID */
	if (i2c_reg_read_byte_dt(&cfg->i2c, MPU6050_REG_CHIP_ID, &id) < 0) {
		LOG_ERR("Failed to read chip ID.");
		return -EIO;
	}

	if (id == MPU6050_CHIP_ID || id == MPU9250_CHIP_ID || id == MPU6880_CHIP_ID) {
		LOG_DBG("MPU6050/MPU9250/MPU6880 detected");
		drv_data->device_type = DEVICE_TYPE_MPU6050;
	} else if (id == MPU6500_CHIP_ID) {
		LOG_DBG("MPU6500 detected");
		drv_data->device_type = DEVICE_TYPE_MPU6500;
	} else {
		LOG_ERR("Invalid chip ID.");
		return -EINVAL;
	}

	/* wake up chip */
	if (i2c_reg_update_byte_dt(&cfg->i2c, MPU6050_REG_PWR_MGMT1,
				   MPU6050_SLEEP_EN, 0) < 0) {
		LOG_ERR("Failed to wake up chip.");
		return -EIO;
	}

	/* set accelerometer full-scale range */
	for (i = 0U; i < 4; i++) {
		if (BIT(i+1) == CONFIG_MPU6050_ACCEL_FS) {
			break;
		}
	}

	if (i == 4U) {
		LOG_ERR("Invalid value for accel full-scale range.");
		return -EINVAL;
	}

	if (i2c_reg_write_byte_dt(&cfg->i2c, MPU6050_REG_ACCEL_CFG,
				  i << MPU6050_ACCEL_FS_SHIFT) < 0) {
		LOG_ERR("Failed to write accel full-scale range.");
		return -EIO;
	}

	drv_data->accel_sensitivity_shift = 14 - i;

	/* set gyroscope full-scale range */
	for (i = 0U; i < 4; i++) {
		if (BIT(i) * 250 == CONFIG_MPU6050_GYRO_FS) {
			break;
		}
	}

	if (i == 4U) {
		LOG_ERR("Invalid value for gyro full-scale range.");
		return -EINVAL;
	}

	if (i2c_reg_write_byte_dt(&cfg->i2c, MPU6050_REG_GYRO_CFG,
				  i << MPU6050_GYRO_FS_SHIFT) < 0) {
		LOG_ERR("Failed to write gyro full-scale range.");
		return -EIO;
	}

	drv_data->gyro_sensitivity_x10 = mpu6050_gyro_sensitivity_x10[i];

#ifdef CONFIG_MPU6050_TRIGGER
	if (cfg->int_gpio.port) {
		if (mpu6050_init_interrupt(dev) < 0) {
			LOG_DBG("Failed to initialize interrupts.");
			return -EIO;
		}
	}
#endif

	return 0;
}
```

# 干净点的重构说明

仅保留main.c

```c
#include <zephyr/drivers/i2c.h>
#include <zephyr/sys/printk.h>
#include <zephyr/logging/log.h>
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/sensor.h>

#define LOG_MODULE_NAME i2c_module

LOG_MODULE_REGISTER(i2c_module, LOG_LEVEL_INF);

#define MPU6050_ADDR 0x68
#define MPU_PWR_MGMT1_REG 0x6B

int main(void)
{
    LOG_INF("Starting MPU6050 application");

    // 获取MPU6050设备
    const struct device *mpu = DEVICE_DT_GET_ONE(invensense_mpu6050);
    if (!device_is_ready(mpu))
    {
        LOG_ERR("MPU6050 device not ready");
        return 1;
    }

    LOG_INF("MPU6050 device %s is ready", mpu->name);

    // 读取传感器数据
    struct sensor_value accel[3], gyro[3], temp;

    while (1)
    {
        // 获取传感器数据
        if (sensor_sample_fetch(mpu) < 0)
        {
            LOG_ERR("Failed to fetch sensor data");
            continue;
        }

        // 读取加速度数据
        sensor_channel_get(mpu, SENSOR_CHAN_ACCEL_XYZ, accel);

        // 读取陀螺仪数据
        sensor_channel_get(mpu, SENSOR_CHAN_GYRO_XYZ, gyro);

        // 读取温度数据
        sensor_channel_get(mpu, SENSOR_CHAN_DIE_TEMP, &temp);

        LOG_INF("Accel (m/s^2): X=%d.%06d, Y=%d.%06d, Z=%d.%06d",
                accel[0].val1, accel[0].val2, accel[1].val1, accel[1].val2, accel[2].val1, accel[2].val2);

        LOG_INF("Gyro (dps): X=%d.%06d, Y=%d.%06d, Z=%d.%06d",
                gyro[0].val1, gyro[0].val2, gyro[1].val1, gyro[1].val2, gyro[2].val1, gyro[2].val2);

        LOG_INF("Temperature (Celsius): %d.%06d", temp.val1, temp.val2);

        k_sleep(K_MSEC(1000));
    }
}

```


