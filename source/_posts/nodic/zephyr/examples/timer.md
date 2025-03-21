---
abbrlink: 30
title: timer
date: '2025-03-21 20:49:56'
---
# Zephyr 定时器示例

## 1. 定时器配置

### 1.1 基础配置 (prj.conf)
```plaintext
# 基础配置
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# 定时器配置
CONFIG_TIMER=y
CONFIG_TIMER_HAS_64BIT_CYCLE_COUNTER=y

# 系统时钟配置
CONFIG_SYS_CLOCK_TICKS_PER_SEC=1000

# 定时器测试配置（可选）
CONFIG_TIMER_RANDOM_GENERATOR=y
```

## 2. 基本定时器示例

### 2.1 k_timer 示例
```c
#include <zephyr/kernel.h>

/* 定义定时器 */
struct k_timer my_timer;

/* 定时器回调函数 */
void timer_expiry_function(struct k_timer *timer_id)
{
    printk("Timer expired!\n");
}

/* 定时器停止回调函数 */
void timer_stop_function(struct k_timer *timer_id)
{
    printk("Timer stopped!\n");
}

int main(void)
{
    /* 初始化定时器 */
    k_timer_init(&my_timer, timer_expiry_function, timer_stop_function);

    /* 启动定时器：2秒后触发，之后每秒触发一次 */
    k_timer_start(&my_timer, K_SECONDS(2), K_SECONDS(1));

    /* 等待一段时间 */
    k_msleep(5000);

    /* 停止定时器 */
    k_timer_stop(&my_timer);

    return 0;
}
```

## 3. 高精度定时器

### 3.1 高精度延时
```c
#include <zephyr/kernel.h>
#include <zephyr/timing/timing.h>

void high_precision_delay(void)
{
    uint32_t start_time, end_time;
    uint32_t cycles, ns;

    /* 初始化timing系统 */
    timing_init();
    timing_start();

    /* 记录开始时间 */
    start_time = timing_counter_get();

    /* 执行需要计时的操作 */
    k_busy_wait(100);  // 延时100微秒

    /* 记录结束时间 */
    end_time = timing_counter_get();

    /* 计算经过的时间 */
    cycles = timing_cycles_get(&start_time, &end_time);
    ns = timing_cycles_to_ns(cycles);

    printk("Operation took %u cycles (%u ns)\n", cycles, ns);
}
```

## 4. 周期性任务

### 4.1 工作队列定时器
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>

/* 定义工作队列 */
static struct k_work_q my_work_q;
static K_THREAD_STACK_DEFINE(my_work_q_stack, 1024);

/* 定义延迟工作 */
static struct k_work_delayable delayed_work;

/* 工作处理函数 */
static void work_handler(struct k_work *work)
{
    printk("Periodic work executed\n");

    /* 重新调度下一次执行 */
    k_work_schedule_for_queue(&my_work_q, &delayed_work, K_SECONDS(1));
}

int main(void)
{
    /* 初始化工作队列 */
    k_work_queue_init(&my_work_q);
    k_work_queue_start(&my_work_q, my_work_q_stack,
                      K_THREAD_STACK_SIZEOF(my_work_q_stack),
                      K_PRIO_PREEMPT(10), NULL);

    /* 初始化延迟工作 */
    k_work_init_delayable(&delayed_work, work_handler);

    /* 开始周期性任务 */
    k_work_schedule_for_queue(&my_work_q, &delayed_work, K_NO_WAIT);

    return 0;
}
```

## 5. 看门狗定时器

### 5.1 watchdog配置 (prj.conf)
```plaintext
CONFIG_WATCHDOG=y
```

### 5.2 watchdog示例
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/watchdog.h>

/* watchdog回调函数 */
static void wdt_callback(const struct device *wdt_dev, int channel_id)
{
    printk("Watchdog callback triggered!\n");
}

int main(void)
{
    const struct device *const wdt = DEVICE_DT_GET(DT_NODELABEL(wdt0));
    int ret;

    if (!device_is_ready(wdt)) {
        printk("Watchdog device not ready\n");
        return -ENODEV;
    }

    /* 配置watchdog */
    struct wdt_timeout_cfg wdt_config = {
        .window.min = 0,
        .window.max = 1000,  /* 1秒超时 */
        .callback = wdt_callback,
        .flags = WDT_FLAG_RESET_SOC
    };

    ret = wdt_install_timeout(wdt, &wdt_config);
    if (ret != 0) {
        printk("Watchdog install error\n");
        return ret;
    }

    /* 启动watchdog */
    ret = wdt_setup(wdt, 0);
    if (ret != 0) {
        printk("Watchdog setup error\n");
        return ret;
    }

    while (1) {
        /* 定期喂狗 */
        wdt_feed(wdt, 0);
        k_msleep(500);  /* 每500ms喂一次狗 */
    }

    return 0;
}
```

## 6. 实时时钟(RTC)

### 6.1 RTC配置 (prj.conf)
```plaintext
CONFIG_RTC=y
CONFIG_COUNTER=y
```

### 6.2 RTC示例
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/counter.h>
#include <time.h>

/* RTC报警回调 */
static void rtc_alarm_callback(const struct device *dev,
                             uint8_t chan_id,
                             uint32_t ticks,
                             void *user_data)
{
    printk("RTC Alarm triggered!\n");
}

int main(void)
{
    const struct device *const rtc = DEVICE_DT_GET(DT_NODELABEL(rtc0));
    uint32_t now;
    int ret;

    if (!device_is_ready(rtc)) {
        printk("RTC device not ready\n");
        return -ENODEV;
    }

    /* 获取当前计数值 */
    ret = counter_get_value(rtc, &now);
    if (ret != 0) {
        printk("Failed to get RTC value\n");
        return ret;
    }

    /* 配置报警 */
    struct counter_alarm_cfg alarm_cfg = {
        .callback = rtc_alarm_callback,
        .flags = 0,
        .ticks = now + counter_us_to_ticks(rtc, 5000000)  /* 5秒后报警 */
    };

    ret = counter_set_channel_alarm(rtc, 0, &alarm_cfg);
    if (ret != 0) {
        printk("Failed to set alarm\n");
        return ret;
    }

    while (1) {
        ret = counter_get_value(rtc, &now);
        if (ret == 0) {
            time_t timestamp = counter_ticks_to_us(rtc, now) / 1000000;
            struct tm *tm_info = localtime(&timestamp);
            printk("Current time: %02d:%02d:%02d\n",
                   tm_info->tm_hour,
                   tm_info->tm_min,
                   tm_info->tm_sec);
        }
        k_msleep(1000);
    }

    return 0;
}
```

## 7. 定时器最佳实践

### 7.1 选择合适的定时器
1. k_timer：用于一般的定时任务
2. k_work_delayable：用于需要在工作队列上下文中执行的定时任务
3. 硬件定时器：用于高精度定时要求
4. watchdog：用于系统监控和故障恢复
5. RTC：用于实时时钟和日期功能

### 7.2 定时器使用建议
1. 避免在定时器回调函数中执行耗时操作
2. 合理设置定时器周期，避免过于频繁的触发
3. 注意检查定时器的返回值
4. 在不需要时及时停止定时器
5. 使用适当的优先级配置工作队列

### 7.3 错误处理示例
```c
#include <zephyr/kernel.h>

void timer_error_handling(void)
{
    struct k_timer timer;
    int ret;

    /* 初始化定时器 */
    k_timer_init(&timer, NULL, NULL);

    /* 启动定时器 */
    k_timer_start(&timer, K_SECONDS(1), K_SECONDS(1));

    /* 等待定时器触发 */
    ret = k_timer_status_sync(&timer);
    if (ret < 0) {
        printk("Timer sync error: %d\n", ret);
        return;
    }

    /* 检查定时器状态 */
    if (k_timer_remaining_get(&timer) > 0) {
        printk("Timer is still running\n");
    }

    /* 停止定时器 */
    k_timer_stop(&timer);
}
```