---
title: 1nRF52832开发入门【一】
tags: nodic
categories: nodic
abbrlink: 13406
date: 2025-03-07 19:59:40
---
# 1 Zephyr简要说明

官方没有log 加载的机制说明

只能自己对着代码看了,mark下做个记录

Zephyr 是一个开源的实时操作系统（RTOS），专为资源受限的设备和物联网应用设计。它支持多种硬件平台，并提供了丰富的功能和模块来简化嵌入式开发。以下是一些关于 Zephyr 的关键点和使用技巧：

### 1. Zephyr 基本概念
- **内核**：Zephyr 提供了一个轻量级的实时内核，支持多线程、中断处理、定时器等功能。
- **模块**：Zephyr 包含了大量的模块，如蓝牙、Wi-Fi、USB、文件系统等，方便开发者快速集成各种功能。
- **配置系统**：Zephyr 使用 Kconfig 和 CMake 来管理项目的配置和构建过程。

### 2. 日志系统
Zephyr 提供了强大的日志系统，可以帮助开发者调试和监控应用程序。以下是使用日志系统的步骤：

#### 2.1 配置日志系统
在 `prj.conf` 文件中启用日志功能：
```conf
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4  # 4 表示信息级别 (INFO)
CONFIG_LOG_BACKEND_UART=y   # 使用 UART 输出日志
CONFIG_UART_CONSOLE=y       # 启用 UART 控制台
```

#### 2.2 注册日志模块
在源文件中注册日志模块：
```c
#define LOG_MODULE_NAME my_module
LOG_MODULE_REGISTER(LOG_MODULE_NAME);
```

#### 2.3 使用日志宏
在代码中使用不同的日志宏输出信息：
```c
LOG_INF("This is an info message");
LOG_DBG("This is a debug message");
LOG_ERR("This is an error message");
```

### 3. 蓝牙功能
Zephyr 支持蓝牙低功耗（BLE）协议栈，可以用于开发各种蓝牙设备。以下是一个简单的 BLE 外设示例：

#### 3.1 初始化蓝牙
```c
int err = bt_enable(NULL);
if (err) {
    printk("Bluetooth init failed (err %d)\n", err);
    return;
}
printk("Bluetooth initialized\n");
```

#### 3.2 广播设置
```c
static const struct bt_data ad[] = {
    BT_DATA_BYTES(BT_DATA_GAP_APPEARANCE,
                  (CONFIG_BT_DEVICE_APPEARANCE >> 0) & 0xff,
                  (CONFIG_BT_DEVICE_APPEARANCE >> 8) & 0xff),
    BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
    BT_DATA_BYTES(BT_DATA_UUID16_ALL, BT_UUID_16_ENCODE(BT_UUID_HIDS_VAL),
                                  BT_UUID_16_ENCODE(BT_UUID_BAS_VAL)),
};

static const struct bt_data sd[] = {
    BT_DATA(BT_DATA_NAME_COMPLETE, DEVICE_NAME, DEVICE_NAME_LEN),
};
```

#### 3.3 连接回调
```c
static void connected(struct bt_conn *conn, uint8_t err) {
    char addr[BT_ADDR_LE_STR_LEN];

    if (err) {
        printk("Connection failed (err %d)\n", err);
        return;
    }

    bt_addr_le_to_str(bt_conn_get_dst(conn), addr, sizeof(addr));
    printk("Connected to %s\n", addr);
}

BT_CONN_CB_DEFINE(conn_callbacks) = {
    .connected = connected,
};
```

### 4. 工作队列和消息队列
Zephyr 提供了工作队列和消息队列机制，方便任务调度和异步处理。

#### 4.1 定义和初始化工作队列
```c
static struct k_work hids_work;

void mouse_handler(struct k_work *work) {
    // 处理鼠标事件
}

// 在 main 函数中初始化工作队列
k_work_init(&hids_work, mouse_handler);
```

#### 4.2 消息队列
```c
K_MSGQ_DEFINE(hids_queue, sizeof(struct mouse_pos), HIDS_QUEUE_SIZE, 4);

struct mouse_pos pos;
pos.x_val = 10;
pos.y_val = 20;

// 发送消息到队列
k_msgq_put(&hids_queue, &pos, K_NO_WAIT);

// 从队列中获取消息
k_msgq_get(&hids_queue, &pos, K_NO_WAIT);
```

### 5. 配置系统
Zephyr 使用 Kconfig 和 CMake 来管理项目的配置和构建过程。

#### 5.1 Kconfig 配置
在 `prj.conf` 文件中添加或修改配置项：
```conf
CONFIG_BT=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=4
CONFIG_UART_CONSOLE=y
```

#### 5.2 CMake 构建
在 `CMakeLists.txt` 文件中指定项目名称和源文件：
```cmake
cmake_minimum_required(VERSION 3.13.1)

find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(my_project)

target_sources(app PRIVATE src/main.c)
```

### 6. 常见问题排查
- **日志无法输出**：检查 `prj.conf` 中的日志配置是否正确，确保日志模块已注册并初始化。
- **蓝牙连接失败**：检查广播数据和连接回调函数是否正确配置，确保蓝牙已成功初始化。
- **内存不足**：优化代码，减少不必要的内存分配，检查堆栈大小配置。

通过以上介绍，你应该对 Zephyr 有了更深入的了解，并能够更好地利用其功能进行嵌入式开发。如果遇到具体问题，可以根据上述步骤进行排查和解决。

官方基础示例的代码路径如下:

```shell
C:\ncs\v2.9.1\zephyr\samples\basic
```

# 2 实操

按照如下的步骤先建立一个新的干净的工程.

![](https://s3.bmp.ovh/imgs/2025/03/07/eb96cab6ffd0eee0.png)

![](https://s3.bmp.ovh/imgs/2025/03/07/dd1c851959223afa.png)

先创建一个空的工程

#### 2.1 配置日志系统

在 `prj.conf` 文件中启用日志功能：

```conf
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3  # 4 表示信息级别 (INFO)
CONFIG_LOG_BACKEND_UART=y   # 使用 UART 输出日志
CONFIG_UART_CONSOLE=y       # 启用 UART 控制台
```

#### 2.2 注册日志模块

在源文件中注册日志模块：

```c
#define LOG_MODULE_NAME my_module
LOG_MODULE_REGISTER(LOG_MODULE_NAME);
```

#### 2.3 使用日志宏

在代码中使用不同的日志宏输出信息：

```c
#include <zephyr/logging/log.h>
LOG_INF("This is an info message");
LOG_DBG("This is a debug message");
LOG_ERR("This is an error message");
```
添加如上信息后发现日志打不完整
[00:00:00.384,246] <dbg> os: k_sched_unlock: scheduler unlocked (0x20000b30:0)
[00:00:00.384,246] <inf> my_module: This is an info1 message
[00:00:00.384,246] <inf> my_module: This is [0m

然后我修改下log等级,并且添加缓冲区，就能正确打印log了.

CONFIG_LOG_DEFAULT_LEVEL=3 # 等级3为LOG_INF

```
CONFIG_LOG_BUFFER_SIZE=4096
```

在 Zephyr 中，日志级别通常定义如下（数值越小，级别越低，日志输出少）：

- 0: OFF（关闭日志）
- 1: ERROR（错误日志）
- 2: WARNING（警告日志）
- 3: INFO（信息日志）
- 4: DEBUG（调试日志）

# 3 LED

由于到鼠标之前 还要掌握不少的内容，就边写 边加(注意这边prj.conf已经添加了log相关使能)

先load 官方demo **blinky**

## 3.1 官方demo

官方的路径默认在:

```
C:\ncs\v2.9.1\zephyr\samples\basic\blinky
```

这边看了官方默认的例子就是单led的闪烁,还是比较简单的 ，可以根据相关的dts，把其他led补上去

```c
#define LED0_NODE DT_ALIAS(led0)
```

```dts
	aliases {
		led0 = &led0;
		led1 = &led1;
		led2 = &led2;
		led3 = &led3;
		pwm-led0 = &pwm_led0;
		sw0 = &button0;
		sw1 = &button1;
		sw2 = &button2;
		sw3 = &button3;
		bootloader-led0 = &led0;
		mcuboot-button0 = &button0;
		mcuboot-led0 = &led0;
		watchdog0 = &wdt0;
	};
```

led相关dts配置

```dts
	leds {
		compatible = "gpio-leds";
		led0: led_0 {
			gpios = <&gpio0 17 GPIO_ACTIVE_LOW>;
			label = "Green LED 0";
		};
		led1: led_1 {
			gpios = <&gpio0 18 GPIO_ACTIVE_LOW>;
			label = "Green LED 1";
		};
		led2: led_2 {
			gpios = <&gpio0 19 GPIO_ACTIVE_LOW>;
			label = "Green LED 2";
		};
		led3: led_3 {
			gpios = <&gpio0 20 GPIO_ACTIVE_LOW>;
			label = "Green LED 3";
		};
	};
```

修修改改完整的led代码如下,可以同时使用四盏灯

```c
/*
 * Copyright (c) 2016 Intel Corporation
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <stdio.h>
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/logging/log.h>

/* 1000 msec = 1 sec */
#define SLEEP_TIME_MS   1000

/* The devicetree node identifier for the "led0" alias. */
#define LED0_NODE DT_ALIAS(led0)
#define LED1_NODE DT_ALIAS(led1)
#define LED2_NODE DT_ALIAS(led2)
#define LED3_NODE DT_ALIAS(led3)
#define LOG_MODULE_NAME my_module
LOG_MODULE_REGISTER(LOG_MODULE_NAME);
/*
 * A build error on this line means your board is unsupported.
 * See the sample documentation for information on how to fix this.
 */
static const struct gpio_dt_spec led0 = GPIO_DT_SPEC_GET(LED0_NODE, gpios);
static const struct gpio_dt_spec led1 = GPIO_DT_SPEC_GET(LED1_NODE, gpios);
static const struct gpio_dt_spec led2 = GPIO_DT_SPEC_GET(LED2_NODE, gpios);
static const struct gpio_dt_spec led3 = GPIO_DT_SPEC_GET(LED3_NODE, gpios);

int main(void)
{
	bool led_state = true;

	if (!gpio_is_ready_dt(&led0)||!gpio_is_ready_dt(&led1)||!gpio_is_ready_dt(&led2)||!gpio_is_ready_dt(&led3)) {
		return 0;
	}

	gpio_pin_configure_dt(&led0, GPIO_OUTPUT_ACTIVE);
    gpio_pin_configure_dt(&led1, GPIO_OUTPUT_ACTIVE);
    gpio_pin_configure_dt(&led2, GPIO_OUTPUT_ACTIVE);
    gpio_pin_configure_dt(&led3, GPIO_OUTPUT_ACTIVE);

	while (1) {
		gpio_pin_toggle_dt(&led0);
        gpio_pin_toggle_dt(&led1);
        gpio_pin_toggle_dt(&led2); 
        gpio_pin_toggle_dt(&led3);

		led_state = !led_state;
		printk("LED state: %s\n", led_state ? "ON" : "OFF");
		k_msleep(SLEEP_TIME_MS);
	}
	return 0;
}
```

但是上面的函数并不知道.详细的操作，只是依样画葫芦

## 3.2 GPIO

然后这边是我问AI的回答

```c
#include <zephyr/drivers/gpio.h>
#include <zephyr/device.h>

// 假设gpio_dev是你的GPIO设备指针，pin_number是你想要控制的引脚号
const struct device *gpio_dev = DEVICE_DT_GET(DT_NODELABEL(gpio0)); // 根据你的设备树配置获取设备指针
gpio_pin_t pin_number = 13; // 引脚号

// 配置引脚为输出
int ret = gpio_pin_configure(gpio_dev, pin_number, GPIO_OUTPUT);
if (ret < 0) {
    // 处理错误
}

// 设置引脚输出高电平（逻辑1）
ret = gpio_pin_set(gpio_dev, pin_number, 1);
if (ret < 0) {
    // 处理错误
}

// 设置引脚输出低电平（逻辑0）
ret = gpio_pin_set(gpio_dev, pin_number, 0);
if (ret < 0) {
    // 处理错误
}
```

对着AI的回复，能看懂基本的信息，然后我这边使用的是led1，是已经配置好相关的Dts,

所以直接看代码

```c
/*
 * Copyright (c) 2016 Intel Corporation
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <stdio.h>
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/logging/log.h>

/* The devicetree node identifier for the "led1" alias. */
#define led1_NODE DT_ALIAS(led1)

#define LOG_MODULE_NAME my_module
LOG_MODULE_REGISTER(LOG_MODULE_NAME);
/*
 * A build error on this line means your board is unsupported.
 * See the sample documentation for information on how to fix this.
 */
static const struct gpio_dt_spec led1 = GPIO_DT_SPEC_GET(led1_NODE, gpios);


int main(void)
{
    if (!gpio_is_ready_dt(&led1))
    {
        return 0;
    }
	//因为已经官方已经配置好dts这边使用的gpio_pin_configure_dt函数
    //gpio_pin_configure是要指定对应的gpio脚
    gpio_pin_configure_dt(&led1, GPIO_OUTPUT);


    while (1) {
        gpio_pin_set_dt(&led1, 1);
        k_msleep(1000);
        gpio_pin_set_dt(&led1, 0);
        k_msleep(2000);
    }
    return 0;
}

```

## 3.3 PWM

灯的基本操作看完了，这边直接看pwm相关的操作.

pwm就不赘述了.

官网的例子路径如下:

```
C:\ncs\v2.9.1\zephyr\samples\drivers\led\pwm
```

官方写的内容有好几个，我这边专注pwm的实现，所以删了不少的内容

下面是对`main.c`文件中每个地方的注释的详细解释：

### 包含头文件

```c
#include <zephyr/device.h>
#include <zephyr/devicetree.h>
#include <errno.h>
#include <zephyr/drivers/led.h>
#include <zephyr/sys/util.h>
#include <zephyr/kernel.h>

#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(main, CONFIG_LOG_DEFAULT_LEVEL);
```
- **`<zephyr/device.h>`**：包含设备驱动程序的通用接口。
- **`<zephyr/devicetree.h>`**：包含设备树相关的宏和函数。
- **`<errno.h>`**：包含错误号定义。
- **`<zephyr/drivers/led.h>`**：包含LED驱动程序的API。
- **`<zephyr/sys/util.h>`**：包含一些通用的系统工具宏。
- **`<zephyr/kernel.h>`**：包含内核相关的API。
- **`<zephyr/logging/log.h>`**：包含日志记录相关的API。
- **`LOG_MODULE_REGISTER(main, CONFIG_LOG_DEFAULT_LEVEL);`**：注册日志模块，设置日志级别为默认级别。

### 定义LED PWM节点

```c
#define LED_PWM_NODE_ID DT_COMPAT_GET_ANY_STATUS_OKAY(pwm_leds)
```
- **`LED_PWM_NODE_ID`**：使用设备树宏`DT_COMPAT_GET_ANY_STATUS_OKAY`获取兼容性为`pwm_leds`且状态为`okay`的节点ID。

### 定义LED标签数组

```c
const char *led_label[] = {
	DT_FOREACH_CHILD_SEP_VARGS(LED_PWM_NODE_ID, DT_PROP_OR, (,), label, NULL)
};

const int num_leds = ARRAY_SIZE(led_label);
```
- **`led_label`**：使用设备树宏`DT_FOREACH_CHILD_SEP_VARGS`遍历`LED_PWM_NODE_ID`的所有子节点，获取每个子节点的`label`属性，并将其存储在数组中。
- **`num_leds`**：计算`led_label`数组的大小，即LED的数量。

### 定义亮度和延迟

```c
#define MAX_BRIGHTNESS 100

#define FADE_DELAY_MS 10
#define FADE_DELAY K_MSEC(FADE_DELAY_MS)
```
- **`MAX_BRIGHTNESS`**：定义LED的最大亮度为100。
- **`FADE_DELAY_MS`**：定义亮度变化的延迟时间为10毫秒。
- **`FADE_DELAY`**：将`FADE_DELAY_MS`转换为内核时间单位（毫秒）。

### 运行LED测试函数

```c
/**
 * @brief Run tests on a single LED using the LED API syscalls.
 *
 * @param led_pwm LED PWM device.
 * @param led Number of the LED to test.
 */
static void run_led_test(const struct device *led_pwm, uint8_t led)
{
	int err;
	uint16_t level;
	/* Increase LED brightness gradually up to the maximum level. */
	LOG_INF("  Increasing brightness gradually");
	for (level = 0; level <= MAX_BRIGHTNESS; level++) {
		err = led_set_brightness(led_pwm, led, level);
		if (err < 0) {
			LOG_ERR("err=%d brightness=%d\n", err, level);
			return;
		}
		k_sleep(FADE_DELAY);
	}
	k_sleep(K_MSEC(1000));
}
```
- **函数说明**：`run_led_test`函数用于对单个LED进行测试。
- **参数**：
  - `led_pwm`：LED PWM设备指针。
  - `led`：要测试的LED编号。
- **功能**：
  - 使用`led_set_brightness`函数逐渐增加LED的亮度，直到达到最大亮度。
  - 每次设置亮度后，使用`k_sleep(FADE_DELAY)`延迟一段时间。
  - 达到最大亮度后，再延迟1000毫秒。

### 主函数

```c
int main(void)
{
	const struct device *led_pwm;
	uint8_t led;

	led_pwm = DEVICE_DT_GET(LED_PWM_NODE_ID);
	if (!device_is_ready(led_pwm)) {
		LOG_ERR("Device %s is not ready", led_pwm->name);
		return 0;
	}

	if (!num_leds) {
		LOG_ERR("No LEDs found for %s", led_pwm->name);
		return 0;
	}

	do {
        //我这边打印看了 只做了一个led支持pwm，需要自己看实际demo添加，实际demo我晚点研究
        LOG_ERR("num_leds %d",num_leds);
		for (led = 0; led < num_leds; led++) {
			run_led_test(led_pwm, led);
		}
	} while (true);
	return 0;
}
```
- **功能**：
  - 获取LED PWM设备指针`led_pwm`。
  - 检查设备是否准备好，如果未准备好则记录错误日志并退出。
  - 检查是否有LED，如果没有找到LED则记录错误日志并退出。
  - 使用`do-while`循环无限循环地对每个LED进行测试。
  - 调用`run_led_test`函数对每个LED进行亮度变化测试。

### 总结

这个文件的主要功能是对通过PWM控制的LED进行亮度变化测试。通过设备树获取LED设备和标签信息，并使用LED API设置LED的亮度，实现亮度的逐渐变化效果。

# 4 Button

led作为输出的入门,输入的入门当然是按钮。

官方示例代码路径:

```
C:\ncs\v2.9.1\zephyr\samples\basic\button
```

按键官方默认映射了四个脚别名，可以直接使用,相关路径如下:

```
C:\ncs\v2.9.1\zephyr\boards\nordic\nrf52dk\nrf52dk_nrf52832-pinctrl.dtsi
```

```dts
	aliases {
		led0 = &led0;
		led1 = &led1;
		led2 = &led2;
		led3 = &led3;
		pwm-led0 = &pwm_led0;
		sw0 = &button0;
		sw1 = &button1;
		sw2 = &button2;
		sw3 = &button3;
		bootloader-led0 = &led0;
		mcuboot-button0 = &button0;
		mcuboot-led0 = &led0;
		watchdog0 = &wdt0;
	};
```

看下buttons的引脚定义

```dts
	buttons {
		compatible = "gpio-keys";
		button0: button_0 {
			gpios = <&gpio0 13 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
			label = "Push button switch 0";
			zephyr,code = <INPUT_KEY_0>;
		};
		button1: button_1 {
			gpios = <&gpio0 14 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
			label = "Push button switch 1";
			zephyr,code = <INPUT_KEY_1>;
		};
		button2: button_2 {
			gpios = <&gpio0 15 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
			label = "Push button switch 2";
			zephyr,code = <INPUT_KEY_2>;
		};
		button3: button_3 {
			gpios = <&gpio0 16 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
			label = "Push button switch 3";
			zephyr,code = <INPUT_KEY_3>;
		};
	};
```



以下是 `main.c` 文件的流程解释，跳过了文件开头的版本信息部分：

---

### **1. 引入头文件**
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/sys/util.h>
#include <inttypes.h>
#include <zephyr/logging/log.h>
```
- 引入了 Zephyr RTOS 的核心头文件和 GPIO 驱动相关的头文件。
- 用于操作设备、GPIO 配置、日志记录等功能。

---

### **2. 定义常量和模块名称**
```c
#define SLEEP_TIME_MS	1
#define LOG_MODULE_NAME my_module
LOG_MODULE_REGISTER(LOG_MODULE_NAME);
```
- 定义了休眠时间（1 毫秒）。
- 注册了一个日志模块，名称为 `my_module`，用于调试和日志输出。

---

### **3. 获取按钮配置**
```c
#define SW0_NODE	DT_ALIAS(sw0)
#if !DT_NODE_HAS_STATUS_OKAY(SW0_NODE)
#error "Unsupported board: sw0 devicetree alias is not defined"
#endif
static const struct gpio_dt_spec button = GPIO_DT_SPEC_GET_OR(SW0_NODE, gpios, {0});
static struct gpio_callback button_cb_data;
```
- 使用 Devicetree (`sw0` 别名) 获取按钮的 GPIO 配置。
- 如果 `sw0` 别名未定义，则编译报错。
- 将按钮的 GPIO 配置存储在 `button` 变量中，并初始化一个 GPIO 回调数据结构 `button_cb_data`。

---

### **4. 获取 LED 配置（可选）**
```c
static struct gpio_dt_spec led = GPIO_DT_SPEC_GET_OR(DT_ALIAS(led0), gpios, {0});
```
- 使用 Devicetree (`led0` 别名) 获取 LED 的 GPIO 配置。
- 如果 `led0` 别名未定义，则默认值为 `{0}`，表示不使用 LED。

---

### **5. 按钮按下回调函数**
```c
void button_pressed(const struct device *dev, struct gpio_callback *cb, uint32_t pins)
{
    LOG_INF("Button pressed at %" PRIu32 "\n", k_cycle_get_32());
}
```
- 当按钮被按下时，触发此回调函数。
- 记录按钮按下的时间戳（以 CPU 周期为单位）。

---

### **6. 主函数逻辑**
#### **6.1 初始化按钮**
```c
if (!gpio_is_ready_dt(&button)) {
    LOG_INF("Error: button device %s is not ready\n", button.port->name);
    return 0;
}

ret = gpio_pin_configure_dt(&button, GPIO_INPUT);
if (ret != 0) {
    LOG_INF("Error %d: failed to configure %s pin %d\n", ret, button.port->name, button.pin);
    return 0;
}
```
- 检查按钮的 GPIO 设备是否可用。
- 配置按钮引脚为输入模式。

#### **6.2 配置中断**
```c
ret = gpio_pin_interrupt_configure_dt(&button, GPIO_INT_EDGE_TO_ACTIVE);
if (ret != 0) {
    LOG_INF("Error %d: failed to configure interrupt on %s pin %d\n", ret, button.port->name, button.pin);
    return 0;
}
```
- 配置按钮引脚的中断，检测边沿触发事件（按钮按下）。

#### **6.3 添加回调函数**
```c
gpio_init_callback(&button_cb_data, button_pressed, BIT(button.pin));
gpio_add_callback(button.port, &button_cb_data);
LOG_INF("Set up button at %s pin %d\n", button.port->name, button.pin);
```
- 初始化并注册按钮的回调函数 `button_pressed`。
- 将回调函数绑定到按钮的 GPIO 引脚。

#### **6.4 初始化 LED（如果存在）**
```c
if (led.port && !gpio_is_ready_dt(&led)) {
    LOG_INF("Error %d: LED device %s is not ready; ignoring it\n", ret, led.port->name);
    led.port = NULL;
}
if (led.port) {
    ret = gpio_pin_configure_dt(&led, GPIO_OUTPUT);
    if (ret != 0) {
        LOG_INF("Error %d: failed to configure LED device %s pin %d\n", ret, led.port->name, led.pin);
        led.port = NULL;
    } else {
        LOG_INF("Set up LED at %s pin %d\n", led.port->name, led.pin);
    }
}
```
- 检查 LED 的 GPIO 设备是否可用。
- 如果可用，配置 LED 引脚为输出模式。

#### **6.5 主循环**
```c
LOG_INF("Press the button\n");
if (led.port) {
    while (1) {
        int val = gpio_pin_get_dt(&button);
        if (val >= 0) {
            gpio_pin_set_dt(&led, val);
        }
        k_msleep(SLEEP_TIME_MS);
    }
}
```
- 提示用户按下按钮。
- 如果存在 LED：
  - 在主循环中读取按钮的状态。
  - 根据按钮状态设置 LED 的亮灭。
  - 每次循环休眠 1 毫秒。

---

### **7. 返回值**
```c
return 0;
```
- 主函数正常结束返回 0。

---

### **总结**
该程序的主要功能是：
1. 监听按钮的按下事件，并通过中断触发回调函数记录事件。
2. 如果存在 LED，则将 LED 的状态与按钮的状态同步（按钮按下时点亮 LED，松开时熄灭 LED）。

##  修改练习

我这边按照我对代码的理解补充一个按键控制另外一个灯的操作,

完整的代码如下:

```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/sys/util.h>
#include <inttypes.h>
#include <zephyr/logging/log.h>

#define SLEEP_TIME_MS	1
#define LOG_MODULE_NAME my_module
LOG_MODULE_REGISTER(LOG_MODULE_NAME);
/*
 * Get button configuration from the devicetree sw0 alias. This is mandatory.
 */
#define SW0_NODE	DT_ALIAS(sw0)
#define SW1_NODE	DT_ALIAS(sw1)
#if !DT_NODE_HAS_STATUS_OKAY(SW0_NODE)
#error "Unsupported board: sw0 devicetree alias is not defined"
#endif
static const struct gpio_dt_spec button0 = GPIO_DT_SPEC_GET_OR(SW0_NODE, gpios,{0});
static const struct gpio_dt_spec button1 = GPIO_DT_SPEC_GET_OR(SW1_NODE, gpios,{0});
static struct gpio_callback button_cb_data0;
static struct gpio_callback button_cb_data1;
/*
 * The led0 devicetree alias is optional. If present, we'll use it
 * to turn on the LED whenever the button is pressed.
 */
static struct gpio_dt_spec led0 = GPIO_DT_SPEC_GET_OR(DT_ALIAS(led0), gpios,{0});
static struct gpio_dt_spec led1 = GPIO_DT_SPEC_GET_OR(DT_ALIAS(led1), gpios,{0});

void button_pressed0(const struct device *dev, struct gpio_callback *cb,
		    uint32_t pins)
{
	LOG_INF("Button pressed at %" PRIu32 "\n", k_cycle_get_32());
}

void button_pressed1(const struct device *dev, struct gpio_callback *cb,
    uint32_t pins)
{
LOG_INF("Button pressed at %" PRIu32 "\n", k_cycle_get_32());
}

void init()
{
	if (!gpio_is_ready_dt(&button0)) {

	}

	 gpio_pin_configure_dt(&button0, GPIO_INPUT);

	 gpio_pin_interrupt_configure_dt(&button0,GPIO_INT_EDGE_TO_ACTIVE);

    if (!gpio_is_ready_dt(&button1)) 
    {

	}

	gpio_pin_configure_dt(&button1, GPIO_INPUT);


	gpio_pin_interrupt_configure_dt(&button1, GPIO_INT_EDGE_TO_ACTIVE);

}
int main(void)
{
    init();

	gpio_init_callback(&button_cb_data0, button_pressed0, BIT(button0.pin));
	gpio_add_callback(button0.port, &button_cb_data0);
	LOG_INF("Set up button at %s pin %d\n", button0.port->name, button0.pin);

    gpio_init_callback(&button_cb_data1, button_pressed1, BIT(button1.pin));
	gpio_add_callback(button1.port, &button_cb_data1);
	LOG_INF("Set up button at %s pin %d\n", button1.port->name, button1.pin);

	if (led0.port && !gpio_is_ready_dt(&led0)) {
		led0.port = NULL;
	}
	if (led0.port) 
    {
		gpio_pin_configure_dt(&led0, GPIO_OUTPUT);
		LOG_INF("Set up LED at %s pin %d\n", led0.port->name, led0.pin);
	}

    if (led1.port && !gpio_is_ready_dt(&led1)) {
		led1.port = NULL;
	}
	if (led1.port) 
    {
		gpio_pin_configure_dt(&led1, GPIO_OUTPUT);
		LOG_INF("Set up LED at %s pin %d\n", led1.port->name, led1.pin);
	}

	LOG_INF("Press the button\n");
	
    while (1) 
    {
        /* If we have an LED, match its state to the button's. */
        int val  = gpio_pin_get_dt(&button0);
        if (val >= 0) 
        {
            gpio_pin_set_dt(&led0, val);
        }

        /* If we have an LED, match its state to the button's. */
        int val1 = gpio_pin_get_dt(&button1);

        if(val1 >= 0)
        {
            gpio_pin_set_dt(&led1, val1);
        }
        k_msleep(SLEEP_TIME_MS);
    }

    return 0;
}

```

# 5 Thread

在做完led(输出)和button(输入)后，我们也看下线程相关的内容,实际项目操作中，非常依赖各个线程之间的操作.咱都用上实时操作系统了,避免不了线程的.

把基础过完后再去看蓝牙相关的内容.

官方的线程非常简单,没什么难度.

当然，以下是详细的代码梳理，并附带详细的说明：

### 文件流程梳理

### 1. 头文件和宏定义
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/sys/printk.h>
#include <zephyr/sys/__assert.h>
#include <string.h>

/* size of stack area used by each thread */
#define STACKSIZE 1024

/* scheduling priority used by each thread */
#define PRIORITY 7
```
- **头文件**：包含了Zephyr RTOS所需的库文件，用于设备驱动、内核功能、GPIO操作、打印输出等。
- **宏定义**：定义了每个线程的堆栈大小为1024字节，优先级为7。

### 2. 设备树检查
```c
#define LED0_NODE DT_ALIAS(led0)
#define LED1_NODE DT_ALIAS(led1)

#if !DT_NODE_HAS_STATUS_OKAY(LED0_NODE)
#error "Unsupported board: led0 devicetree alias is not defined"
#endif

#if !DT_NODE_HAS_STATUS_OKAY(LED1_NODE)
#error "Unsupported board: led1 devicetree alias is not defined"
#endif
```
- **设备树别名**：通过设备树别名获取`led0`和`led1`节点。
- **检查设备树节点**：确保设备树中定义了`led0`和`led1`节点，否则编译报错。

### 3. 数据结构定义
```c
struct printk_data_t {
	void *fifo_reserved; /* 1st word reserved for use by fifo */
	uint32_t led;
	uint32_t cnt;
};

K_FIFO_DEFINE(printk_fifo);
```
- **结构体定义**：`printk_data_t`用于存储LED编号和计数器信息。
- **FIFO队列**：定义了一个FIFO队列`printk_fifo`，用于线程间通信。

### 4. LED初始化
```c
struct led {
	struct gpio_dt_spec spec;
	uint8_t num;
};

static const struct led led0 = {
	.spec = GPIO_DT_SPEC_GET_OR(LED0_NODE, gpios, {0}),
	.num = 0,
};

static const struct led led1 = {
	.spec = GPIO_DT_SPEC_GET_OR(LED1_NODE, gpios, {0}),
	.num = 1,
};
```
- **结构体定义**：`led`结构体用于封装LED的GPIO配置信息。
- **LED初始化**：`led0`和`led1`是两个常量结构体，分别初始化了两个LED的GPIO配置。

### 5. LED闪烁函数 (`blink`)
```c
void blink(const struct led *led, uint32_t sleep_ms, uint32_t id)
{
	const struct gpio_dt_spec *spec = &led->spec;
	int cnt = 0;
	int ret;

	if (!device_is_ready(spec->port)) {
		printk("Error: %s device is not ready\n", spec->port->name);
		return;
	}

	ret = gpio_pin_configure_dt(spec, GPIO_OUTPUT);
	if (ret != 0) {
		printk("Error %d: failed to configure pin %d (LED '%d')\n",
			ret, spec->pin, led->num);
		return;
	}

	while (1) {
		gpio_pin_set(spec->port, spec->pin, cnt % 2);

		struct printk_data_t tx_data = { .led = id, .cnt = cnt };

		size_t size = sizeof(struct printk_data_t);
		char *mem_ptr = k_malloc(size);
		__ASSERT_NO_MSG(mem_ptr != 0);

		memcpy(mem_ptr, &tx_data, size);

		k_fifo_put(&printk_fifo, mem_ptr);

		k_msleep(sleep_ms);
		cnt++;
	}
}
```
- **函数功能**：`blink`函数接受一个LED结构体指针、延时时间和LED标识作为参数。
- **设备检查**：检查对应的GPIO设备是否准备好。
- **GPIO配置**：将GPIO配置为输出模式。
- **无限循环**：在无限循环中切换LED状态，并将当前状态（LED编号和计数器）放入FIFO队列中。
- **延时**：每次切换后休眠指定的时间。

### 6. 线程函数定义
```c
void blink0(void)
{
	blink(&led0, 100, 0);
}

void blink1(void)
{
	blink(&led1, 1000, 1);
}

void uart_out(void)
{
	while (1) {
		struct printk_data_t *rx_data = k_fifo_get(&printk_fifo,
							   K_FOREVER);
		printk("Toggled led%d; counter=%d\n",
		       rx_data->led, rx_data->cnt);
		k_free(rx_data);
	}
}
```
- **`blink0`函数**：调用`blink`函数控制`led0`，每100ms切换一次。
- **`blink1`函数**：调用`blink`函数控制`led1`，每1000ms切换一次。
- **`uart_out`函数**：从FIFO队列中获取数据并打印到串口。

### 7. 线程创建
```c
K_THREAD_DEFINE(blink0_id, STACKSIZE, blink0, NULL, NULL, NULL,
		PRIORITY, 0, 0);
K_THREAD_DEFINE(blink1_id, STACKSIZE, blink1, NULL, NULL, NULL,
		PRIORITY, 0, 0);
K_THREAD_DEFINE(uart_out_id, STACKSIZE, uart_out, NULL, NULL, NULL,
		PRIORITY, 0, 0);
```
- **线程定义**：使用`K_THREAD_DEFINE`宏定义了三个线程：
  - `blink0_id`：控制`led0`，每100ms切换一次。
  - `blink1_id`：控制`led1`，每1000ms切换一次。
  - `uart_out_id`：负责从FIFO队列中读取数据并通过串口打印。

## 总结
该文件实现了一个简单的多线程应用程序，使用Zephyr RTOS管理多个线程来控制两个LED的闪烁，并通过FIFO队列和串口输出LED的状态信息。每个线程都有明确的功能，确保系统的模块化和可维护性。







