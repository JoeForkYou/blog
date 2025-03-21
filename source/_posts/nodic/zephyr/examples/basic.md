---
title: Zephyr 基础示例
tags: 
  - zephyr
  - examples
  - embedded
categories: 
  - nodic
  - zephyr
  - examples
abbrlink: 25
date: 2023-12-21 10:00:00
---
# Zephyr 基础示例

本文档提供了 Zephyr RTOS 的基础示例代码，包括 Hello World、LED 控制、按键输入、定时器使用和多线程编程等内容。这些示例适合初学者快速上手 Zephyr 开发。

## Hello World

这是最基本的 Zephyr 应用程序，展示了如何创建一个简单的 Zephyr 项目并输出 "Hello World" 消息。

### 源代码

**src/main.c**:
```c
/*
 * Copyright (c) 2023 Your Name
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>

void main(void)
{
    printk("Hello World! %s\n", CONFIG_BOARD);
}
```

**prj.conf**:
```
# 空配置文件，Hello World 不需要特殊配置
```

### 构建和运行

```bash
west build -b qemu_x86 samples/hello_world
west run
```

### 输出结果

```
Hello World! qemu_x86
```

### 代码说明

1. `#include <zephyr/kernel.h>` - 包含 Zephyr 内核头文件
2. `#include <zephyr/sys/printk.h>` - 包含打印函数头文件
3. `printk()` - 输出字符串到控制台
4. `CONFIG_BOARD` - 预定义宏，表示当前的开发板名称

## LED 控制

这个示例展示了如何控制开发板上的 LED，实现 LED 闪烁功能。

### 源代码

**src/main.c**:
```c
/*
 * Copyright (c) 2023 Your Name
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/* LED 设备树别名 */
#define LED0_NODE DT_ALIAS(led0)

#if DT_NODE_HAS_STATUS(LED0_NODE, okay)
#define LED0 DT_GPIO_CTLR(LED0_NODE, gpios)
#define LED0_PIN DT_GPIO_PIN(LED0_NODE, gpios)
#define LED0_FLAGS DT_GPIO_FLAGS(LED0_NODE, gpios)
#else
/* 如果设备树中没有定义 LED，使用默认值 */
#define LED0 DT_NODELABEL(gpio0)
#define LED0_PIN 13
#define LED0_FLAGS 0
#endif

void main(void)
{
    const struct device *dev;
    bool led_state = false;
    int ret;

    /* 获取 GPIO 设备 */
    dev = DEVICE_DT_GET(LED0);
    if (!device_is_ready(dev)) {
        printk("Error: %s device not ready\n", dev->name);
        return;
    }

    /* 配置 LED 引脚为输出 */
    ret = gpio_pin_configure(dev, LED0_PIN, GPIO_OUTPUT | LED0_FLAGS);
    if (ret < 0) {
        printk("Error %d: failed to configure pin %d\n", ret, LED0_PIN);
        return;
    }

    printk("LED control example started\n");

    /* 循环闪烁 LED */
    while (1) {
        /* 切换 LED 状态 */
        led_state = !led_state;
        gpio_pin_set(dev, LED0_PIN, led_state);
        
        /* 延时 1 秒 */
        k_sleep(K_MSEC(1000));
        
        printk("LED state: %s\n", led_state ? "ON" : "OFF");
    }
}
```

**prj.conf**:
```
CONFIG_GPIO=y
```

### 构建和运行

```bash
west build -b <board> samples/basic/blinky
west flash
```

### 代码说明

1. `#include <zephyr/drivers/gpio.h>` - 包含 GPIO 驱动头文件
2. `DT_ALIAS(led0)` - 获取设备树中 LED 的别名
3. `DEVICE_DT_GET()` - 获取设备实例
4. `device_is_ready()` - 检查设备是否准备就绪
5. `gpio_pin_configure()` - 配置 GPIO 引脚
6. `gpio_pin_set()` - 设置 GPIO 引脚状态
7. `k_sleep()` - 延时函数

## 按键输入

这个示例展示了如何读取按键输入并响应按键事件。

### 源代码

**src/main.c**:
```c
/*
 * Copyright (c) 2023 Your Name
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/sys/printk.h>

/* 按键设备树别名 */
#define BUTTON_NODE DT_ALIAS(sw0)

#if DT_NODE_HAS_STATUS(BUTTON_NODE, okay)
#define BUTTON_GPIO_LABEL DT_GPIO_CTLR(BUTTON_NODE, gpios)
#define BUTTON_PIN DT_GPIO_PIN(BUTTON_NODE, gpios)
#define BUTTON_FLAGS DT_GPIO_FLAGS(BUTTON_NODE, gpios)
#else
/* 如果设备树中没有定义按键，使用默认值 */
#define BUTTON_GPIO_LABEL DT_NODELABEL(gpio0)
#define BUTTON_PIN 11
#define BUTTON_FLAGS (GPIO_INPUT | GPIO_PULL_UP)
#endif

/* LED 设备树别名 */
#define LED_NODE DT_ALIAS(led0)

#if DT_NODE_HAS_STATUS(LED_NODE, okay)
#define LED_GPIO_LABEL DT_GPIO_CTLR(LED_NODE, gpios)
#define LED_PIN DT_GPIO_PIN(LED_NODE, gpios)
#define LED_FLAGS DT_GPIO_FLAGS(LED_NODE, gpios)
#else
/* 如果设备树中没有定义 LED，使用默认值 */
#define LED_GPIO_LABEL DT_NODELABEL(gpio0)
#define LED_PIN 13
#define LED_FLAGS 0
#endif

/* 按键中断回调函数 */
static struct gpio_callback button_cb_data;

void button_pressed(const struct device *dev, struct gpio_callback *cb,
                    uint32_t pins)
{
    printk("Button pressed at %" PRIu32 "\n", k_cycle_get_32());
}

void main(void)
{
    const struct device *button_dev, *led_dev;
    int ret;

    /* 获取 GPIO 设备 */
    button_dev = DEVICE_DT_GET(BUTTON_GPIO_LABEL);
    if (!device_is_ready(button_dev)) {
        printk("Error: button device %s is not ready\n", button_dev->name);
        return;
    }

    led_dev = DEVICE_DT_GET(LED_GPIO_LABEL);
    if (!device_is_ready(led_dev)) {
        printk("Error: LED device %s is not ready\n", led_dev->name);
        return;
    }

    /* 配置按键引脚 */
    ret = gpio_pin_configure(button_dev, BUTTON_PIN, GPIO_INPUT | BUTTON_FLAGS);
    if (ret != 0) {
        printk("Error %d: failed to configure button pin %d\n",
               ret, BUTTON_PIN);
        return;
    }

    /* 配置 LED 引脚 */
    ret = gpio_pin_configure(led_dev, LED_PIN, GPIO_OUTPUT | LED_FLAGS);
    if (ret != 0) {
        printk("Error %d: failed to configure LED pin %d\n",
               ret, LED_PIN);
        return;
    }

    /* 配置按键中断 */
    ret = gpio_pin_interrupt_configure(button_dev, BUTTON_PIN,
                                      GPIO_INT_EDGE_TO_ACTIVE);
    if (ret != 0) {
        printk("Error %d: failed to configure button interrupt\n", ret);
        return;
    }

    /* 设置中断回调 */
    gpio_init_callback(&button_cb_data, button_pressed, BIT(BUTTON_PIN));
    gpio_add_callback(button_dev, &button_cb_data);

    printk("Button example started\n");

    /* 主循环 */
    while (1) {
        /* 读取按键状态 */
        int val = gpio_pin_get(button_dev, BUTTON_PIN);
        
        /* 设置 LED 状态与按键状态相反（按下时点亮） */
        gpio_pin_set(led_dev, LED_PIN, val == 0);
        
        /* 延时 100 毫秒 */
        k_sleep(K_MSEC(100));
    }
}
```

**prj.conf**:
```
CONFIG_GPIO=y
```

### 构建和运行

```bash
west build -b <board> samples/basic/button
west flash
```

### 代码说明

1. `gpio_pin_interrupt_configure()` - 配置 GPIO 中断
2. `gpio_init_callback()` - 初始化中断回调
3. `gpio_add_callback()` - 添加中断回调
4. `gpio_pin_get()` - 获取 GPIO 引脚状态
5. `k_cycle_get_32()` - 获取系统时钟周期计数

## 定时器使用

这个示例展示了如何使用 Zephyr 的定时器功能。

### 源代码

**src/main.c**:
```c
/*
 * Copyright (c) 2023 Your Name
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>

/* 定时器回调函数 */
void timer_handler(struct k_timer *timer)
{
    printk("Timer expired at %d ms\n", k_uptime_get_32());
}

/* 定时器停止回调函数 */
void timer_stop_handler(struct k_timer *timer)
{
    printk("Timer stopped\n");
}

/* 定义定时器 */
K_TIMER_DEFINE(my_timer, timer_handler, timer_stop_handler);

void main(void)
{
    printk("Timer example started\n");

    /* 启动定时器：2秒后到期，之后每1秒触发一次 */
    k_timer_start(&my_timer, K_SECONDS(2), K_SECONDS(1));

    /* 等待 5 秒 */
    k_sleep(K_SECONDS(5));

    /* 获取剩余时间 */
    int32_t remaining = k_timer_remaining_get(&my_timer);
    printk("Remaining time: %d ms\n", remaining);

    /* 获取定时器状态 */
    uint32_t status = k_timer_status_get(&my_timer);
    printk("Timer expired %u times\n", status);

    /* 等待 2 秒 */
    k_sleep(K_SECONDS(2));

    /* 停止定时器 */
    k_timer_stop(&my_timer);

    /* 使用单次定时器 */
    printk("Starting one-shot timer\n");
    k_timer_start(&my_timer, K_SECONDS(1), K_NO_WAIT);

    /* 等待定时器到期 */
    k_timer_status_sync(&my_timer);
    printk("One-shot timer expired\n");

    printk("Timer example completed\n");
}
```

**prj.conf**:
```
# 不需要特殊配置
```

### 构建和运行

```bash
west build -b <board> samples/basic/timer
west flash
```

### 代码说明

1. `K_TIMER_DEFINE()` - 定义定时器
2. `k_timer_start()` - 启动定时器
3. `k_timer_remaining_get()` - 获取剩余时间
4. `k_timer_status_get()` - 获取定时器状态
5. `k_timer_stop()` - 停止定时器
6. `k_timer_status_sync()` - 同步等待定时器到期
7. `k_uptime_get_32()` - 获取系统启动后的毫秒数

## 多线程编程

这个示例展示了如何在 Zephyr 中创建和使用多个线程。

### 源代码

**src/main.c**:
```c
/*
 * Copyright (c) 2023 Your Name
 *
 * SPDX-License-Identifier: Apache-2.0
 */

#include <zephyr/kernel.h>
#include <zephyr/sys/printk.h>

/* 定义线程栈大小 */
#define STACK_SIZE 1024

/* 定义线程优先级 */
#define THREAD_A_PRIORITY 5
#define THREAD_B_PRIORITY 5

/* 定义线程栈和数据 */
K_THREAD_STACK_DEFINE(thread_a_stack, STACK_SIZE);
K_THREAD_STACK_DEFINE(thread_b_stack, STACK_SIZE);

struct k_thread thread_a_data;
struct k_thread thread_b_data;

/* 定义信号量用于同步 */
K_SEM_DEFINE(sem_a, 0, 1);
K_SEM_DEFINE(sem_b, 0, 1);

/* 线程 A 入口函数 */
void thread_a_entry(void *p1, void *p2, void *p3)
{
    int count = 0;

    while (1) {
        /* 等待信号量 */
        k_sem_take(&sem_a, K_FOREVER);

        /* 执行任务 */
        printk("Thread A running, count: %d\n", count++);

        /* 延时 1 秒 */
        k_sleep(K_SECONDS(1));

        /* 释放线程 B 的信号量 */
        k_sem_give(&sem_b);
    }
}

/* 线程 B 入口函数 */
void thread_b_entry(void *p1, void *p2, void *p3)
{
    int count = 0;

    while (1) {
        /* 等待信号量 */
        k_sem_take(&sem_b, K_FOREVER);

        /* 执行任务 */
        printk("Thread B running, count: %d\n", count++);

        /* 延时 1 秒 */
        k_sleep(K_SECONDS(1));

        /* 释放线程 A 的信号量 */
        k_sem_give(&sem_a);
    }
}

void main(void)
{
    printk("Multi-threading example started\n");

    /* 创建线程 A */
    k_thread_create(&thread_a_data, thread_a_stack,
                    K_THREAD_STACK_SIZEOF(thread_a_stack),
                    thread_a_entry, NULL, NULL, NULL,
                    THREAD_A_PRIORITY, 0, K_NO_WAIT);

    /* 创建线程 B */
    k_thread_create(&thread_b_data, thread_b_stack,
                    K_THREAD_STACK_SIZEOF(thread_b_stack),
                    thread_b_entry, NULL, NULL, NULL,
                    THREAD_B_PRIORITY, 0, K_NO_WAIT);

    /* 设置线程名称（如果启用） */
#ifdef CONFIG_THREAD_NAME
    k_thread_name_set(&thread_a_data, "thread_a");
    k_thread_name_set(&thread_b_data, "thread_b");
#endif

    /* 启动线程 A */
    k_sem_give(&sem_a);

    /* 主线程工作 */
    while (1) {
        printk("Main thread running\n");
        k_sleep(K_SECONDS(2));
    }
}
```

**prj.conf**:
```
CONFIG_THREAD_NAME=y
```

### 构建和运行

```bash
west build -b <board> samples/basic/threads
west flash
```

### 代码说明

1. `K_THREAD_STACK_DEFINE()` - 定义线程栈
2. `K_SEM_DEFINE()` - 定义信号量
3. `k_thread_create()` - 创建线程
4. `k_thread_name_set()` - 设置线程名称
5. `k_sem_take()` - 获取信号量
6. `k_sem_give()` - 释放信号量

## 总结

这些基础示例展示了 Zephyr RTOS 的核心功能，包括基本输出、GPIO 控制、中断处理、定时器使用和多线程编程。通过学习和实践这些示例，您可以快速掌握 Zephyr 开发的基础知识，为更复杂的应用开发打下基础。

每个示例都可以独立构建和运行，并且可以根据需要进行修改和扩展。建议初学者按照顺序学习这些示例，从简单到复杂，逐步掌握 Zephyr RTOS 的各项功能。