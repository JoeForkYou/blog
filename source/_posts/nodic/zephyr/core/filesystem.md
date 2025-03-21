---
abbrlink: 3
title: filesystem
date: '2025-03-21 20:49:56'
---
# Zephyr 文件系统

Zephyr RTOS 提供了多种文件系统支持，可以用于数据存储和管理。本文档将详细介绍 Zephyr 文件系统的架构和使用方法。

## 文件系统概述

### 支持的文件系统

Zephyr 支持多种文件系统：

1. **FAT 文件系统**
   - FAT12/16/32
   - 基于 FatFs 库实现
   - 适用于 SD 卡、USB 存储等

2. **LittleFS**
   - 为闪存设计的轻量级文件系统
   - 支持掉电保护
   - 磨损均衡

3. **NFFS (Newtron Flash File System)**
   - 为闪存设计的文件系统
   - 支持磨损均衡

4. **NVS (Non-Volatile Storage)**
   - 简单的键值存储
   - 支持掉电保护
   - 适用于配置数据

### 文件系统架构

Zephyr 文件系统架构包括：

1. **VFS 层**：提供统一的文件系统 API
2. **文件系统实现**：特定文件系统的实现
3. **存储层**：与底层存储设备交互

## 文件系统配置

### 启用文件系统

在 `prj.conf` 中启用文件系统功能：

```
# 启用文件系统
CONFIG_FILE_SYSTEM=y

# 启用特定文件系统
CONFIG_FAT_FILESYSTEM_ELM=y  # FAT 文件系统
CONFIG_FILE_SYSTEM_LITTLEFS=y  # LittleFS
CONFIG_FILE_SYSTEM_NFFS=y  # NFFS
CONFIG_NVS=y  # NVS

# 文件系统缓冲区配置
CONFIG_FS_BUFFER_SIZE=64
```

### 挂载文件系统

```c
#include <zephyr/fs/fs.h>
#include <zephyr/fs/littlefs.h>

// LittleFS 配置
FS_LITTLEFS_DECLARE_DEFAULT_CONFIG(lfs_data);

// 文件系统挂载点
static struct fs_mount_t lfs_mount = {
    .type = FS_LITTLEFS,
    .fs_data = &lfs_data,
    .storage_dev = (void *)FLASH_AREA_ID(storage),
    .mnt_point = "/lfs",
};

// 挂载文件系统
int mount_filesystem(void)
{
    int ret;

    ret = fs_mount(&lfs_mount);
    if (ret < 0) {
        printk("Error mounting LittleFS [%d]\n", ret);
        return ret;
    }

    printk("LittleFS mounted at %s\n", lfs_mount.mnt_point);
    return 0;
}
```

## 文件操作

### 文件读写

```c
#include <zephyr/fs/fs.h>

// 写入文件
void write_file(void)
{
    struct fs_file_t file;
    int ret;

    // 打开文件
    ret = fs_open(&file, "/lfs/data.txt", FS_O_CREATE | FS_O_WRITE);
    if (ret < 0) {
        printk("Error opening file [%d]\n", ret);
        return;
    }

    // 写入数据
    const char *data = "Hello, Zephyr File System!";
    ret = fs_write(&file, data, strlen(data));
    if (ret < 0) {
        printk("Error writing file [%d]\n", ret);
    } else {
        printk("Wrote %d bytes to file\n", ret);
    }

    // 关闭文件
    fs_close(&file);
}

// 读取文件
void read_file(void)
{
    struct fs_file_t file;
    int ret;

    // 打开文件
    ret = fs_open(&file, "/lfs/data.txt", FS_O_READ);
    if (ret < 0) {
        printk("Error opening file [%d]\n", ret);
        return;
    }

    // 读取数据
    char buffer[64];
    ret = fs_read(&file, buffer, sizeof(buffer) - 1);
    if (ret < 0) {
        printk("Error reading file [%d]\n", ret);
    } else {
        buffer[ret] = '\0';
        printk("Read %d bytes: %s\n", ret, buffer);
    }

    // 关闭文件
    fs_close(&file);
}
```

### 目录操作

```c
#include <zephyr/fs/fs.h>

// 创建目录
void create_directory(void)
{
    int ret;

    ret = fs_mkdir("/lfs/mydir");
    if (ret < 0) {
        printk("Error creating directory [%d]\n", ret);
    } else {
        printk("Directory created\n");
    }
}

// 列出目录内容
void list_directory(void)
{
    struct fs_dir_t dir;
    int ret;

    // 打开目录
    ret = fs_opendir(&dir, "/lfs");
    if (ret < 0) {
        printk("Error opening directory [%d]\n", ret);
        return;
    }

    // 读取目录项
    while (1) {
        struct fs_dirent entry;
        
        ret = fs_readdir(&dir, &entry);
        if (ret < 0) {
            printk("Error reading directory [%d]\n", ret);
            break;
        }
        
        // 到达目录末尾
        if (entry.name[0] == '\0') {
            break;
        }
        
        // 打印文件信息
        printk("  %s [%s] %zu bytes\n", entry.name,
               (entry.type == FS_DIR_ENTRY_FILE) ? "FILE" : "DIR",
               entry.size);
    }

    // 关闭目录
    fs_closedir(&dir);
}
```

### 文件和目录管理

```c
#include <zephyr/fs/fs.h>

// 检查文件是否存在
void check_file(void)
{
    struct fs_dirent entry;
    int ret;

    ret = fs_stat("/lfs/data.txt", &entry);
    if (ret == 0) {
        printk("File exists, size: %zu bytes\n", entry.size);
    } else {
        printk("File does not exist [%d]\n", ret);
    }
}

// 重命名文件
void rename_file(void)
{
    int ret;

    ret = fs_rename("/lfs/data.txt", "/lfs/newdata.txt");
    if (ret < 0) {
        printk("Error renaming file [%d]\n", ret);
    } else {
        printk("File renamed\n");
    }
}

// 删除文件
void delete_file(void)
{
    int ret;

    ret = fs_unlink("/lfs/newdata.txt");
    if (ret < 0) {
        printk("Error deleting file [%d]\n", ret);
    } else {
        printk("File deleted\n");
    }
}
```

## 特定文件系统

### FAT 文件系统

```c
#include <zephyr/fs/fs.h>
#include <zephyr/storage/disk_access.h>

// FAT 文件系统挂载
void mount_fat(void)
{
    static const char *disk_mount_pt = "/SD:";
    static const char *disk_pdrv = "SD";

    // 挂载配置
    static struct fs_mount_t mp = {
        .type = FS_FATFS,
        .mnt_point = disk_mount_pt,
        .fs_data = NULL,
    };

    // 检查磁盘是否就绪
    if (disk_access_init(disk_pdrv) != 0) {
        printk("Disk access initialization failed\n");
        return;
    }

    // 挂载文件系统
    mp.storage_dev = (void *)disk_pdrv;
    int ret = fs_mount(&mp);
    if (ret != 0) {
        printk("Error mounting FAT [%d]\n", ret);
        return;
    }

    printk("FAT file system mounted\n");
}
```

### LittleFS

```c
#include <zephyr/fs/fs.h>
#include <zephyr/fs/littlefs.h>
#include <zephyr/storage/flash_map.h>

// LittleFS 配置
FS_LITTLEFS_DECLARE_DEFAULT_CONFIG(lfs_data);

// 挂载 LittleFS
void mount_littlefs(void)
{
    // 挂载配置
    static struct fs_mount_t mp = {
        .type = FS_LITTLEFS,
        .fs_data = &lfs_data,
        .storage_dev = (void *)FLASH_AREA_ID(lfs_storage),
        .mnt_point = "/lfs",
    };

    // 挂载文件系统
    int ret = fs_mount(&mp);
    if (ret != 0) {
        printk("Error mounting LittleFS [%d]\n", ret);
        return;
    }

    printk("LittleFS mounted\n");
}
```

### NVS (Non-Volatile Storage)

```c
#include <zephyr/fs/nvs.h>
#include <zephyr/storage/flash_map.h>

// NVS 实例
static struct nvs_fs fs = {
    .sector_size = 4096,  // 扇区大小
    .sector_count = 4,    // 扇区数量
    .offset = FLASH_AREA_OFFSET(nvs_storage),  // 闪存区域偏移
};

// 初始化 NVS
void init_nvs(void)
{
    int ret;

    // 初始化 NVS
    ret = nvs_init(&fs, DT_CHOSEN_ZEPHYR_FLASH_CONTROLLER_LABEL);
    if (ret < 0) {
        printk("Error initializing NVS [%d]\n", ret);
        return;
    }

    printk("NVS initialized\n");
}

// 使用 NVS
void use_nvs(void)
{
    int ret;
    
    // 写入数据
    uint32_t value = 12345;
    ret = nvs_write(&fs, 1, &value, sizeof(value));
    if (ret < 0) {
        printk("Error writing to NVS [%d]\n", ret);
    } else {
        printk("Data written to NVS\n");
    }
    
    // 读取数据
    uint32_t read_value;
    ret = nvs_read(&fs, 1, &read_value, sizeof(read_value));
    if (ret < 0) {
        printk("Error reading from NVS [%d]\n", ret);
    } else {
        printk("Read value: %u\n", read_value);
    }
    
    // 删除数据
    ret = nvs_delete(&fs, 1);
    if (ret < 0) {
        printk("Error deleting from NVS [%d]\n", ret);
    } else {
        printk("Data deleted from NVS\n");
    }
}
```

## 存储分区

### 分区配置

在设备树中配置存储分区：

```dts
/ {
    chosen {
        zephyr,code-partition = &slot0_partition;
    };

    flash_partitions {
        compatible = "fixed-partitions";
        #address-cells = <1>;
        #size-cells = <1>;

        boot_partition: partition@0 {
            label = "mcuboot";
            reg = <0x00000000 0x10000>;
        };
        slot0_partition: partition@10000 {
            label = "image-0";
            reg = <0x00010000 0x40000>;
        };
        slot1_partition: partition@50000 {
            label = "image-1";
            reg = <0x00050000 0x40000>;
        };
        storage_partition: partition@90000 {
            label = "storage";
            reg = <0x00090000 0x10000>;
        };
    };
};
```

### 访问分区

```c
#include <zephyr/storage/flash_map.h>

// 获取分区信息
void access_partition(void)
{
    const struct flash_area *fa;
    int ret;

    // 打开分区
    ret = flash_area_open(FLASH_AREA_ID(storage), &fa);
    if (ret < 0) {
        printk("Error opening flash area [%d]\n", ret);
        return;
    }

    // 擦除分区
    ret = flash_area_erase(fa, 0, fa->fa_size);
    if (ret < 0) {
        printk("Error erasing flash area [%d]\n", ret);
        goto out;
    }

    // 写入数据
    uint8_t data[] = {0x01, 0x02, 0x03, 0x04};
    ret = flash_area_write(fa, 0, data, sizeof(data));
    if (ret < 0) {
        printk("Error writing to flash area [%d]\n", ret);
        goto out;
    }

    // 读取数据
    uint8_t read_data[4];
    ret = flash_area_read(fa, 0, read_data, sizeof(read_data));
    if (ret < 0) {
        printk("Error reading from flash area [%d]\n", ret);
        goto out;
    }

    printk("Read data: %02x %02x %02x %02x\n",
           read_data[0], read_data[1], read_data[2], read_data[3]);

out:
    // 关闭分区
    flash_area_close(fa);
}
```

## 文件系统 Shell 命令

Zephyr 提供了文件系统 Shell 命令，可以用于交互式文件操作：

```
# 在 prj.conf 中启用文件系统 Shell
CONFIG_FILE_SYSTEM_SHELL=y
```

可用命令：

```
fs cd <path>                  # 更改当前目录
fs ls [path]                  # 列出目录内容
fs mkdir <path>               # 创建目录
fs read <path> [offset] [len] # 读取文件
fs write <path> <str>         # 写入文件
fs rm <path>                  # 删除文件
fs mount <path>               # 挂载文件系统
fs unmount <path>             # 卸载文件系统
```

## 最佳实践

1. **文件系统选择**
   - 对于小型闪存，使用 LittleFS
   - 对于 SD 卡等外部存储，使用 FAT
   - 对于简单键值存储，使用 NVS

2. **性能优化**
   - 适当配置缓冲区大小
   - 批量读写操作
   - 避免频繁打开关闭文件

3. **可靠性保证**
   - 使用支持掉电保护的文件系统
   - 定期同步文件系统
   - 实现错误恢复机制

4. **资源管理**
   - 及时关闭文件和目录
   - 避免过度分配文件句柄
   - 定期清理临时文件

## 常见问题

1. **挂载失败**
   - 检查存储设备是否正常
   - 验证分区配置
   - 确认文件系统类型

2. **读写错误**
   - 检查文件权限
   - 验证路径是否正确
   - 确认存储空间是否充足

3. **性能问题**
   - 增加缓冲区大小
   - 减少小块读写操作
   - 使用更适合的文件系统

4. **文件系统损坏**
   - 使用文件系统检查工具
   - 实现自动修复机制
   - 备份重要数据

## 总结

Zephyr 文件系统提供了丰富的功能，支持多种文件系统类型和存储设备。通过合理配置和使用这些功能，可以开发出高效、可靠的数据存储应用。深入理解这些文件系统接口对于开发高质量的 Zephyr 应用至关重要。