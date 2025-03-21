---
abbrlink: 18
title: porting
date: '2025-03-21 20:49:56'
---
# Zephyr 移植指南

本文档详细介绍了如何将 Zephyr RTOS 移植到新的硬件平台，包括板级支持包开发、SOC 适配、驱动移植和启动配置等内容。

## 板级支持包

### 基本结构

1. **目录结构**
```
boards/
└── arm/
    └── my_board/
        ├── board.cmake
        ├── CMakeLists.txt
        ├── Kconfig.board
        ├── Kconfig.defconfig
        ├── my_board_defconfig
        ├── my_board.dts
        └── my_board.yaml
```

2. **配置文件**
```cmake
# board.cmake
board_runner_args(jlink "--device=nrf52" "--speed=4000")
include(${ZEPHYR_BASE}/boards/common/jlink.board.cmake)

# CMakeLists.txt
if(CONFIG_PINMUX)
  zephyr_library()
  zephyr_library_sources(pinmux.c)
endif()
```

### 板级定义

1. **Kconfig 配置**
```
# Kconfig.board
config BOARD_MY_BOARD
    bool "My Custom Board"
    depends on SOC_NRF52840_QIAA

# Kconfig.defconfig
if BOARD_MY_BOARD

config BOARD
    default "my_board"

config GPIO_AS_PINMUX
    default y

endif # BOARD_MY_BOARD
```

2. **设备树文件**
```dts
/* my_board.dts */
/dts-v1/;
#include <nordic/nrf52840_qiaa.dtsi>

/ {
    model = "My Custom Board";
    compatible = "vendor,my-board";

    chosen {
        zephyr,console = &uart0;
        zephyr,shell-uart = &uart0;
        zephyr,uart-mcumgr = &uart0;
        zephyr,bt-mon-uart = &uart0;
        zephyr,bt-c2h-uart = &uart0;
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
            gpios = <&gpio0 11 (GPIO_PULL_UP | GPIO_ACTIVE_LOW)>;
            label = "Push button switch 0";
        };
    };
};

&uart0 {
    status = "okay";
    current-speed = <115200>;
    tx-pin = <6>;
    rx-pin = <8>;
};

&i2c0 {
    status = "okay";
    sda-pin = <26>;
    scl-pin = <27>;
};

&spi0 {
    status = "okay";
    sck-pin = <27>;
    mosi-pin = <26>;
    miso-pin = <29>;
};
```

## SOC 适配

### SOC 支持

1. **SOC 定义**
```
# soc/arm/vendor/my_soc/Kconfig.soc
config SOC_SERIES_MY_SOC
    bool "My SOC Series"
    select ARM
    select CPU_CORTEX_M4
    select CPU_HAS_FPU
    help
      Enable support for My SOC Series

config SOC_MY_SOC
    bool "My SOC"
    select SOC_SERIES_MY_SOC
    help
      My SOC
```

2. **SOC 配置**
```cmake
# soc/arm/vendor/my_soc/CMakeLists.txt
zephyr_library()
zephyr_library_sources(
    soc.c
    power.c
    )
zephyr_library_include_directories(.)
```

### 时钟配置

1. **时钟初始化**
```c
/* soc/arm/vendor/my_soc/soc.c */
void sys_clock_init(void)
{
    /* 配置系统时钟 */
    NRF_CLOCK->EVENTS_HFCLKSTARTED = 0;
    NRF_CLOCK->TASKS_HFCLKSTART = 1;
    while (NRF_CLOCK->EVENTS_HFCLKSTARTED == 0) {
        /* 等待时钟稳定 */
    }

    /* 配置 LFCLK */
    NRF_CLOCK->LFCLKSRC = CLOCK_LFCLKSRC_SRC_Xtal;
    NRF_CLOCK->TASKS_LFCLKSTART = 1;
}
```

2. **电源管理**
```c
/* soc/arm/vendor/my_soc/power.c */
void sys_set_power_state(enum power_states state)
{
    switch (state) {
    case POWER_STATE_SLEEP:
        /* 进入睡眠模式 */
        break;
    case POWER_STATE_DEEP_SLEEP:
        /* 进入深度睡眠模式 */
        break;
    default:
        break;
    }
}
```

## 驱动移植

### GPIO 驱动

1. **驱动结构**
```c
/* drivers/gpio/gpio_my_soc.c */
struct gpio_my_soc_config {
    /* 设备配置 */
    uint32_t port;
    uint32_t base_addr;
    uint32_t irq_num;
};

struct gpio_my_soc_data {
    /* 驱动数据 */
    struct gpio_driver_config config;
    struct k_sem lock;
};
```

2. **驱动实现**
```c
/* GPIO 初始化 */
static int gpio_my_soc_init(const struct device *dev)
{
    const struct gpio_my_soc_config *config = dev->config;
    struct gpio_my_soc_data *data = dev->data;

    /* 初始化硬件 */
    sys_write32(0xFFFFFFFF, config->base_addr + GPIO_OUTCLR);

    /* 初始化同步原语 */
    k_sem_init(&data->lock, 1, 1);

    return 0;
}

/* GPIO 配置 */
static int gpio_my_soc_configure(const struct device *dev,
                               gpio_pin_t pin,
                               gpio_flags_t flags)
{
    const struct gpio_my_soc_config *config = dev->config;
    struct gpio_my_soc_data *data = dev->data;
    uint32_t reg;

    if (pin >= 32) {
        return -EINVAL;
    }

    k_sem_take(&data->lock, K_FOREVER);

    /* 配置 GPIO */
    reg = sys_read32(config->base_addr + GPIO_CNF(pin));
    
    if (flags & GPIO_OUTPUT) {
        reg |= GPIO_CNF_DIR_OUTPUT;
    } else {
        reg &= ~GPIO_CNF_DIR_OUTPUT;
    }

    if (flags & GPIO_PULL_UP) {
        reg |= GPIO_CNF_PULL_UP;
    } else if (flags & GPIO_PULL_DOWN) {
        reg |= GPIO_CNF_PULL_DOWN;
    }

    sys_write32(reg, config->base_addr + GPIO_CNF(pin));

    k_sem_give(&data->lock);
    return 0;
}
```

### UART 驱动

1. **驱动结构**
```c
/* drivers/serial/uart_my_soc.c */
struct uart_my_soc_config {
    uint32_t base_addr;
    uint32_t clock_freq;
    uint8_t tx_pin;
    uint8_t rx_pin;
};

struct uart_my_soc_data {
    struct uart_config config;
    struct k_sem tx_sem;
    struct k_sem rx_sem;
};
```

2. **驱动实现**
```c
/* UART 初始化 */
static int uart_my_soc_init(const struct device *dev)
{
    const struct uart_my_soc_config *config = dev->config;
    struct uart_my_soc_data *data = dev->data;

    /* 配置引脚 */
    configure_pins(config);

    /* 初始化硬件 */
    sys_write32(UART_ENABLE, config->base_addr + UART_CONFIG);

    /* 配置波特率 */
    set_baudrate(dev, data->config.baudrate);

    /* 初始化信号量 */
    k_sem_init(&data->tx_sem, 1, 1);
    k_sem_init(&data->rx_sem, 0, 1);

    /* 使能中断 */
    sys_write32(UART_INT_RXDRDY | UART_INT_TXDRDY,
                config->base_addr + UART_INTENSET);

    return 0;
}

/* UART 发送 */
static int uart_my_soc_poll_out(const struct device *dev,
                               unsigned char c)
{
    const struct uart_my_soc_config *config = dev->config;

    /* 等待发送完成 */
    while (!(sys_read32(config->base_addr + UART_STATUS) &
             UART_STATUS_TXDRDY)) {
    }

    /* 发送数据 */
    sys_write32(c, config->base_addr + UART_TXD);

    return 0;
}
```

## 启动配置

### 启动文件

1. **向量表**
```c
/* soc/arm/vendor/my_soc/vector_table.h */
typedef void (*vector_t)(void);
struct vector_table {
    void *stack;
    vector_t reset;
    vector_t nmi;
    vector_t hard_fault;
    vector_t mpu_fault;
    vector_t bus_fault;
    /* ... */
};
```

2. **启动代码**
```c
/* soc/arm/vendor/my_soc/startup.c */
void z_arm_reset(void)
{
    /* 初始化系统 */
    SystemInit();

    /* 初始化 BSS */
    z_bss_zero();

    /* 初始化数据段 */
    z_data_copy();

    /* 调用主函数 */
    z_arm_start();
}
```

### 链接脚本

1. **内存布局**
```
/* soc/arm/vendor/my_soc/linker.ld */
MEMORY
{
    FLASH (rx) : ORIGIN = 0x00000000, LENGTH = 1M
    RAM (rwx) : ORIGIN = 0x20000000, LENGTH = 256K
}

SECTIONS
{
    .text :
    {
        __text_start = .;
        *(.text*)
        __text_end = .;
    } > FLASH

    .rodata :
    {
        __rodata_start = .;
        *(.rodata*)
        __rodata_end = .;
    } > FLASH

    .data :
    {
        __data_start = .;
        *(.data*)
        __data_end = .;
    } > RAM AT > FLASH

    .bss :
    {
        __bss_start = .;
        *(.bss*)
        __bss_end = .;
    } > RAM
}
```

### 系统初始化

1. **时钟配置**
```c
/* soc/arm/vendor/my_soc/soc_init.c */
void SystemInit(void)
{
    /* 配置时钟 */
    configure_system_clock();

    /* 配置中断 */
    NVIC_SetPriority(PendSV_IRQn, _EXC_PENDSV_PRIO);
    NVIC_SetPriority(SysTick_IRQn, _EXC_SYSTICK_PRIO);
}
```

2. **中断配置**
```c
/* soc/arm/vendor/my_soc/irq_init.c */
void z_arm_irq_init(void)
{
    /* 配置中断向量表 */
    SCB->VTOR = (uint32_t)_vector_table;

    /* 配置中断优先级组 */
    NVIC_SetPriorityGrouping(0);

    /* 使能中断 */
    __enable_irq();
}
```

## 调试支持

### JTAG/SWD

1. **调试接口配置**
```
# boards/arm/my_board/board.cmake
board_runner_args(jlink "--device=my_soc" "--speed=4000")
include(${ZEPHYR_BASE}/boards/common/jlink.board.cmake)
```

2. **OpenOCD 配置**
```tcl
# boards/arm/my_board/support/openocd.cfg
source [find interface/jlink.cfg]
transport select swd
source [find target/my_soc.cfg]

adapter_khz 4000

$_TARGETNAME configure -event gdb-attach {
    echo "Debugger attaching..."
    reset init
}
```

### 串口调试

1. **串口配置**
```c
/* 串口初始化 */
void board_init_debug_console(void)
{
    const struct device *dev;

    dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));
    if (!device_is_ready(dev)) {
        return;
    }

    uart_configure(dev, &uart_config);
}
```

2. **调试输出**
```c
/* 调试输出函数 */
int arch_printk_char_out(int c)
{
    const struct device *dev;

    dev = DEVICE_DT_GET(DT_CHOSEN(zephyr_console));
    if (!device_is_ready(dev)) {
        return -ENODEV;
    }

    uart_poll_out(dev, c);
    return 0;
}
```

## 最佳实践

### 1. 移植策略

- 分步骤移植
- 验证每个步骤
- 保持代码清晰
- 文档完善

### 2. 硬件抽象

- 使用设备树
- 驱动抽象
- 配置灵活
- 接口统一

### 3. 调试支持

- 完整调试接口
- 日志系统
- 错误处理
- 性能监控

### 4. 文档维护

- 硬件文档
- 移植指南
- API 文档
- 示例代码

## 常见问题

### 1. 启动问题

**问题**：系统无法启动

**解决方案**：
- 检查时钟配置
- 验证启动代码
- 检查链接脚本
- 调试复位向量

### 2. 驱动问题

**问题**：驱动无法工作

**解决方案**：
- 检查硬件配置
- 验证驱动代码
- 测试中断处理
- 检查时序要求

### 3. 内存问题

**问题**：内存访问错误

**解决方案**：
- 检查内存映射
- 验证栈配置
- 检查对齐要求
- 分析内存使用

### 4. 调试问题

**问题**：无法调试

**解决方案**：
- 检查调试接口
- 验证调试配置
- 测试串口通信
- 使用调试工具

## 总结

将 Zephyr RTOS 移植到新的硬件平台需要系统的方法和细致的工作。通过正确实施板级支持包开发、SOC 适配、驱动移植和启动配置，可以成功将 Zephyr 移植到新的硬件平台。本文档提供了详细的指导和实例，帮助开发者完成移植工作。