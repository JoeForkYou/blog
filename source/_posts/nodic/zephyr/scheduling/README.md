---
abbrlink: 37
title: Zephyr 调度系统指南
date: '2025-03-21 20:49:56'
---
# Zephyr 任务调度指南

## 1. 调度器配置

### 1.1 基础配置 (prj.conf)
```plaintext
# 调度器配置
CONFIG_MULTITHREADING=y
CONFIG_NUM_PREEMPT_PRIORITIES=16
CONFIG_NUM_COOP_PRIORITIES=16
CONFIG_TIMESLICING=y
CONFIG_TIMESLICE_SIZE=10
CONFIG_TIMESLICE_PRIORITY=0

# 线程配置
CONFIG_MAIN_STACK_SIZE=2048
CONFIG_IDLE_STACK_SIZE=512
CONFIG_THREAD_STACK_INFO=y
CONFIG_THREAD_MONITOR=y
CONFIG_THREAD_NAME=y

# 调度器调试
CONFIG_SCHED_THREAD_USAGE=y
CONFIG_THREAD_RUNTIME_STATS=y
```

### 1.2 线程优先级定义
```c
/* 优先级定义 */
#define THREAD_PRIORITY_HIGH    K_PRIO_PREEMPT(2)
#define THREAD_PRIORITY_NORMAL  K_PRIO_PREEMPT(8)
#define THREAD_PRIORITY_LOW     K_PRIO_PREEMPT(14)
```

## 2. 线程创建与管理

### 2.1 基本线程创建
```c
#include <zephyr/kernel.h>

/* 定义线程栈和数据结构 */
#define STACK_SIZE 1024
K_THREAD_STACK_DEFINE(thread_stack, STACK_SIZE);
struct k_thread thread_data;

/* 线程入口函数 */
void thread_entry(void *p1, void *p2, void *p3)
{
    while (1) {
        printk("Thread running\n");
        k_msleep(1000);
    }
}

int main(void)
{
    k_tid_t tid;

    /* 创建线程 */
    tid = k_thread_create(&thread_data, thread_stack,
                         STACK_SIZE,
                         thread_entry,
                         NULL, NULL, NULL,
                         THREAD_PRIORITY_NORMAL,
                         0, K_NO_WAIT);

    /* 设置线程名称 */
    k_thread_name_set(tid, "example_thread");

    return 0;
}
```

### 2.2 线程生命周期管理
```c
#include <zephyr/kernel.h>

void thread_lifecycle(void)
{
    k_tid_t tid;
    
    /* 创建线程 */
    tid = k_thread_create(...);
    
    /* 挂起线程 */
    k_thread_suspend(tid);
    
    /* 恢复线程 */
    k_thread_resume(tid);
    
    /* 中止线程 */
    k_thread_abort(tid);
    
    /* 等待线程结束 */
    k_thread_join(&thread_data, K_FOREVER);
}
```

## 3. 调度策略

### 3.1 抢占式调度
```c
#include <zephyr/kernel.h>

/* 高优先级线程 */
void high_priority_thread(void *p1, void *p2, void *p3)
{
    while (1) {
        /* 执行高优先级任务 */
        k_msleep(100);
    }
}

/* 低优先级线程 */
void low_priority_thread(void *p1, void *p2, void *p3)
{
    while (1) {
        /* 执行低优先级任务 */
        k_msleep(200);
    }
}

int main(void)
{
    /* 创建高优先级线程 */
    k_thread_create(..., high_priority_thread, ...,
                   K_PRIO_PREEMPT(2), 0, K_NO_WAIT);
    
    /* 创建低优先级线程 */
    k_thread_create(..., low_priority_thread, ...,
                   K_PRIO_PREEMPT(8), 0, K_NO_WAIT);

    return 0;
}
```

### 3.2 时间片轮转
```c
#include <zephyr/kernel.h>

/* 定义多个相同优先级的线程 */
void thread_function(void *p1, void *p2, void *p3)
{
    while (1) {
        /* 执行任务 */
        printk("Thread %p running\n", k_current_get());
        k_msleep(100);
    }
}

int main(void)
{
    /* 创建多个相同优先级的线程 */
    k_thread_create(..., thread_function, ...,
                   K_PRIO_PREEMPT(5), 0, K_NO_WAIT);
    k_thread_create(..., thread_function, ...,
                   K_PRIO_PREEMPT(5), 0, K_NO_WAIT);
    k_thread_create(..., thread_function, ...,
                   K_PRIO_PREEMPT(5), 0, K_NO_WAIT);

    return 0;
}
```

## 4. 线程同步

### 4.1 互斥量
```c
#include <zephyr/kernel.h>

/* 定义互斥量 */
K_MUTEX_DEFINE(my_mutex);

void mutex_example(void)
{
    /* 获取互斥量 */
    if (k_mutex_lock(&my_mutex, K_MSEC(100)) == 0) {
        /* 临界区代码 */
        
        /* 释放互斥量 */
        k_mutex_unlock(&my_mutex);
    }
}
```

### 4.2 信号量
```c
#include <zephyr/kernel.h>

/* 定义信号量 */
K_SEM_DEFINE(my_sem, 0, 1);

void semaphore_example(void)
{
    /* 等待信号量 */
    if (k_sem_take(&my_sem, K_MSEC(100)) == 0) {
        /* 获得信号量后的代码 */
    }
    
    /* 释放信号量 */
    k_sem_give(&my_sem);
}
```

## 5. 任务通信

### 5.1 消息队列
```c
#include <zephyr/kernel.h>

/* 定义消息队列 */
K_MSGQ_DEFINE(my_msgq, sizeof(uint32_t), 10, 4);

void msgq_example(void)
{
    uint32_t data = 42;
    
    /* 发送消息 */
    k_msgq_put(&my_msgq, &data, K_NO_WAIT);
    
    /* 接收消息 */
    uint32_t received;
    if (k_msgq_get(&my_msgq, &received, K_FOREVER) == 0) {
        printk("Received: %u\n", received);
    }
}
```

### 5.2 管道
```c
#include <zephyr/kernel.h>

/* 定义管道 */
K_PIPE_DEFINE(my_pipe, 64, 4);

void pipe_example(void)
{
    uint8_t data[] = "Hello";
    size_t bytes_written;
    
    /* 写入数据 */
    k_pipe_put(&my_pipe, data, sizeof(data),
               &bytes_written, 1, K_NO_WAIT);
    
    /* 读取数据 */
    uint8_t buffer[64];
    size_t bytes_read;
    k_pipe_get(&my_pipe, buffer, sizeof(buffer),
               &bytes_read, 1, K_NO_WAIT);
}
```

## 6. 任务监控

### 6.1 线程统计
```c
#include <zephyr/kernel.h>
#include <zephyr/timing/timing.h>

void thread_monitor(void)
{
    k_thread_runtime_stats_t stats;
    
    /* 获取线程运行时统计 */
    k_thread_runtime_stats_all_get(&stats);
    
    printk("Total cycles: %llu\n", stats.total_cycles);
    printk("Execution cycles: %llu\n", stats.execution_cycles);
    printk("Idle cycles: %llu\n", stats.idle_cycles);
}
```

### 6.2 调度器调试
```c
#include <zephyr/kernel.h>

void scheduler_debug(void)
{
    struct k_thread *thread;
    
    /* 遍历所有线程 */
    for (thread = _kernel.threads; thread; thread = thread->next_thread) {
        printk("Thread %p: priority %d, state %d\n",
               thread, thread->base.prio,
               thread->base.thread_state);
    }
}
```

## 7. 高级特性

### 7.1 工作队列
```c
#include <zephyr/kernel.h>

/* 定义工作队列 */
K_THREAD_STACK_DEFINE(my_stack_area, 1024);
struct k_work_q my_work_q;

/* 工作项处理函数 */
void work_handler(struct k_work *work)
{
    printk("Work executed\n");
}

/* 定义工作项 */
K_WORK_DEFINE(my_work, work_handler);

int main(void)
{
    /* 初始化工作队列 */
    k_work_queue_init(&my_work_q);
    
    /* 启动工作队列 */
    k_work_queue_start(&my_work_q, my_stack_area,
                      K_THREAD_STACK_SIZEOF(my_stack_area),
                      K_PRIO_PREEMPT(10), NULL);
    
    /* 提交工作 */
    k_work_submit_to_queue(&my_work_q, &my_work);

    return 0;
}
```

### 7.2 定时器线程
```c
#include <zephyr/kernel.h>

/* 定时器回调 */
void timer_handler(struct k_timer *timer)
{
    printk("Timer expired\n");
}

void timer_thread_example(void)
{
    struct k_timer my_timer;
    
    /* 初始化定时器 */
    k_timer_init(&my_timer, timer_handler, NULL);
    
    /* 启动定时器 */
    k_timer_start(&my_timer, K_MSEC(1000), K_MSEC(1000));
}
```

### 7.3 空闲线程钩子
```c
#include <zephyr/kernel.h>

/* 空闲线程钩子函数 */
void idle_hook(void)
{
    /* 在系统空闲时执行 */
    printk("System is idle\n");
}

int main(void)
{
    /* 注册空闲钩子 */
    k_idle_thread_set_callback(idle_hook);

    return 0;
}
```