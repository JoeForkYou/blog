---
abbrlink: 43
title: build_system
date: '2025-03-21 20:49:56'
---
# Zephyr 构建系统

本文档详细介绍了 Zephyr RTOS 的构建系统，包括 CMake 构建流程、West 命令工具、构建配置选项以及自定义构建方法。

## CMake 构建流程

Zephyr 使用 CMake 作为其主要构建系统，结合 Ninja 或 Make 作为构建工具。

### 构建系统架构

Zephyr 的构建系统由以下几个主要部分组成：

1. **CMake 脚本**：定义构建逻辑和目标
2. **Kconfig 系统**：管理配置选项
3. **设备树**：描述硬件配置
4. **West**：元构建工具，简化命令行操作

### 基本构建流程

Zephyr 应用程序的构建流程如下：

1. **配置阶段**
   - 处理 Kconfig 选项
   - 解析设备树源文件
   - 生成构建系统缓存

2. **生成阶段**
   - 生成构建规则
   - 创建构建目标

3. **构建阶段**
   - 编译源代码
   - 链接目标文件
   - 生成最终二进制文件

### CMake 脚本结构

Zephyr 项目的 CMake 脚本组织结构：

- **zephyr/CMakeLists.txt**：主 CMake 文件
- **zephyr/cmake/**：构建系统模块
- **zephyr/modules/*/CMakeLists.txt**：模块构建脚本
- **app/CMakeLists.txt**：应用程序 CMake 文件

### 应用程序 CMakeLists.txt

一个典型的 Zephyr 应用程序 CMakeLists.txt 文件：

```cmake
# 设置最低 CMake 版本
cmake_minimum_required(VERSION 3.20.0)

# 查找 Zephyr 包
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})

# 设置项目名称
project(my_zephyr_app)

# 添加应用程序源文件
target_sources(app PRIVATE src/main.c)

# 添加包含目录
target_include_directories(app PRIVATE include)
```

## West 命令工具

West 是 Zephyr 的元工具，用于管理多仓库项目和简化构建命令。

### West 基本命令

```bash
# 初始化 Zephyr 工作区
west init -m https://github.com/zephyrproject-rtos/zephyr --mr v2.9.1 zephyrproject

# 更新所有项目
west update

# 列出所有项目
west list

# 构建应用程序
west build -b <board> <app_directory>

# 烧录应用程序
west flash

# 运行应用程序
west debug

# 清理构建目录
west build -t clean
```

### West 构建命令

West 构建命令的完整语法：

```bash
west build [-h] [-b BOARD] [-d BUILD_DIR]
           [-t TARGET] [-p] [-c] [--cmake-only]
           [--sysbuild | --no-sysbuild]
           [--pristine | --pristine=always | --pristine=auto | --pristine=never]
           [--context CONTEXT] [--force]
           [source_dir] [-- [cmake_opt [cmake_opt ...]]]
```

主要选项：

- **-b, --board BOARD**：指定目标板子
- **-d, --build-dir BUILD_DIR**：指定构建目录
- **-t, --target TARGET**：指定构建目标
- **-p, --pristine**：清理构建目录
- **-c, --cmake**：重新运行 CMake
- **--sysbuild**：使用系统构建
- **-- [cmake_opt ...]**：传递额外的 CMake 选项

### West 配置

West 配置存储在 `.west/config` 文件中，可以使用以下命令进行管理：

```bash
# 设置配置选项
west config build.board nrf52840dk_nrf52840

# 获取配置选项
west config build.board

# 删除配置选项
west config --delete build.board
```

常用配置选项：

- **build.board**：默认板子
- **build.dir**：默认构建目录
- **build.generator**：CMake 生成器（Ninja 或 Make）
- **build.pristine**：构建前是否清理

## 构建配置选项

Zephyr 提供了多种方式配置构建过程：

### Kconfig 配置

Kconfig 用于配置软件选项：

```
# prj.conf
CONFIG_GPIO=y
CONFIG_SERIAL=y
CONFIG_UART_INTERRUPT_DRIVEN=y
```

可以通过以下方式指定 Kconfig 配置文件：

```bash
# 使用默认的 prj.conf
west build -b nrf52840dk_nrf52840 app

# 指定配置文件
west build -b nrf52840dk_nrf52840 app -- -DCONF_FILE=my_prj.conf

# 合并多个配置文件
west build -b nrf52840dk_nrf52840 app -- -DCONF_FILE="prj.conf;overlay.conf"
```

### 设备树配置

设备树用于配置硬件：

```dts
/* app.overlay */
/ {
    leds {
        compatible = "gpio-leds";
        led0: led_0 {
            gpios = <&gpio0 13 GPIO_ACTIVE_LOW>;
            label = "Green LED 0";
        };
    };
};
```

可以通过以下方式指定设备树覆盖文件：

```bash
# 使用默认的 app.overlay
west build -b nrf52840dk_nrf52840 app

# 指定覆盖文件
west build -b nrf52840dk_nrf52840 app -- -DDTC_OVERLAY_FILE=my_overlay.dts

# 合并多个覆盖文件
west build -b nrf52840dk_nrf52840 app -- -DDTC_OVERLAY_FILE="app.overlay;extra.overlay"
```

### CMake 变量

可以通过 CMake 变量配置构建过程：

```bash
# 设置构建类型
west build -b nrf52840dk_nrf52840 app -- -DCMAKE_BUILD_TYPE=Debug

# 启用覆盖率分析
west build -b nrf52840dk_nrf52840 app -- -DCMAKE_BUILD_TYPE=Debug -DCONFIG_COVERAGE=y

# 启用编译器优化
west build -b nrf52840dk_nrf52840 app -- -DEXTRA_CFLAGS="-O2 -fomit-frame-pointer"
```

### 板级配置

每个板子可以有特定的配置文件：

- **boards/<arch>/<board>/<board>_defconfig**：默认板级 Kconfig 配置
- **boards/<arch>/<board>/<board>.dts**：默认板级设备树

可以创建板级特定的应用程序配置：

```
app/
├── boards/
│   ├── nrf52840dk_nrf52840.conf     # 板级特定 Kconfig
│   └── nrf52840dk_nrf52840.overlay  # 板级特定设备树
├── prj.conf                         # 通用 Kconfig
└── app.overlay                      # 通用设备树
```

## 自定义构建

### 自定义构建目标

可以在应用程序的 CMakeLists.txt 中定义自定义目标：

```cmake
# 添加自定义目标
add_custom_target(my_target
    COMMAND ${CMAKE_COMMAND} -E echo "Running custom target"
    COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/scripts/my_script.py
)
```

使用自定义目标：

```bash
west build -t my_target
```

### 自定义构建脚本

可以创建自定义 CMake 模块：

```cmake
# cmake/custom_module.cmake
function(my_custom_function)
    # 实现自定义功能
endfunction()
```

在应用程序的 CMakeLists.txt 中使用自定义模块：

```cmake
# 包含自定义模块
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)
include(custom_module)

# 调用自定义函数
my_custom_function()
```

### 条件构建

可以使用 CMake 条件语句根据配置选项控制构建：

```cmake
# 根据 Kconfig 选项控制源文件
if(CONFIG_FEATURE_X)
    target_sources(app PRIVATE src/feature_x.c)
endif()

# 根据板子类型控制包含目录
if(${BOARD} MATCHES "nrf52.*")
    target_include_directories(app PRIVATE include/nrf52)
elseif(${BOARD} MATCHES "stm32.*")
    target_include_directories(app PRIVATE include/stm32)
endif()
```

### 多应用程序构建

Zephyr 支持使用 sysbuild 构建多个应用程序：

```bash
# 启用 sysbuild
west build -b nrf52840dk_nrf52840 --sysbuild app
```

sysbuild 配置示例 (sysbuild.cmake)：

```cmake
set(SB_CONFIG_BOOTLOADER_MCUBOOT y)
set(SB_CONFIG_MCUBOOT_BUILD_STRATEGY_FROM_SOURCE y)

add_child_image(
    NAME mcuboot
    SOURCE_DIR ${ZEPHYR_BASE}/../bootloader/mcuboot/boot/zephyr
)

add_child_image(
    NAME app
    SOURCE_DIR ${APP_DIR}
)
```

## 构建缓存和优化

### 使用 ccache

启用 ccache 加速重复构建：

```bash
# 在 prj.conf 中启用
CONFIG_CCACHE=y

# 或在构建命令中启用
west build -b nrf52840dk_nrf52840 app -- -DCCACHE=1
```

### 增量构建

默认情况下，Zephyr 使用增量构建。可以控制构建清理级别：

```bash
# 完全清理构建目录
west build --pristine=always

# 智能清理（仅在必要时清理）
west build --pristine=auto

# 从不清理
west build --pristine=never
```

### 并行构建

配置并行构建任务数：

```bash
# 在 .west/config 中设置
west config build.jobs 8

# 或在环境变量中设置
export ZEPHYR_JOBS=8
```

## 构建输出

构建过程生成的主要文件：

- **zephyr.elf**：ELF 格式的可执行文件
- **zephyr.bin**：二进制格式的固件
- **zephyr.hex**：Intel HEX 格式的固件
- **zephyr.map**：链接映射文件
- **zephyr.lst**：反汇编列表
- **zephyr.stat**：内存使用统计

这些文件位于构建目录的 `zephyr` 子目录中：

```bash
build/zephyr/zephyr.hex
```

## 常见问题

### 1. 构建错误

**问题**：构建过程中出现错误

**解决方案**：
- 尝试使用 `--pristine` 选项重新构建
- 检查配置文件和源代码错误
- 验证工具链和依赖是否正确安装

### 2. 配置问题

**问题**：配置选项不生效

**解决方案**：
- 检查配置文件路径是否正确
- 确认配置选项名称和值是否正确
- 使用 `west build -t menuconfig` 验证配置

### 3. 设备树问题

**问题**：设备树覆盖不生效

**解决方案**：
- 检查设备树语法是否正确
- 确认覆盖文件路径是否正确
- 使用 `west build -t devicetree` 检查处理后的设备树

### 4. 内存不足

**问题**：链接阶段报告内存不足

**解决方案**：
- 优化代码减少内存使用
- 调整链接脚本增加可用内存
- 禁用不必要的功能和模块

### 5. 构建性能问题

**问题**：构建过程缓慢

**解决方案**：
- 启用 ccache
- 增加并行任务数
- 使用增量构建
- 使用更快的存储设备

## 总结

Zephyr 的构建系统基于 CMake，提供了强大而灵活的配置和构建能力。通过 West 工具，可以简化构建命令和工作流程。Kconfig 和设备树系统提供了软件和硬件配置的分离，使得应用程序更加可移植。了解这些构建系统组件和工作流程对于高效开发 Zephyr 应用程序至关重要。