---
abbrlink: 23
title: Zephyr 错误处理指南
date: '2025-03-21 20:49:56'
---
# Zephyr 错误处理指南

## 1. 错误码系统

### 1.1 标准错误码
```c
#include <zephyr/sys/errno.h>

/* 常见错误码 */
#define EPERM           1  /* Operation not permitted */
#define ENOENT          2  /* No such file or directory */
#define ESRCH           3  /* No such process */
#define EINTR           4  /* Interrupted system call */
#define EIO             5  /* I/O error */
#define ENXIO           6  /* No such device or address */
#define E2BIG           7  /* Argument list too long */
#define ENOEXEC         8  /* Exec format error */
#define EBADF           9  /* Bad file number */
#define ECHILD         10  /* No child processes */
#define EAGAIN         11  /* Try again */
#define ENOMEM         12  /* Out of memory */
#define EACCES         13  /* Permission denied */
#define EFAULT         14  /* Bad address */
#define EBUSY          16  /* Device or resource busy */
#define EEXIST         17  /* File exists */
#define EXDEV          18  /* Cross-device link */
#define ENODEV         19  /* No such device */
#define ENOTDIR        20  /* Not a directory */
#define EISDIR         21  /* Is a directory */
#define EINVAL         22  /* Invalid argument */
#define ENFILE         23  /* File table overflow */
#define EMFILE         24  /* Too many open files */
#define ENOSPC         28  /* No space left on device */
```

### 1.2 错误码使用示例
```c
#include <zephyr/kernel.h>
#include <zephyr/sys/errno.h>

int file_operation_example(const char *filename)
{
    struct fs_file_t file;
    int ret;

    /* 初始化文件对象 */
    fs_file_t_init(&file);

    /* 打开文件 */
    ret = fs_open(&file, filename, FS_O_READ);
    if (ret < 0) {
        if (ret == -ENOENT) {
            printk("File not found: %s\n", filename);
        } else if (ret == -EACCES) {
            printk("Permission denied: %s\n", filename);
        } else {
            printk("Failed to open file: %d\n", ret);
        }
        return ret;
    }

    /* 关闭文件 */
    fs_close(&file);
    return 0;
}
```

## 2. 断言机制

### 2.1 基本断言
```c
#include <zephyr/sys/__assert.h>

void assert_example(void *ptr, size_t size)
{
    /* 参数检查 */
    __ASSERT(ptr != NULL, "Null pointer");
    __ASSERT(size > 0, "Invalid size");
    __ASSERT(size <= 1024, "Size too large");

    /* 状态检查 */
    __ASSERT(k_is_in_isr() == false,
             "Cannot be called from ISR");

    /* 复杂条件检查 */
    __ASSERT((size & (size - 1)) == 0,
             "Size must be power of 2");
}
```

### 2.2 条件编译断言
```c
#include <zephyr/sys/__assert.h>

/* 仅在调试模式启用的断言 */
__ASSERT_NO_MSG(condition);

/* 带消息的条件编译断言 */
#ifdef CONFIG_DEBUG
    __ASSERT(condition, "Error message");
#endif

/* 永久断言（不受配置影响） */
__ASSERT_ALWAYS(condition, "Critical error");
```

## 3. 异常处理

### 3.1 故障处理
```c
#include <zephyr/kernel.h>

/* 故障处理函数 */
void k_sys_fatal_error_handler(unsigned int reason,
                             const z_arch_esf_t *esf)
{
    printk("Fatal system error! Reason: %u\n", reason);

    /* 打印错误信息 */
    switch (reason) {
    case K_ERR_CPU_EXCEPTION:
        printk("CPU Exception\n");
        break;
    case K_ERR_SPURIOUS_IRQ:
        printk("Spurious interrupt\n");
        break;
    case K_ERR_STACK_CHK_FAIL:
        printk("Stack overflow\n");
        break;
    default:
        printk("Unknown error\n");
        break;
    }

    /* 系统重置 */
    sys_reboot(SYS_REBOOT_COLD);
}
```

### 3.2 内存保护错误
```c
#include <zephyr/kernel.h>

void memory_protection_example(void)
{
    /* 使用try-catch风格的错误处理 */
    if (k_mem_domain_add_thread(domain, k_current_get()) != 0) {
        /* 处理内存域添加失败 */
        return;
    }

    /* 访问保护内存 */
    if (k_mem_paging_is_enabled()) {
        /* 处理页面错误 */
        k_mem_paging_stats_get();
    }
}
```

## 4. 错误恢复

### 4.1 优雅降级
```c
#include <zephyr/kernel.h>

void graceful_degradation(void)
{
    int retry_count = 3;
    int ret;

    while (retry_count--) {
        ret = perform_operation();
        if (ret == 0) {
            /* 操作成功 */
            break;
        } else if (ret == -EAGAIN) {
            /* 临时错误，重试 */
            k_msleep(100);
            continue;
        } else {
            /* 严重错误，降级处理 */
            handle_fallback();
            break;
        }
    }
}
```

### 4.2 系统重启
```c
#include <zephyr/kernel.h>
#include <zephyr/sys/reboot.h>

void system_recovery(void)
{
    /* 保存关键数据 */
    save_critical_data();

    /* 根据错误类型选择重启方式 */
    if (is_hardware_error()) {
        /* 冷重启 */
        sys_reboot(SYS_REBOOT_COLD);
    } else {
        /* 热重启 */
        sys_reboot(SYS_REBOOT_WARM);
    }
}
```

## 5. 日志记录

### 5.1 错误日志
```c
#include <zephyr/logging/log.h>
LOG_MODULE_REGISTER(error_log, LOG_LEVEL_DBG);

void error_logging_example(int status)
{
    /* 不同级别的日志 */
    if (status < 0) {
        LOG_ERR("Operation failed: %d", status);
    } else if (status > 0) {
        LOG_WRN("Operation completed with warnings");
    } else {
        LOG_INF("Operation successful");
    }

    /* 带hex dump的日志 */
    uint8_t data[] = {0x01, 0x02, 0x03, 0x04};
    LOG_HEXDUMP_ERR(data, sizeof(data), "Error data:");
}
```

### 5.2 错误追踪
```c
#include <zephyr/logging/log.h>
#include <zephyr/debug/stack.h>

void error_trace_example(void)
{
    /* 打印调用栈 */
    k_thread_stack_print_debug(k_current_get());

    /* 记录错误现场 */
    LOG_ERR("Error context:");
    LOG_ERR("Thread ID: %p", k_current_get());
    LOG_ERR("CPU: %d", arch_curr_cpu()->id);
    LOG_ERR("Priority: %d", k_thread_priority_get(k_current_get()));
}
```

## 6. 错误注入

### 6.1 故障注入
```c
#include <zephyr/kernel.h>

/* 故障注入配置 */
struct fault_inject_config {
    bool enabled;
    int error_rate;    /* 错误率（百分比） */
    int error_type;    /* 错误类型 */
};

/* 故障注入函数 */
int inject_fault(const struct fault_inject_config *config)
{
    if (config->enabled) {
        /* 根据错误率决定是否注入错误 */
        if (sys_rand32_get() % 100 < config->error_rate) {
            switch (config->error_type) {
            case 1:
                return -ENOMEM;
            case 2:
                return -EIO;
            default:
                return -EINVAL;
            }
        }
    }
    return 0;
}
```

### 6.2 测试用例
```c
#include <zephyr/ztest.h>

ZTEST_SUITE(error_handling_tests, NULL, NULL, NULL, NULL, NULL);

/* 测试错误处理 */
ZTEST(error_handling_tests, test_error_handling)
{
    struct fault_inject_config config = {
        .enabled = true,
        .error_rate = 100,  /* 总是注入错误 */
        .error_type = 1
    };

    /* 验证错误处理 */
    int ret = inject_fault(&config);
    zassert_equal(ret, -ENOMEM,
                 "Expected ENOMEM error");
}
```

## 7. 错误处理最佳实践

### 7.1 错误传播
```c
#include <zephyr/kernel.h>

/* 错误传播示例 */
int process_data(const uint8_t *data, size_t len)
{
    int ret;

    /* 参数验证 */
    if (data == NULL || len == 0) {
        return -EINVAL;
    }

    /* 处理数据 */
    ret = validate_data(data, len);
    if (ret < 0) {
        /* 传播错误，保持原始错误码 */
        return ret;
    }

    ret = transform_data(data, len);
    if (ret < 0) {
        /* 可以添加额外的错误信息 */
        LOG_ERR("Data transformation failed: %d", ret);
        return ret;
    }

    return 0;
}
```

### 7.2 清理处理
```c
#include <zephyr/kernel.h>

/* 资源清理示例 */
int resource_management(void)
{
    void *resource1 = NULL;
    void *resource2 = NULL;
    int ret;

    /* 分配第一个资源 */
    resource1 = allocate_resource1();
    if (resource1 == NULL) {
        ret = -ENOMEM;
        goto error;
    }

    /* 分配第二个资源 */
    resource2 = allocate_resource2();
    if (resource2 == NULL) {
        ret = -ENOMEM;
        goto error;
    }

    /* 使用资源 */
    ret = use_resources(resource1, resource2);
    if (ret < 0) {
        goto error;
    }

    /* 成功路径 */
    free_resource2(resource2);
    free_resource1(resource1);
    return 0;

error:
    /* 错误路径，清理资源 */
    if (resource2 != NULL) {
        free_resource2(resource2);
    }
    if (resource1 != NULL) {
        free_resource1(resource1);
    }
    return ret;
}
```