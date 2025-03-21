---
abbrlink: 27
title: interrupt
date: '2025-03-21 20:49:56'
---
# Zephyr 中断处理示例

## 1. 中断配置

### 1.1 基础配置 (prj.conf)
```plaintext
# 基础配置
CONFIG_PRINTK=y
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y

# 中断配置
CONFIG_IRQ_OFFLOAD=y
CONFIG_GEN_IRQ_VECTOR_TABLE=y

# GPIO中断配置（用于示例）
CONFIG_GPIO=y
CONFIG_GPIO_INTERRUPT=y
```

## 2. GPIO中断示例

### 2.1 基本GPIO中断
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/* 定义GPIO设备和引脚 */
static const struct gpio_dt_spec button = GPIO_DT_SPEC_GET(DT_NODELABEL(button0), gpios);
static struct gpio_callback button_cb_data;

/* 中断回调函数 */
void button_pressed_callback(const struct device *dev,
                           struct gpio_callback *cb,
                           uint32_t pins)
{
    printk("Button pressed! Pin: %d\n", button.pin);
}

int main(void)
{
    int ret;

    if (!device_is_ready(button.port)) {
        printk("Error: button device %s is not ready\n", button.port->name);
        return -ENODEV;
    }

    /* 配置GPIO为输入，启用上拉和中断 */
    ret = gpio_pin_configure_dt(&button,
                              GPIO_INPUT | GPIO_PULL_UP |
                              GPIO_INT_EDGE_TO_ACTIVE);
    if (ret != 0) {
        printk("Error %d: failed to configure button\n", ret);
        return ret;
    }

    /* 初始化GPIO回调 */
    gpio_init_callback(&button_cb_data,
                      button_pressed_callback,
                      BIT(button.pin));

    /* 添加回调 */
    ret = gpio_add_callback(button.port, &button_cb_data);
    if (ret != 0) {
        printk("Error %d: failed to add callback\n", ret);
        return ret;
    }

    /* 使能中断 */
    ret = gpio_pin_interrupt_configure_dt(&button, GPIO_INT_EDGE_TO_ACTIVE);
    if (ret != 0) {
        printk("Error %d: failed to configure interrupt\n", ret);
        return ret;
    }

    printk("Press the button\n");
    while (1) {
        k_msleep(100);
    }

    return 0;
}
```

## 3. 中断优先级

### 3.1 优先级配置示例
```c
#include <zephyr/kernel.h>
#include <zephyr/irq.h>

/* 定义IRQ处理函数 */
void my_isr(const void *unused)
{
    ARG_UNUSED(unused);
    printk("ISR executed!\n");
}

int main(void)
{
    /* 配置中断 */
    IRQ_CONNECT(MY_IRQ_NUM,    /* IRQ号 */
                0,             /* 优先级 */
                my_isr,       /* ISR */
                NULL,         /* ISR参数 */
                0);           /* 标志 */

    /* 使能中断 */
    irq_enable(MY_IRQ_NUM);

    return 0;
}
```

## 4. 中断上下文

### 4.1 中断安全代码
```c
#include <zephyr/kernel.h>
#include <zephyr/irq.h>

/* 共享数据 */
static volatile int shared_data;
static K_MUTEX_DEFINE(data_mutex);

/* 中断处理函数 */
void data_isr(const void *unused)
{
    ARG_UNUSED(unused);

    /* 在中断上下文中安全地修改数据 */
    shared_data++;
}

/* 线程上下文函数 */
void thread_function(void)
{
    int local_copy;

    /* 在线程上下文中安全地访问数据 */
    k_mutex_lock(&data_mutex, K_FOREVER);
    local_copy = shared_data;
    k_mutex_unlock(&data_mutex);

    printk("Data value: %d\n", local_copy);
}
```

## 5. 嵌套中断

### 5.1 嵌套中断示例
```c
#include <zephyr/kernel.h>
#include <zephyr/irq.h>

/* 高优先级中断处理函数 */
void high_priority_isr(const void *unused)
{
    ARG_UNUSED(unused);
    printk("High priority ISR\n");
}

/* 低优先级中断处理函数 */
void low_priority_isr(const void *unused)
{
    ARG_UNUSED(unused);
    printk("Low priority ISR start\n");
    
    /* 模拟一些处理时间 */
    k_busy_wait(1000);
    
    printk("Low priority ISR end\n");
}

int main(void)
{
    /* 配置中断优先级 */
    IRQ_CONNECT(HIGH_PRIORITY_IRQ, 0, high_priority_isr, NULL, 0);
    IRQ_CONNECT(LOW_PRIORITY_IRQ, 1, low_priority_isr, NULL, 0);

    /* 使能中断 */
    irq_enable(HIGH_PRIORITY_IRQ);
    irq_enable(LOW_PRIORITY_IRQ);

    return 0;
}
```

## 6. 软件中断

### 6.1 软件中断示例
```c
#include <zephyr/kernel.h>
#include <zephyr/irq.h>

/* 定义软件中断处理函数 */
void soft_isr(const void *unused)
{
    ARG_UNUSED(unused);
    printk("Software interrupt executed\n");
}

int main(void)
{
    /* 配置软件中断 */
    IRQ_CONNECT(CONFIG_IRQ_OFFLOAD_IRQ, 0, soft_isr, NULL, 0);
    irq_enable(CONFIG_IRQ_OFFLOAD_IRQ);

    /* 触发软件中断 */
    irq_offload(soft_isr, NULL);

    return 0;
}
```

## 7. 中断延迟处理

### 7.1 工作队列延迟处理
```c
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

/* 定义工作队列 */
static struct k_work_q isr_work_q;
static K_THREAD_STACK_DEFINE(isr_work_q_stack, 1024);

/* 定义工作项 */
static struct k_work isr_work;

/* 工作处理函数 */
static void isr_work_handler(struct k_work *work)
{
    /* 在线程上下文中处理中断事件 */
    printk("Processing interrupt event\n");
}

/* 中断处理函数 */
static void gpio_isr(const struct device *dev,
                    struct gpio_callback *cb,
                    uint32_t pins)
{
    /* 提交工作到工作队列 */
    k_work_submit_to_queue(&isr_work_q, &isr_work);
}

int main(void)
{
    int ret;

    /* 初始化工作队列 */
    k_work_queue_init(&isr_work_q);
    k_work_queue_start(&isr_work_q,
                      isr_work_q_stack,
                      K_THREAD_STACK_SIZEOF(isr_work_q_stack),
                      K_PRIO_PREEMPT(10),
                      NULL);

    /* 初始化工作项 */
    k_work_init(&isr_work, isr_work_handler);

    /* 配置GPIO中断 */
    // ... GPIO配置代码 ...

    return 0;
}
```

## 8. 中断调试

### 8.1 中断调试技巧
```c
#include <zephyr/kernel.h>
#include <zephyr/irq.h>

/* 中断统计结构 */
struct irq_stats {
    uint32_t count;
    uint32_t last_time;
    uint32_t max_latency;
};

static struct irq_stats stats;

/* 带调试信息的中断处理函数 */
void debug_isr(const void *unused)
{
    uint32_t current_time = k_cycle_get_32();
    uint32_t latency;

    /* 更新统计信息 */
    stats.count++;
    latency = current_time - stats.last_time;
    if (latency > stats.max_latency) {
        stats.max_latency = latency;
    }
    stats.last_time = current_time;

    /* 打印调试信息 */
    printk("IRQ triggered: count=%u, latency=%u cycles\n",
           stats.count, latency);
}

/* 打印中断统计信息 */
void print_irq_stats(void)
{
    printk("IRQ Statistics:\n");
    printk("Total count: %u\n", stats.count);
    printk("Max latency: %u cycles\n", stats.max_latency);
}
```

### 8.2 中断锁定检测
```c
#include <zephyr/kernel.h>

void check_irq_lock(void)
{
    unsigned int key;
    bool was_locked;

    /* 检查中断是否被锁定 */
    was_locked = irq_lock_is_locked();
    if (was_locked) {
        printk("WARNING: IRQs are locked!\n");
    }

    /* 安全地锁定/解锁中断 */
    key = irq_lock();
    /* 临界区代码 */
    irq_unlock(key);
}
```