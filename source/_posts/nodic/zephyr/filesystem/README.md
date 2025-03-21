---
abbrlink: 32
title: Zephyr 文件系统指南
date: '2025-03-21 20:49:56'
---
# Zephyr 文件系统指南

## 1. 文件系统概述

Zephyr RTOS 支持多种文件系统，包括但不限于：

- LittleFS：适用于嵌入式系统的轻量级文件系统
- FAT：广泛兼容的文件系统
- NFFS：专为闪存设计的文件系统

## 2. 基础配置

### 2.1 文件系统配置 (prj.conf)

```plaintext
# 文件系统支持
CONFIG_FILE_SYSTEM=y

# LittleFS 支持
CONFIG_FILE_SYSTEM_LITTLEFS=y

# FAT 文件系统支持（可选）
CONFIG_FAT_FILESYSTEM_ELM=y

# Flash 驱动配置
CONFIG_FLASH=y
CONFIG_FLASH_MAP=y
CONFIG_FLASH_PAGE_LAYOUT=y

# 文件系统 Shell 命令（可选，用于调试）
CONFIG_FILE_SYSTEM_SHELL=y
```

### 2.2 分区配置 (boards/xxx.overlay)

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

## 3. 基本文件操作

### 3.1 文件读写

```c
#include <zephyr/kernel.h>
#include <zephyr/fs/fs.h>

void file_operations_example(void)
{
    struct fs_file_t file;
    char buf[128];
    ssize_t bytes_written, bytes_read;
    int ret;

    /* 初始化文件对象 */
    fs_file_t_init(&file);

    /* 写文件 */
    ret = fs_open(&file, "/lfs1/test.txt", FS_O_CREATE | FS_O_WRITE);
    if (ret < 0) {
        printk("Failed to open file for writing: %d\n", ret);
        return;
    }

    bytes_written = fs_write(&file, "Hello, Zephyr!", 14);
    if (bytes_written < 0) {
        printk("Failed to write file: %d\n", bytes_written);
        fs_close(&file);
        return;
    }

    fs_close(&file);

    /* 读文件 */
    ret = fs_open(&file, "/lfs1/test.txt", FS_O_READ);
    if (ret < 0) {
        printk("Failed to open file for reading: %d\n", ret);
        return;
    }

    bytes_read = fs_read(&file, buf, sizeof(buf));
    if (bytes_read < 0) {
        printk("Failed to read file: %d\n", bytes_read);
        fs_close(&file);
        return;
    }

    buf[bytes_read] = '\0';
    printk("Read from file: %s\n", buf);

    fs_close(&file);
}
```

### 3.2 目录操作

```c
#include <zephyr/kernel.h>
#include <zephyr/fs/fs.h>

void directory_operations_example(void)
{
    int ret;

    /* 创建目录 */
    ret = fs_mkdir("/lfs1/mydir");
    if (ret < 0) {
        printk("Failed to create directory: %d\n", ret);
        return;
    }

    /* 列出目录内容 */
    struct fs_dir_t dir;
    struct fs_dirent entry;

    fs_dir_t_init(&dir);
    ret = fs_opendir(&dir, "/lfs1");
    if (ret < 0) {
        printk("Failed to open directory: %d\n", ret);
        return;
    }

    while (1) {
        ret = fs_readdir(&dir, &entry);
        if (ret < 0) {
            printk("Failed to read directory: %d\n", ret);
            break;
        }

        if (entry.name[0] == 0) {
            break;
        }

        printk("Found file: %s\n", entry.name);
    }

    fs_closedir(&dir);
}
```

## 4. 高级文件系统操作

### 4.1 文件系统挂载

```c
#include <zephyr/kernel.h>
#include <zephyr/fs/fs.h>
#include <zephyr/fs/littlefs.h>

/* 定义文件系统配置 */
static struct fs_mount_t lfs_storage_mnt = {
    .type = FS_LITTLEFS,
    .mnt_point = "/lfs1",
};

void mount_filesystem(void)
{
    int ret;

    /* 挂载文件系统 */
    ret = fs_mount(&lfs_storage_mnt);
    if (ret < 0) {
        printk("Failed to mount filesystem: %d\n", ret);
        return;
    }

    printk("LittleFS mounted at %s\n", lfs_storage_mnt.mnt_point);
}
```

### 4.2 文件系统信息

```c
#include <zephyr/kernel.h>
#include <zephyr/fs/fs.h>

void filesystem_info(void)
{
    int ret;
    struct fs_statvfs stats;

    ret = fs_statvfs("/lfs1", &stats);
    if (ret < 0) {
        printk("Failed to get filesystem stats: %d\n", ret);
        return;
    }

    printk("Filesystem info for /lfs1:\n");
    printk("Total blocks: %lu\n", stats.f_blocks);
    printk("Free blocks: %lu\n", stats.f_bfree);
    printk("Block size: %lu\n", stats.f_bsize);
    printk("Max name length: %lu\n", stats.f_namemax);
}
```

## 5. 文件系统测试

### 5.1 基本功能测试

```c
#include <zephyr/kernel.h>
#include <zephyr/fs/fs.h>
#include <zephyr/ztest.h>

ZTEST(filesystem_tests, test_file_operations)
{
    struct fs_file_t file;
    char write_buf[] = "Test data";
    char read_buf[20];
    ssize_t bytes_written, bytes_read;
    int ret;

    fs_file_t_init(&file);

    /* 写测试 */
    ret = fs_open(&file, "/lfs1/test.txt", FS_O_CREATE | FS_O_WRITE);
    zassert_equal(ret, 0, "Failed to open file for writing");

    bytes_written = fs_write(&file, write_buf, sizeof(write_buf));
    zassert_equal(bytes_written, sizeof(write_buf), "Failed to write file");

    fs_close(&file);

    /* 读测试 */
    ret = fs_open(&file, "/lfs1/test.txt", FS_O_READ);
    zassert_equal(ret, 0, "Failed to open file for reading");

    bytes_read = fs_read(&file, read_buf, sizeof(read_buf));
    zassert_equal(bytes_read, sizeof(write_buf), "Failed to read file");

    zassert_mem_equal(write_buf, read_buf, sizeof(write_buf), "Read data does not match written data");

    fs_close(&file);
}

ZTEST_SUITE(filesystem_tests, NULL, NULL, NULL, NULL, NULL);
```

## 6. 最佳实践

1. 错误处理：始终检查文件操作的返回值，并适当处理错误。

2. 资源管理：使用完文件或目录后，务必关闭它们以释放资源。

3. 缓冲区管理：在读写操作中使用适当大小的缓冲区，避免缓冲区溢出。

4. 文件系统一致性：考虑使用日志或事务来确保文件系统操作的原子性，特别是在写入关键数据时。

5. 性能优化：对于频繁访问的小文件，考虑使用内存文件系统或缓存机制。

6. 安全性：在处理用户输入的文件路径时，进行适当的验证和清理，防止路径遍历攻击。

7. 电源管理：在进行文件操作时，确保系统不会意外进入低功耗模式。

8. 文件系统选择：根据应用需求（如读写性能、磨损均衡、掉电保护等）选择合适的文件系统。

9. 分区管理：合理规划存储空间，为不同用途（如配置、日志、用户数据）分配独立的分区。

10. 定期维护：实现文件系统检查和修复机制，以应对意外断电等情况导致的文件系统不一致。
```