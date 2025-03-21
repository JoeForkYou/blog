---
title: Zephyr 系统配置指南
tags: zephyr
categories: zephyr
abbrlink: 15164
date: 2025-03-21 01:15:27
mermaid: true
---

# Zephyr 系统配置指南

## 版本信息
- 版本：V1.0
- 更新时间：2025年03月21日 01:15

## 1. 配置系统概述

### 1.1 配置文件层次
```plaintext
项目目录/
├── prj.conf           # 项目配置
├── boards/
│   └── xxx.conf      # 板级配置
├── arch/
│   └── xxx.conf      # 架构配置
└── soc/
    └── xxx.conf      # SoC配置
```

### 1.2 配置优先级
1. 命令行配置 (-DCONFIG_XXX=Y)
2. 项目配置 (prj.conf)
3. 应用程序覆盖配置 (app.overlay)
4. 板级配置 (boards/xxx.conf)
5. SoC配置 (soc/xxx.conf)
6. 架构配置 (arch/xxx.conf)
7. 默认配置 (Kconfig默认值)

## 2. 项目配置

### 2.1 基本配置 (prj.conf)
```plaintext
# 系统配置
CONFIG_MAIN_STACK_SIZE=2048
CONFIG_HEAP_MEM_POOL_SIZE=16384
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048

# 内核配置
CONFIG_MULTITHREADING=y
CONFIG_NUM_PREEMPT_PRIORITIES=16
CONFIG_TIMESLICING=y

# 调试配置
CONFIG_DEBUG=y
CONFIG_DEBUG_INFO=y
CONFIG_STACK_USAGE=y
CONFIG_THREAD_MONITOR=y

# 控制台配置
CONFIG_CONSOLE=y
CONFIG_UART_CONSOLE=y
CONFIG_SERIAL=y
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3
```

### 2.2 设备树覆盖 (app.overlay)
```dts
/ {
    chosen {
        zephyr,console = &uart0;
        zephyr,shell-uart = &uart0;
        zephyr,uart-mcumgr = &uart0;
    };

    leds {
        compatible = "gpio-leds";
        led0: led_0 {
            gpios = <&gpio0 13 GPIO_ACTIVE_LOW>;
            label = "Green LED 0";
        };
    };

    buttons {
        compatible = "gpio-keys";
        button0: button_0 {
            gpios = <&gpio0 11 GPIO_ACTIVE_LOW>;
            label = "Push button switch 0";
        };
    };
};
```

## 3. 内核配置

### 3.1 内存配置
```plaintext
# 堆内存配置
CONFIG_HEAP_MEM_POOL_SIZE=16384
CONFIG_HEAP_MEM_POOL_MIN_SIZE=64

# 栈配置
CONFIG_MAIN_STACK_SIZE=2048
CONFIG_IDLE_STACK_SIZE=512
CONFIG_ISR_STACK_SIZE=2048
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048

# 内存保护
CONFIG_MPU=y
CONFIG_MPU_STACK_GUARD=y
CONFIG_USERSPACE=y
```

### 3.2 调度器配置
```plaintext
# 线程配置
CONFIG_NUM_PREEMPT_PRIORITIES=16
CONFIG_NUM_COOP_PRIORITIES=16
CONFIG_TIMESLICING=y
CONFIG_TIMESLICE_SIZE=10
CONFIG_TIMESLICE_PRIORITY=0

# 工作队列配置
CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048
CONFIG_SYSTEM_WORKQUEUE_PRIORITY=0
```

## 4. 驱动配置

### 4.1 GPIO配置
```plaintext
# GPIO配置
CONFIG_GPIO=y
CONFIG_GPIO_SHELL=y

# 中断配置
CONFIG_GPIO_INTERRUPT=y
CONFIG_GPIO_INIT_PRIORITY=40
```

### 4.2 串口配置
```plaintext
# UART配置
CONFIG_SERIAL=y
CONFIG_UART_CONSOLE=y
CONFIG_UART_INTERRUPT_DRIVEN=y
CONFIG_UART_LINE_CTRL=y
CONFIG_UART_SHELL_ON_DEV_NAME="UART_0"
```

## 5. 网络配置

### 5.1 TCP/IP配置
```plaintext
# 网络基础配置
CONFIG_NETWORKING=y
CONFIG_NET_IPV4=y
CONFIG_NET_IPV6=y
CONFIG_NET_TCP=y
CONFIG_NET_UDP=y

# 网络缓冲区配置
CONFIG_NET_BUF_RX_COUNT=64
CONFIG_NET_BUF_TX_COUNT=64
CONFIG_NET_BUF_DATA_SIZE=128

# DHCP配置
CONFIG_NET_DHCPV4=y
CONFIG_NET_DHCPV4_INITIAL_RETRY_MAX=10

# DNS配置
CONFIG_DNS_RESOLVER=y
CONFIG_DNS_SERVER_IP_ADDRESSES=y
CONFIG_DNS_SERVER1="8.8.8.8"
```

### 5.2 蓝牙配置
```plaintext
# 蓝牙基础配置
CONFIG_BT=y
CONFIG_BT_PERIPHERAL=y
CONFIG_BT_CENTRAL=y
CONFIG_BT_DEBUG_LOG=y

# GATT配置
CONFIG_BT_GATT_DYNAMIC_DB=y
CONFIG_BT_GATT_CLIENT=y

# 安全配置
CONFIG_BT_SMP=y
CONFIG_BT_PRIVACY=y
CONFIG_BT_SIGNING=y
```

## 6. 文件系统配置

### 6.1 文件系统配置
```plaintext
# 文件系统支持
CONFIG_FILE_SYSTEM=y
CONFIG_FILE_SYSTEM_LITTLEFS=y

# FAT文件系统支持
CONFIG_FAT_FILESYSTEM_ELM=y
CONFIG_FS_FATFS_NUM_FILES=4
CONFIG_FS_FATFS_NUM_DIRS=4

# Flash驱动配置
CONFIG_FLASH=y
CONFIG_FLASH_MAP=y
CONFIG_FLASH_PAGE_LAYOUT=y
```

### 6.2 存储分区配置
```dts
/ {
    fstab {
        compatible = "zephyr,fstab";
        lfs1: lfs1 {
            compatible = "zephyr,fstab,littlefs";
            mount-point = "/lfs1";
            partition = <&lfs1_part>;
            read-size = <16>;
            prog-size = <16>;
            cache-size = <64>;
            lookahead-size = <32>;
            block-cycles = <512>;
        };
    };
};

&flash0 {
    partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        lfs1_part: partition@70000 {
            label = "storage";
            reg = <0x70000 0x10000>;
        };
    };
};
```

## 7. 调试配置

### 7.1 日志配置
```plaintext
# 日志系统配置
CONFIG_LOG=y
CONFIG_LOG_DEFAULT_LEVEL=3
CONFIG_LOG_BACKEND_UART=y
CONFIG_LOG_BACKEND_SHOW_COLOR=y

# 内存调试
CONFIG_DEBUG_OPTIMIZATIONS=y
CONFIG_DEBUG_INFO=y
CONFIG_STACK_USAGE=y
CONFIG_STACK_SENTINEL=y
CONFIG_HEAP_MEMORY_INFO=y

# 线程监控
CONFIG_THREAD_MONITOR=y
CONFIG_THREAD_NAME=y
CONFIG_THREAD_STACK_INFO=y
```

### 7.2 Shell配置
```plaintext
# Shell配置
CONFIG_SHELL=y
CONFIG_SHELL_BACKEND_SERIAL=y
CONFIG_SHELL_PROMPT_UART="zephyr:~$ "

# Shell历史记录
CONFIG_SHELL_HISTORY=y
CONFIG_SHELL_HISTORY_BUFFER=512

# Shell命令
CONFIG_SHELL_CMD_BUFF_SIZE=256
CONFIG_SHELL_ARGC_MAX=12
CONFIG_SHELL_WILDCARD=y
```

## 8. 安全配置

### 8.1 加密配置
```plaintext
# 加密支持
CONFIG_CRYPTO=y
CONFIG_CRYPTO_MBEDTLS=y

# 随机数生成器
CONFIG_ENTROPY_GENERATOR=y
CONFIG_TEST_RANDOM_GENERATOR=n

# TLS配置
CONFIG_MBEDTLS=y
CONFIG_MBEDTLS_BUILTIN=y
CONFIG_MBEDTLS_SSL_MAX_CONTENT_LEN=1500
```

### 8.2 安全启动配置
```plaintext
# 安全启动
CONFIG_BOOTLOADER_MCUBOOT=y
CONFIG_MCUBOOT_SIGNATURE_KEY_FILE="bootloader/mcuboot/root-rsa-2048.pem"
CONFIG_MCUBOOT_EXTRA_IMGTOOL_ARGS="--version 1.0.0"

# 固件更新
CONFIG_MCUBOOT_IMG_MANAGER=y
CONFIG_MCUBOOT_SERIAL=y
CONFIG_BOOT_UPGRADE_ONLY=y
```