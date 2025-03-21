---
abbrlink: 33
title: Zephyr 内存管理指南
date: '2025-03-21 20:49:56'
---
# Zephyr 内存管理指南

## 1. 内存管理配置

### 1.1 内存配置 (prj.conf)
```plaintext
# 堆内存配置
CONFIG_HEAP_MEM_POOL_SIZE=16384
CONFIG_HEAP_MEM_POOL_MIN_SIZE=64

# 系统内存池配置
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048
CONFIG_MAIN_STACK_SIZE=2048

# 内存调试配置
CONFIG_DEBUG_OPTIMIZATIONS=y
CONFIG_DEBUG_INFO=y
CONFIG_STACK_USAGE=y
CONFIG_STACK_SENTINEL=y
CONFIG_HEAP_MEMORY_INFO=y
```

### 1.2 内存分区示例 (boards/xxx.overlay)
```dts
/ {
    soc {
        sram0: memory@20000000 {
            compatible = "mmio-sram";
            reg = <0x20000000 0x40000>;
        };
    };

    chosen {
        zephyr,sram = &sram0;
    };
};
```

## 2. 静态内存分配

### 2.1 静态缓冲区
```c
#include <zephyr/kernel.h>

/* 静态数组 */
static uint8_t static_buffer[1024];

/* 静态结构体 */
static struct data_struct {
    uint32_t value;
    uint8_t buffer[64];
} static_data;

/* 对齐的静态缓冲区 */
K_ALIGNED(32) static uint8_t aligned_buffer[256];

int main(void)
{
    /* 使用静态缓冲区 */
    memset(static_buffer, 0, sizeof(static_buffer));
    static_data.value = 42;

    return 0;
}
```

### 2.2 线程栈
```c
#include <zephyr/kernel.h>

#define STACK_SIZE 1024

/* 定义线程栈 */
K_THREAD_STACK_DEFINE(thread_stack, STACK_SIZE);
struct k_thread thread_data;

void thread_entry(void *p1, void *p2, void *p3)
{
    /* 线程代码 */
    while (1) {
        k_msleep(1000);
    }
}

int main(void)
{
    /* 创建线程 */
    k_thread_create(&thread_data, thread_stack,
                    STACK_SIZE,
                    thread_entry,
                    NULL, NULL, NULL,
                    K_PRIO_PREEMPT(10), 0, K_NO_WAIT);

    return 0;
}
```

## 3. 动态内存分配

### 3.1 堆内存管理
```c
#include <zephyr/kernel.h>

void heap_example(void)
{
    /* 分配内存 */
    void *ptr = k_malloc(256);
    if (ptr == NULL) {
        printk("Failed to allocate memory\n");
        return;
    }

    /* 使用内存 */
    memset(ptr, 0, 256);

    /* 重新调整大小 */
    void *new_ptr = k_realloc(ptr, 512);
    if (new_ptr == NULL) {
        printk("Failed to reallocate memory\n");
        k_free(ptr);
        return;
    }
    ptr = new_ptr;

    /* 释放内存 */
    k_free(ptr);
}
```

### 3.2 内存池
```c
#include <zephyr/kernel.h>

/* 定义内存池 */
K_MEM_POOL_DEFINE(my_pool,    /* 名称 */
                  32,         /* 最小块大小 */
                  256,        /* 最大块大小 */
                  8,          /* 最大块数量 */
                  4);         /* 对齐大小 */

void memory_pool_example(void)
{
    struct k_mem_block block;
    int ret;

    /* 分配内存块 */
    ret = k_mem_pool_alloc(&my_pool, &block, 128, K_NO_WAIT);
    if (ret != 0) {
        printk("Failed to allocate memory block\n");
        return;
    }

    /* 使用内存块 */
    memset(block.data, 0, 128);

    /* 释放内存块 */
    k_mem_pool_free(&block);
}
```

## 4. 共享内存

### 4.1 共享内存区域
```c
#include <zephyr/kernel.h>
#include <zephyr/sys/mem_manage.h>

/* 定义共享内存区域 */
K_APPMEM_PARTITION_DEFINE(shared_mem_partition);
K_APP_DMEM(shared_mem_partition) uint8_t shared_data[1024];

void shared_memory_example(void)
{
    /* 配置内存保护 */
    k_mem_domain_init();
    
    /* 使用共享内存 */
    memset(shared_data, 0, sizeof(shared_data));
}
```

### 4.2 内存映射
```c
#include <zephyr/kernel.h>
#include <zephyr/sys/mmap.h>

void mmap_example(void)
{
    void *addr;
    int ret;

    /* 映射内存区域 */
    ret = k_mem_map(&addr, 4096, K_MEM_PERM_RW);
    if (ret != 0) {
        printk("Memory mapping failed\n");
        return;
    }

    /* 使用映射的内存 */
    memset(addr, 0, 4096);

    /* 取消映射 */
    k_mem_unmap(addr, 4096);
}
```

## 5. 内存保护

### 5.1 用户空间内存
```c
#include <zephyr/kernel.h>
#include <zephyr/app_memory/app_memdomain.h>

/* 定义用户空间内存分区 */
K_APPMEM_PARTITION_DEFINE(app_partition);

/* 在分区中定义数据 */
K_APP_DMEM(app_partition) uint8_t app_data[256];
K_APP_BMEM(app_partition) static volatile int app_flag;

void memory_protection_example(void)
{
    /* 配置内存域 */
    struct k_mem_domain app_domain;
    k_mem_domain_init(&app_domain, 0, NULL);
    
    /* 添加内存分区 */
    k_mem_domain_add_partition(&app_domain, &app_partition);
}
```

### 5.2 栈保护
```c
#include <zephyr/kernel.h>

#define STACK_SIZE 1024

/* 启用栈保护的线程栈 */
K_THREAD_STACK_DEFINE(protected_stack, STACK_SIZE);
struct k_thread protected_thread_data;

void protected_thread(void *p1, void *p2, void *p3)
{
    volatile uint8_t buffer[256];
    
    /* 使用栈内存 */
    memset((void *)buffer, 0, sizeof(buffer));
    
    while (1) {
        k_msleep(1000);
    }
}

int main(void)
{
    /* 创建启用栈保护的线程 */
    k_thread_create(&protected_thread_data,
                    protected_stack,
                    STACK_SIZE,
                    protected_thread,
                    NULL, NULL, NULL,
                    K_PRIO_PREEMPT(10),
                    K_INHERIT_PERMS,
                    K_NO_WAIT);

    return 0;
}
```

## 6. 内存调试

### 6.1 内存使用统计
```c
#include <zephyr/kernel.h>
#include <zephyr/debug/heap.h>

void memory_stats(void)
{
    struct k_heap_stats stats;

    /* 获取堆统计信息 */
    k_heap_stats_get(&stats);

    printk("Heap Statistics:\n");
    printk("Total bytes: %zu\n", stats.total_bytes);
    printk("Used bytes: %zu\n", stats.used_bytes);
    printk("Free bytes: %zu\n", stats.free_bytes);
    printk("Allocated blocks: %zu\n", stats.allocated_blocks);
    printk("Free blocks: %zu\n", stats.free_blocks);
}
```

### 6.2 内存泄漏检测
```c
#include <zephyr/kernel.h>
#include <zephyr/debug/heap.h>

void leak_detection(void)
{
    void *ptr1, *ptr2;
    
    /* 分配内存 */
    ptr1 = k_malloc(100);
    ptr2 = k_malloc(200);
    
    /* 打印堆内存状态 */
    k_heap_dump(NULL);
    
    /* 释放部分内存（模拟泄漏） */
    k_free(ptr1);
    // k_free(ptr2); // 故意不释放以模拟泄漏
    
    /* 再次打印堆状态以检测泄漏 */
    k_heap_dump(NULL);
}
```

## 7. 最佳实践

### 7.1 内存分配策略
```c
#include <zephyr/kernel.h>

/* 使用静态分配的缓冲池 */
#define POOL_SIZE 4
#define BLOCK_SIZE 256

static uint8_t buffer_pool[POOL_SIZE][BLOCK_SIZE];
static bool buffer_used[POOL_SIZE];

/* 从池中分配缓冲区 */
static uint8_t *alloc_buffer(void)
{
    for (int i = 0; i < POOL_SIZE; i++) {
        if (!buffer_used[i]) {
            buffer_used[i] = true;
            return buffer_pool[i];
        }
    }
    return NULL;
}

/* 释放缓冲区 */
static void free_buffer(uint8_t *buffer)
{
    for (int i = 0; i < POOL_SIZE; i++) {
        if (buffer == buffer_pool[i]) {
            buffer_used[i] = false;
            return;
        }
    }
}

void memory_strategy_example(void)
{
    /* 使用预分配的缓冲区 */
    uint8_t *buf = alloc_buffer();
    if (buf != NULL) {
        /* 使用缓冲区 */
        memset(buf, 0, BLOCK_SIZE);
        /* 完成后释放 */
        free_buffer(buf);
    }
}
```

### 7.2 错误处理
```c
#include <zephyr/kernel.h>

void error_handling_example(void)
{
    void *ptr;
    int ret;

    /* 分配内存 */
    ptr = k_malloc(1024);
    if (ptr == NULL) {
        /* 处理内存分配失败 */
        printk("Memory allocation failed\n");
        /* 尝试恢复策略 */
        k_mem_pool_defrag(NULL);
        return;
    }

    /* 使用try-catch风格的错误处理 */
    ret = some_operation(ptr);
    if (ret < 0) {
        goto cleanup;
    }

    /* 成功处理 */
    printk("Operation successful\n");

cleanup:
    k_free(ptr);
}
```

### 7.3 内存对齐
```c
#include <zephyr/kernel.h>

/* 对齐结构体 */
typedef struct {
    uint32_t value;
    uint8_t data[32];
} __aligned(32) aligned_struct_t;

void alignment_example(void)
{
    /* 静态分配对齐的内存 */
    static aligned_struct_t aligned_data;

    /* 动态分配对齐的内存 */
    aligned_struct_t *ptr = k_aligned_alloc(32, sizeof(aligned_struct_t));
    if (ptr != NULL) {
        /* 使用对齐的内存 */
        ptr->value = 42;
        memset(ptr->data, 0, sizeof(ptr->data));
        
        /* 释放内存 */
        k_free(ptr);
    }
}
```