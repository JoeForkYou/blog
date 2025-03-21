---
title: Zephyr 音频子系统指南
tags: zephyr
categories: zephyr
abbrlink: 15158
date: 2025-03-20 23:45:22
mermaid: true
---

# Zephyr 音频子系统指南

## 版本信息
- 版本：V1.0
- 更新时间：2025年03月20日 23:45

## 1. 音频子系统概述

Zephyr RTOS 提供了完整的音频子系统，支持音频捕获、处理和播放功能。音频子系统基于模块化架构，包括音频编解码器驱动、音频控制器、音频流管理和音频处理框架。

### 1.1 基础配置 (prj.conf)
```plaintext
# 音频子系统支持
CONFIG_AUDIO=y
CONFIG_AUDIO_CODEC=y
CONFIG_AUDIO_DMIC=y
CONFIG_I2S=y

# 音频处理支持
CONFIG_AUDIO_PROCESSING=y
CONFIG_AUDIO_SAMPLE_RATE_CONVERTER=y

# 内存配置
CONFIG_HEAP_MEM_POOL_SIZE=32768
CONFIG_AUDIO_BUFFER_SIZE=4096
```

### 1.2 设备树配置
```dts
/ {
    aliases {
        dmic = &dmic0;
        codec = &codec0;
        i2s = &i2s0;
    };

    dmic0: dmic0 {
        compatible = "zephyr,dmic";
        status = "okay";
        clock-frequency = <3072000>;
        sample-rate = <48000>;
        channels = <2>;
    };

    codec0: codec0 {
        compatible = "zephyr,audio-codec";
        status = "okay";
        sample-rate = <48000>;
        channels = <2>;
    };

    i2s0: i2s0 {
        compatible = "zephyr,i2s";
        status = "okay";
        clock-frequency = <12288000>;
        sample-rate = <48000>;
        channels = <2>;
        word-size = <16>;
    };
};
```

## 2. 音频编解码器驱动

### 2.1 编解码器初始化
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/audio/codec.h>

void codec_example(void)
{
    const struct device *codec_dev = DEVICE_DT_GET(DT_NODELABEL(codec0));
    struct audio_codec_cfg codec_cfg;
    int ret;

    if (!device_is_ready(codec_dev)) {
        printk("Codec device not ready\n");
        return;
    }

    /* 配置编解码器 */
    codec_cfg.dai_type = AUDIO_DAI_TYPE_I2S;
    codec_cfg.dai_cfg.i2s = {
        .word_size = 16,
        .channels = 2,
        .format = I2S_FMT_DATA_FORMAT_I2S,
        .options = I2S_OPT_FRAME_CLK_MASTER | I2S_OPT_BIT_CLK_MASTER,
        .frame_clk_freq = 48000,
        .block_size = 512,
        .timeout = 1000,
    };

    /* 初始化编解码器 */
    ret = audio_codec_configure(codec_dev, &codec_cfg);
    if (ret != 0) {
        printk("Failed to configure codec: %d\n", ret);
        return;
    }

    /* 启动编解码器 */
    ret = audio_codec_start(codec_dev);
    if (ret != 0) {
        printk("Failed to start codec: %d\n", ret);
        return;
    }
}
```

### 2.2 编解码器控制
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/audio/codec.h>

void codec_control_example(void)
{
    const struct device *codec_dev = DEVICE_DT_GET(DT_NODELABEL(codec0));
    int ret;

    if (!device_is_ready(codec_dev)) {
        return;
    }

    /* 设置音量 */
    ret = audio_codec_set_property(codec_dev,
                                  AUDIO_PROPERTY_VOLUME,
                                  AUDIO_CHANNEL_ALL,
                                  80);
    if (ret != 0) {
        printk("Failed to set volume: %d\n", ret);
        return;
    }

    /* 设置静音 */
    ret = audio_codec_set_property(codec_dev,
                                  AUDIO_PROPERTY_MUTE,
                                  AUDIO_CHANNEL_ALL,
                                  1);
    if (ret != 0) {
        printk("Failed to set mute: %d\n", ret);
        return;
    }

    /* 设置增益 */
    ret = audio_codec_set_property(codec_dev,
                                  AUDIO_PROPERTY_GAIN,
                                  AUDIO_CHANNEL_LEFT,
                                  6);
    if (ret != 0) {
        printk("Failed to set gain: %d\n", ret);
        return;
    }
}
```

## 3. 音频数据流

### 3.1 I2S 配置
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/i2s.h>

#define SAMPLE_RATE   48000
#define SAMPLE_BIT_WIDTH 16
#define CHANNELS      2
#define BLOCK_SIZE    512
#define BLOCK_COUNT   4

static const struct i2s_config i2s_cfg = {
    .word_size = SAMPLE_BIT_WIDTH,
    .channels = CHANNELS,
    .format = I2S_FMT_DATA_FORMAT_I2S,
    .options = I2S_OPT_FRAME_CLK_MASTER | I2S_OPT_BIT_CLK_MASTER,
    .frame_clk_freq = SAMPLE_RATE,
    .block_size = BLOCK_SIZE,
    .timeout = 1000,
};

void i2s_example(void)
{
    const struct device *i2s_dev = DEVICE_DT_GET(DT_NODELABEL(i2s0));
    int ret;

    if (!device_is_ready(i2s_dev)) {
        printk("I2S device not ready\n");
        return;
    }

    /* 配置 I2S */
    ret = i2s_configure(i2s_dev, I2S_DIR_TX, &i2s_cfg);
    if (ret != 0) {
        printk("Failed to configure I2S: %d\n", ret);
        return;
    }

    /* 触发 I2S 开始 */
    ret = i2s_trigger(i2s_dev, I2S_DIR_TX, I2S_TRIGGER_START);
    if (ret != 0) {
        printk("Failed to start I2S: %d\n", ret);
        return;
    }
}
```

### 3.2 音频数据传输
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/drivers/i2s.h>

#define BLOCK_SIZE    512
#define SAMPLE_RATE   48000
#define SAMPLE_BIT_WIDTH 16
#define CHANNELS      2

static int16_t audio_buffer[BLOCK_SIZE / sizeof(int16_t)];

void audio_transfer_example(void)
{
    const struct device *i2s_dev = DEVICE_DT_GET(DT_NODELABEL(i2s0));
    void *tx_block;
    size_t size;
    int ret;

    /* 准备音频数据 */
    for (int i = 0; i < ARRAY_SIZE(audio_buffer); i++) {
        /* 生成正弦波 */
        audio_buffer[i] = (int16_t)(32767.0f *
                                  sinf(2.0f * 3.14159f * 1000.0f *
                                       i / SAMPLE_RATE));
    }

    /* 获取写入缓冲区 */
    ret = i2s_buf_write(i2s_dev, audio_buffer, BLOCK_SIZE);
    if (ret < 0) {
        printk("Failed to write to I2S buffer: %d\n", ret);
        return;
    }

    /* 读取音频数据 */
    ret = i2s_buf_read(i2s_dev, &tx_block, &size);
    if (ret < 0) {
        printk("Failed to read from I2S buffer: %d\n", ret);
        return;
    }

    /* 处理读取的数据 */
    process_audio_data(tx_block, size);

    /* 释放缓冲区 */
    i2s_buf_release(i2s_dev, tx_block);
}
```

## 4. 数字麦克风 (DMIC)

### 4.1 DMIC 配置
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/audio/dmic.h>

#define SAMPLE_RATE     48000
#define CHANNEL_COUNT   2
#define SAMPLE_BIT_WIDTH 16
#define BLOCK_SIZE      512
#define BLOCK_COUNT     4

static struct dmic_cfg dmic_config = {
    .io = {
        .min_pdm_clk_freq = 1024000,
        .max_pdm_clk_freq = 4096000,
        .min_pdm_clk_dc = 40,
        .max_pdm_clk_dc = 60,
        .pdm_clk_pol = 0,
        .pdm_data_pol = 0,
    },
    .streams = {
        {
            .pcm_rate = SAMPLE_RATE,
            .pcm_width = SAMPLE_BIT_WIDTH,
            .block_size = BLOCK_SIZE,
            .mem_slab = NULL,
        },
    },
    .channel = {
        .req_num_chan = CHANNEL_COUNT,
        .req_chan_map_lo = dmic_build_channel_map(0, 0, PDM_CHAN_LEFT) |
                         dmic_build_channel_map(1, 0, PDM_CHAN_RIGHT),
        .req_chan_map_hi = 0,
    },
};

void dmic_example(void)
{
    const struct device *dmic_dev = DEVICE_DT_GET(DT_NODELABEL(dmic0));
    int ret;

    if (!device_is_ready(dmic_dev)) {
        printk("DMIC device not ready\n");
        return;
    }

    /* 配置 DMIC */
    ret = dmic_configure(dmic_dev, &dmic_config);
    if (ret != 0) {
        printk("Failed to configure DMIC: %d\n", ret);
        return;
    }

    /* 触发 DMIC 开始 */
    ret = dmic_trigger(dmic_dev, DMIC_TRIGGER_START);
    if (ret != 0) {
        printk("Failed to start DMIC: %d\n", ret);
        return;
    }
}
```

### 4.2 DMIC 数据采集
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/audio/dmic.h>

#define BLOCK_SIZE      512

static K_MEM_SLAB_DEFINE(mic_mem_slab, BLOCK_SIZE, 4, 4);
static struct dmic_cfg dmic_config = {
    /* ... 配置同上 ... */
    .streams = {
        {
            .pcm_rate = 48000,
            .pcm_width = 16,
            .block_size = BLOCK_SIZE,
            .mem_slab = &mic_mem_slab,
        },
    },
};

void dmic_capture_example(void)
{
    const struct device *dmic_dev = DEVICE_DT_GET(DT_NODELABEL(dmic0));
    void *buffer;
    int ret;

    /* 配置 DMIC */
    ret = dmic_configure(dmic_dev, &dmic_config);
    if (ret != 0) {
        return;
    }

    /* 触发 DMIC 开始 */
    ret = dmic_trigger(dmic_dev, DMIC_TRIGGER_START);
    if (ret != 0) {
        return;
    }

    /* 读取麦克风数据 */
    ret = dmic_read(dmic_dev, 0, &buffer, BLOCK_SIZE, 1000);
    if (ret < 0) {
        printk("Failed to read DMIC data: %d\n", ret);
        return;
    }

    /* 处理麦克风数据 */
    process_mic_data(buffer, BLOCK_SIZE);

    /* 释放缓冲区 */
    k_mem_slab_free(&mic_mem_slab, buffer);

    /* 停止 DMIC */
    dmic_trigger(dmic_dev, DMIC_TRIGGER_STOP);
}
```

## 5. 音频处理

### 5.1 音频格式转换
```c
#include <zephyr/kernel.h>
#include <zephyr/audio/dsp.h>

/* 格式转换：16位有符号整数到32位浮点数 */
void audio_format_conversion(void)
{
    int16_t input[256];
    float output[256];
    int i;

    /* 准备输入数据 */
    for (i = 0; i < ARRAY_SIZE(input); i++) {
        input[i] = i * 100;
    }

    /* 转换格式 */
    for (i = 0; i < ARRAY_SIZE(input); i++) {
        output[i] = input[i] / 32768.0f;
    }

    /* 使用转换后的数据 */
    for (i = 0; i < ARRAY_SIZE(output); i++) {
        printk("Sample %d: %f\n", i, output[i]);
    }
}
```

### 5.2 音频滤波器
```c
#include <zephyr/kernel.h>
#include <zephyr/audio/dsp.h>

/* 简单的低通滤波器 */
void audio_filter_example(void)
{
    float input[256];
    float output[256];
    float coeff[5] = {0.1f, 0.2f, 0.4f, 0.2f, 0.1f};
    int i, j;

    /* 准备输入数据 */
    for (i = 0; i < ARRAY_SIZE(input); i++) {
        input[i] = sinf(2.0f * 3.14159f * 1000.0f * i / 48000.0f);
    }

    /* 应用滤波器 */
    for (i = 2; i < ARRAY_SIZE(input) - 2; i++) {
        output[i] = 0.0f;
        for (j = 0; j < ARRAY_SIZE(coeff); j++) {
            output[i] += input[i + j - 2] * coeff[j];
        }
    }

    /* 处理边界 */
    output[0] = input[0] * coeff[2];
    output[1] = input[0] * coeff[1] + input[1] * coeff[2] + input[2] * coeff[3];
    output[ARRAY_SIZE(output) - 2] = input[ARRAY_SIZE(input) - 4] * coeff[0] +
                                    input[ARRAY_SIZE(input) - 3] * coeff[1] +
                                    input[ARRAY_SIZE(input) - 2] * coeff[2];
    output[ARRAY_SIZE(output) - 1] = input[ARRAY_SIZE(input) - 3] * coeff[0] +
                                    input[ARRAY_SIZE(input) - 2] * coeff[1] +
                                    input[ARRAY_SIZE(input) - 1] * coeff[2];
}
```

### 5.3 音量控制
```c
#include <zephyr/kernel.h>

/* 音量控制函数 */
void audio_volume_control(int16_t *buffer, size_t size, int volume_percent)
{
    float gain = volume_percent / 100.0f;
    int i;

    /* 应用音量增益 */
    for (i = 0; i < size / sizeof(int16_t); i++) {
        float sample = buffer[i];
        sample *= gain;

        /* 限幅 */
        if (sample > 32767.0f) {
            sample = 32767.0f;
        } else if (sample < -32768.0f) {
            sample = -32768.0f;
        }

        buffer[i] = (int16_t)sample;
    }
}
```

## 6. 音频编解码

### 6.1 PCM 编码
```c
#include <zephyr/kernel.h>

/* PCM 编码参数 */
struct pcm_params {
    uint32_t sample_rate;
    uint8_t bit_depth;
    uint8_t channels;
};

/* PCM 编码函数 */
int pcm_encode(const float *samples, size_t num_samples,
              uint8_t *pcm_data, size_t pcm_size,
              const struct pcm_params *params)
{
    size_t bytes_per_sample = params->bit_depth / 8;
    size_t required_size = num_samples * bytes_per_sample;

    if (pcm_size < required_size) {
        return -ENOMEM;
    }

    for (size_t i = 0; i < num_samples; i++) {
        /* 将浮点样本转换为整数 */
        int32_t sample_int;
        
        if (params->bit_depth == 16) {
            sample_int = (int32_t)(samples[i] * 32767.0f);
            if (sample_int > 32767) sample_int = 32767;
            if (sample_int < -32768) sample_int = -32768;
            
            /* 写入 16 位 PCM 数据 */
            pcm_data[i * 2] = sample_int & 0xFF;
            pcm_data[i * 2 + 1] = (sample_int >> 8) & 0xFF;
        } else if (params->bit_depth == 24) {
            sample_int = (int32_t)(samples[i] * 8388607.0f);
            if (sample_int > 8388607) sample_int = 8388607;
            if (sample_int < -8388608) sample_int = -8388608;
            
            /* 写入 24 位 PCM 数据 */
            pcm_data[i * 3] = sample_int & 0xFF;
            pcm_data[i * 3 + 1] = (sample_int >> 8) & 0xFF;
            pcm_data[i * 3 + 2] = (sample_int >> 16) & 0xFF;
        }
    }

    return required_size;
}
```

### 6.2 WAV 文件格式
```c
#include <zephyr/kernel.h>
#include <zephyr/fs/fs.h>

/* WAV 文件头 */
struct wav_header {
    /* RIFF 块 */
    uint8_t riff_header[4];    /* "RIFF" */
    uint32_t wav_size;         /* 文件大小 - 8 */
    uint8_t wave_header[4];    /* "WAVE" */

    /* fmt 块 */
    uint8_t fmt_header[4];     /* "fmt " */
    uint32_t fmt_chunk_size;   /* 16 for PCM */
    uint16_t audio_format;     /* 1 for PCM */
    uint16_t num_channels;
    uint32_t sample_rate;
    uint32_t byte_rate;
    uint16_t block_align;
    uint16_t bits_per_sample;

    /* data 块 */
    uint8_t data_header[4];    /* "data" */
    uint32_t data_bytes;
};

/* 创建 WAV 文件 */
int create_wav_file(const char *filename, const uint8_t *audio_data,
                   size_t data_size, uint32_t sample_rate,
                   uint16_t channels, uint16_t bit_depth)
{
    struct fs_file_t file;
    struct wav_header header;
    int ret;

    /* 初始化文件对象 */
    fs_file_t_init(&file);

    /* 创建文件 */
    ret = fs_open(&file, filename, FS_O_CREATE | FS_O_WRITE);
    if (ret != 0) {
        return ret;
    }

    /* 填充 WAV 头 */
    memcpy(header.riff_header, "RIFF", 4);
    header.wav_size = data_size + sizeof(header) - 8;
    memcpy(header.wave_header, "WAVE", 4);
    
    memcpy(header.fmt_header, "fmt ", 4);
    header.fmt_chunk_size = 16;
    header.audio_format = 1; /* PCM */
    header.num_channels = channels;
    header.sample_rate = sample_rate;
    header.byte_rate = sample_rate * channels * (bit_depth / 8);
    header.block_align = channels * (bit_depth / 8);
    header.bits_per_sample = bit_depth;
    
    memcpy(header.data_header, "data", 4);
    header.data_bytes = data_size;

    /* 写入 WAV 头 */
    ret = fs_write(&file, &header, sizeof(header));
    if (ret < 0) {
        fs_close(&file);
        return ret;
    }

    /* 写入音频数据 */
    ret = fs_write(&file, audio_data, data_size);
    if (ret < 0) {
        fs_close(&file);
        return ret;
    }

    /* 关闭文件 */
    fs_close(&file);
    return 0;
}
```

## 7. 音频应用示例

### 7.1 音频录制与播放
```c
#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/audio/dmic.h>
#include <zephyr/drivers/i2s.h>

#define SAMPLE_RATE     48000
#define SAMPLE_BIT_WIDTH 16
#define CHANNELS        2
#define BLOCK_SIZE      512
#define BLOCK_COUNT     4

/* 内存分配 */
static K_MEM_SLAB_DEFINE(mic_mem_slab, BLOCK_SIZE, BLOCK_COUNT, 4);
static K_MEM_SLAB_DEFINE(i2s_mem_slab, BLOCK_SIZE, BLOCK_COUNT, 4);

/* DMIC 配置 */
static struct dmic_cfg dmic_config = {
    /* ... 配置同上 ... */
    .streams = {
        {
            .pcm_rate = SAMPLE_RATE,
            .pcm_width = SAMPLE_BIT_WIDTH,
            .block_size = BLOCK_SIZE,
            .mem_slab = &mic_mem_slab,
        },
    },
};

/* I2S 配置 */
static const struct i2s_config i2s_cfg = {
    .word_size = SAMPLE_BIT_WIDTH,
    .channels = CHANNELS,
    .format = I2S_FMT_DATA_FORMAT_I2S,
    .options = I2S_OPT_FRAME_CLK_MASTER | I2S_OPT_BIT_CLK_MASTER,
    .frame_clk_freq = SAMPLE_RATE,
    .block_size = BLOCK_SIZE,
    .timeout = 1000,
    .mem_slab = &i2s_mem_slab,
};

void audio_record_playback(void)
{
    const struct device *dmic_dev = DEVICE_DT_GET(DT_NODELABEL(dmic0));
    const struct device *i2s_dev = DEVICE_DT_GET(DT_NODELABEL(i2s0));
    void *buffer;
    int ret;

    /* 配置 DMIC */
    ret = dmic_configure(dmic_dev, &dmic_config);
    if (ret != 0) {
        return;
    }

    /* 配置 I2S */
    ret = i2s_configure(i2s_dev, I2S_DIR_TX, &i2s_cfg);
    if (ret != 0) {
        return;
    }

    /* 启动 DMIC 和 I2S */
    dmic_trigger(dmic_dev, DMIC_TRIGGER_START);
    i2s_trigger(i2s_dev, I2S_DIR_TX, I2S_TRIGGER_START);

    while (1) {
        /* 读取麦克风数据 */
        ret = dmic_read(dmic_dev, 0, &buffer, BLOCK_SIZE, 1000);
        if (ret < 0) {
            continue;
        }

        /* 处理音频数据 */
        process_audio_data(buffer, BLOCK_SIZE);

        /* 写入 I2S 播放 */
        ret = i2s_write(i2s_dev, buffer, BLOCK_SIZE);
        if (ret < 0) {
            k_mem_slab_free(&mic_mem_slab, buffer);
            continue;
        }

        /* 释放缓冲区 */
        k_mem_slab_free(&mic_mem_slab, buffer);
    }
}
```

### 7.2 音频效果处理
```c
#include <zephyr/kernel.h>

/* 回声效果参数 */
struct echo_params {
    int16_t *delay_buffer;
    size_t buffer_size;
    size_t delay_samples;
    float decay;
    size_t write_index;
};

/* 初始化回声效果 */
void echo_init(struct echo_params *echo, size_t delay_ms, float decay,
              uint32_t sample_rate)
{
    echo->delay_samples = (delay_ms * sample_rate) / 1000;
    echo->buffer_size = echo->delay_samples * sizeof(int16_t);
    echo->delay_buffer = k_malloc(echo->buffer_size);
    echo->decay = decay;
    echo->write_index = 0;

    if (echo->delay_buffer) {
        memset(echo->delay_buffer, 0, echo->buffer_size);
    }
}

/* 应用回声效果 */
void apply_echo_effect(struct echo_params *echo, int16_t *buffer,
                      size_t num_samples)
{
    size_t i;
    size_t read_index;

    if (!echo->delay_buffer) {
        return;
    }

    for (i = 0; i < num_samples; i++) {
        /* 计算读取索引 */
        read_index = (echo->write_index + i) % echo->delay_samples;

        /* 添加回声 */
        int32_t sample = buffer[i];
        sample += (int32_t)(echo->delay_buffer[read_index] * echo->decay);

        /* 限幅 */
        if (sample > 32767) {
            sample = 32767;
        } else if (sample < -32768) {
            sample = -32768;
        }

        /* 更新缓冲区 */
        echo->delay_buffer[read_index] = buffer[i];
        buffer[i] = (int16_t)sample;
    }

    /* 更新写入索引 */
    echo->write_index = (echo->write_index + num_samples) % echo->delay_samples;
}
```

## 8. 音频子系统最佳实践

1. 缓冲区管理：
   - 使用适当大小的缓冲区以平衡延迟和处理开销
   - 实现双缓冲或环形缓冲区以避免数据丢失
   - 考虑使用 DMA 进行高效数据传输

2. 采样率和位深度：
   - 根据应用需求选择合适的采样率和位深度
   - 考虑功耗和性能之间的权衡
   - 实现采样率转换以支持不同设备

3. 音频处理：
   - 在定点数学运算中使用适当的缩放以避免溢出
   - 考虑使用 DSP 加速指令进行处理
   - 实现高效的滤波器算法

4. 同步处理：
   - 确保音频捕获和播放之间的同步
   - 实现缓冲区调整以处理时钟偏差
   - 考虑使用硬件同步机制

5. 电源管理：
   - 在不使用时禁用音频设备
   - 实现动态时钟调整以节省功耗
   - 考虑使用低功耗模式

6. 错误处理：
   - 实现完善的错误检测和恢复机制
   - 处理缓冲区溢出和欠载情况
   - 提供诊断信息

7. 测试与验证：
   - 进行音频质量测试
   - 测量延迟和抖动
   - 验证在不同条件下的性能

8. 可扩展性：
   - 设计模块化的音频处理管道
   - 支持动态加载和卸载音频处理模块
   - 考虑未来的功能扩展