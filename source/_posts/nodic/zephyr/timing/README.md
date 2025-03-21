---
abbrlink: 40
title: Zephyr 定时系统指南
date: '2025-03-21 20:49:56'
---
# Zephyr 时间管理指南

## 1. 时间管理配置

### 1.1 基础配置 (prj.conf)
```plaintext
# 系统时钟配置
CONFIG_SYS_CLOCK_TICKS_PER_SEC=1000
CONFIG_TICKLESS_KERNEL=y

# 定时器配置
CONFIG_TIMER_HAS_64BIT_CYCLE_COUNTER=y
CONFIG_TIMER_READS_ITS_FREQUENCY_AT_RUNTIME=y

# RTC配置（如果使用）
CONFIG_COUNTER=y
CONFIG_COUNTER_RTC0=y

# 时间测量配置
CONFIG_TIMING_FUNCTIONS=y
```

### 1.2 时钟源配置 (boards/xxx.overlay)
```dts
/ {
    soc {
        timer0: timer@40000000 {
            compatible = "nordic,nrf-timer";
            reg = <0x40000000 0x1000>;
            interrupts = <8 1>;
            status = "okay";
        };
    };
};
```

## 2. 基本时间操作

### 2.1 延时函数
```c
#include <zephyr/kernel.h>

void delay_example(void)
{
    /* 毫秒延时 */
    k_msleep(1000);    // 延时1秒

    /* 微秒延时 */
    k_usleep(1000);    // 延时1毫秒

    /* 纳秒延时 */
    k_nsleep(1000);    // 延时1微秒

    /* 忙等待延时 */
    k_busy_wait(1000); // 延时1微秒（不释放CPU）

    /* 自定义时间单位延时 */
    k_sleep(K_SECONDS(1));     // 1秒
    k_sleep(K_MSEC(500));      // 500毫秒
    k_sleep(K_MINUTES(1));     // 1分钟
    k_sleep(K_HOURS(1));       // 1小时
}
```

### 2.2 时间获取
```c
#include <zephyr/kernel.h>

void time_get_example(void)
{
    /* 获取系统启动后的时间 */
    int64_t uptime = k_uptime_get();
    printk("System uptime: %lld ms\n", uptime);

    /* 获取32位系统时间 */
    uint32_t uptime32 = k_uptime_get_32();
    printk("System uptime (32-bit): %u ms\n", uptime32);

    /* 获取CPU cycles */
    uint32_t cycles = k_cycle_get_32();
    printk("CPU cycles: %u\n", cycles);

    /* 获取64位CPU cycles */
    uint64_t cycles64 = k_cycle_get_64();
    printk("CPU cycles (64-bit): %llu\n", cycles64);
}
```

## 3. 定时器管理

### 3.1 基本定时器
```c
#include <zephyr/kernel.h>

/* 定义定时器 */
struct k_timer my_timer;

/* 定时器回调函数 */
void timer_handler(struct k_timer *timer)
{
    printk("Timer expired!\n");
}

/* 定时器停止回调函数 */
void timer_stop_handler(struct k_timer *timer)
{
    printk("Timer stopped!\n");
}

void timer_example(void)
{
    /* 初始化定时器 */
    k_timer_init(&my_timer, timer_handler, timer_stop_handler);

    /* 启动定时器（2秒后触发，每1秒重复） */
    k_timer_start(&my_timer, K_SECONDS(2), K_SECONDS(1));

    /* 等待定时器触发 */
    k_timer_status_sync(&my_timer);

    /* 获取剩余时间 */
    uint32_t remaining = k_timer_remaining_get(&my_timer);
    printk("Remaining time: %u ms\n", remaining);

    /* 停止定时器 */
    k_timer_stop(&my_timer);
}
```

### 3.2 高精度定时器
```c
#include <zephyr/kernel.h>
#include <zephyr/timing/timing.h>

void high_precision_timer(void)
{
    uint32_t start_time, end_time;
    uint32_t cycles, ns;

    /* 初始化timing系统 */
    timing_init();
    timing_start();

    /* 获取开始时间 */
    start_time = timing_counter_get();

    /* 执行需要测量的操作 */
    k_busy_wait(1000);

    /* 获取结束时间 */
    end_time = timing_counter_get();

    /* 计算时间差 */
    cycles = timing_cycles_get(&start_time, &end_time);
    ns = timing_cycles_to_ns(cycles);

    printk("Operation took %u cycles (%u ns)\n", cycles, ns);
}
```

## 4. 实时时钟(RTC)

### 4.1 RTC操作
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/counter.h>
#include <time.h>

void rtc_example(void)
{
    const struct device *rtc;
    uint32_t now;
    int ret;

    /* 获取RTC设备 */
    rtc = DEVICE_DT_GET(DT_NODELABEL(rtc0));
    if (!device_is_ready(rtc)) {
        printk("RTC device not ready\n");
        return;
    }

    /* 获取当前计数值 */
    ret = counter_get_value(rtc, &now);
    if (ret != 0) {
        printk("Failed to get counter value\n");
        return;
    }

    /* 转换为时间戳 */
    time_t timestamp = counter_ticks_to_us(rtc, now) / 1000000;
    struct tm *time_info = localtime(&timestamp);

    printk("Current time: %02d:%02d:%02d\n",
           time_info->tm_hour,
           time_info->tm_min,
           time_info->tm_sec);
}
```

### 4.2 RTC闹钟
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/counter.h>

/* 闹钟回调函数 */
static void alarm_callback(const struct device *dev,
                         uint8_t chan_id,
                         uint32_t ticks,
                         void *user_data)
{
    printk("Alarm triggered!\n");
}

void rtc_alarm_example(void)
{
    const struct device *rtc;
    uint32_t now;
    int ret;

    rtc = DEVICE_DT_GET(DT_NODELABEL(rtc0));
    if (!device_is_ready(rtc)) {
        return;
    }

    /* 获取当前时间 */
    ret = counter_get_value(rtc, &now);
    if (ret != 0) {
        return;
    }

    /* 配置闹钟 */
    struct counter_alarm_cfg alarm_cfg = {
        .callback = alarm_callback,
        .flags = 0,
        .ticks = now + counter_us_to_ticks(rtc, 5000000) /* 5秒后触发 */
    };

    ret = counter_set_channel_alarm(rtc, 0, &alarm_cfg);
    if (ret != 0) {
        printk("Failed to set alarm\n");
        return;
    }
}
```

## 5. 周期性任务

### 5.1 工作队列定时器
```c
#include <zephyr/kernel.h>

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
    struct k_work_delayable *dwork = k_work_delayable_from_work(work);
    k_work_schedule_for_queue(&my_work_q, dwork, K_SECONDS(1));
}

void periodic_work_example(void)
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
}
```

### 5.2 定时器线程
```c
#include <zephyr/kernel.h>

/* 定义线程栈 */
#define STACK_SIZE 1024
K_THREAD_STACK_DEFINE(timer_stack, STACK_SIZE);
struct k_thread timer_thread_data;

/* 线程入口函数 */
void timer_thread(void *p1, void *p2, void *p3)
{
    while (1) {
        /* 执行周期性任务 */
        printk("Timer thread running\n");

        /* 精确延时 */
        k_msleep(1000);
    }
}

void timer_thread_example(void)
{
    /* 创建定时器线程 */
    k_thread_create(&timer_thread_data, timer_stack,
                    STACK_SIZE,
                    timer_thread,
                    NULL, NULL, NULL,
                    K_PRIO_PREEMPT(10), 0, K_NO_WAIT);
}
```

## 6. 时间同步

### 6.1 时间同步配置
```c
#include <zephyr/kernel.h>
#include <zephyr/net/sntp.h>

void time_sync_example(void)
{
    struct sntp_ctx ctx;
    struct sockaddr_in addr;
    int ret;
    uint64_t epoch_time;

    /* 配置SNTP服务器地址 */
    addr.sin_family = AF_INET;
    addr.sin_port = htons(123);
    inet_pton(AF_INET, "pool.ntp.org", &addr.sin_addr);

    /* 初始化SNTP */
    ret = sntp_init(&ctx);
    if (ret < 0) {
        return;
    }

    /* 获取时间 */
    ret = sntp_query(&ctx, (struct sockaddr *)&addr,
                     sizeof(addr), &epoch_time);
    if (ret < 0) {
        sntp_close(&ctx);
        return;
    }

    /* 设置系统时间 */
    printk("Time synchronized: %llu\n", epoch_time);
    sntp_close(&ctx);
}
```

### 6.2 时区管理
```c
#include <zephyr/kernel.h>
#include <time.h>

void timezone_example(void)
{
    time_t now;
    struct tm *time_info;
    char time_str[64];

    /* 获取当前时间戳 */
    time(&now);

    /* 转换为本地时间 */
    time_info = localtime(&now);
    strftime(time_str, sizeof(time_str),
             "%Y-%m-%d %H:%M:%S", time_info);
    printk("Local time: %s\n", time_str);

    /* 转换为UTC时间 */
    time_info = gmtime(&now);
    strftime(time_str, sizeof(time_str),
             "%Y-%m-%d %H:%M:%S", time_info);
    printk("UTC time: %s\n", time_str);
}
```

## 7. 性能优化

### 7.1 时间测量优化
```c
#include <zephyr/kernel.h>
#include <zephyr/timing/timing.h>

void optimized_timing(void)
{
    uint32_t start, end;
    uint32_t min_time = UINT32_MAX;
    uint32_t max_time = 0;
    uint32_t total_time = 0;
    int count = 100;

    /* 预热CPU缓存 */
    k_busy_wait(1000);

    /* 多次测量以获得统计数据 */
    for (int i = 0; i < count; i++) {
        start = k_cycle_get_32();

        /* 被测量的代码 */
        k_busy_wait(100);

        end = k_cycle_get_32();
        uint32_t elapsed = end - start;

        min_time = MIN(min_time, elapsed);
        max_time = MAX(max_time, elapsed);
        total_time += elapsed;
    }

    printk("Timing statistics:\n");
    printk("Min: %u cycles\n", min_time);
    printk("Max: %u cycles\n", max_time);
    printk("Avg: %u cycles\n", total_time / count);
}
```

### 7.2 定时器优化
```c
#include <zephyr/kernel.h>

/* 使用静态定义的定时器 */
K_TIMER_DEFINE(static_timer, NULL, NULL);

/* 定时器池 */
#define TIMER_POOL_SIZE 4
static struct k_timer timer_pool[TIMER_POOL_SIZE];
static bool timer_used[TIMER_POOL_SIZE];

/* 从池中获取定时器 */
static struct k_timer *get_timer(void)
{
    for (int i = 0; i < TIMER_POOL_SIZE; i++) {
        if (!timer_used[i]) {
            timer_used[i] = true;
            return &timer_pool[i];
        }
    }
    return NULL;
}

/* 释放定时器回池 */
static void release_timer(struct k_timer *timer)
{
    for (int i = 0; i < TIMER_POOL_SIZE; i++) {
        if (timer == &timer_pool[i]) {
            timer_used[i] = false;
            return;
        }
    }
}

void timer_pool_example(void)
{
    struct k_timer *timer = get_timer();
    if (timer != NULL) {
        /* 使用定时器 */
        k_timer_start(timer, K_SECONDS(1), K_NO_WAIT);
        
        /* ... */
        
        /* 停止并释放定时器 */
        k_timer_stop(timer);
        release_timer(timer);
    }
}
```