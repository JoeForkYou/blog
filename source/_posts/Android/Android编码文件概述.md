---
title: Android编码文件概述
tags: Android
abbrlink: 9532
date: 2024-11-10 02:35:12
---
@[toc]
# 1 概述
需要小心的是修改item后要注意CTS测项testGetWithId(android.media.cts.CamcorderProfileTest)，也就是high profile一定要和分辨率的profile适配，比如spec为1920 x 1080的spec(平台所supprot的)
那么high的分辨率和1080p的分辨率一定要一样

```xml
     <EncoderProfile quality="1080p" fileFormat="3gp" duration="30">
            <Video codec="h264"
                   bitRate="17000000"
                   width="1920"
                   height="1080"
                   frameRate="30" />
            <Audio codec="aac"
                   bitRate="128000"
                   sampleRate="48000"
                   channels="2" />
        </EncoderProfile>
```

```xml
        <EncoderProfile quality="high" fileFormat="3gp" duration="30">
            <Video codec="h264"
                   bitRate="17000000"
                   width="1920"
                   height="1080"
                   frameRate="30" />
            <Audio codec="aac"
                   bitRate="128000"
                   sampleRate="48000"
                   channels="2" />
        </EncoderProfile>
```

# 2 标准尺寸

详细参考谷歌官网说明:

https://developer.android.com/reference/android/media/CamcorderProfile.html#QUALITY_1080P

```
public static final int QUALITY_1080P 
public static final int QUALITY_2160P
public static final int QUALITY_2K
public static final int QUALITY_480P
public static final int QUALITY_4KDCI
public static final int QUALITY_720P
public static final int QUALITY_8KUHD
public static final int QUALITY_CIF
public static final int QUALITY_HIGH
public static final int QUALITY_HIGH_SPEED_1080P
```

| 标准尺寸                 |                                                              | 常量值                     |
| ------------------------ | ------------------------------------------------------------ | -------------------------- |
| QUALITY_1080P            | 1920x1080                                                    | 6          (0x00000006)    |
| QUALITY_2160P            | 3840x2160                                                    | 8          (0x00000008)    |
| QUALITY_2K               | 2048x1080                                                    | 12          (0x0000000c)   |
| QUALITY_480P             | 720 x 480                                                    | 4          (0x00000004)    |
| QUALITY_4KDCI            | 4096 x 2160                                                  | 10          (0x0000000a)   |
| QUALITY_720P             | 1280 x 720                                                   | 5          (0x00000005)    |
| QUALITY_8KUHD            | 7680 x 4320                                                  | 13          (0x0000000d)   |
| QUALITY_CIF              | 352 x 288                                                    | 3          (0x00000003)    |
| QUALITY_HIGH             | Quality level corresponding to the highest available resolution. | 1          (0x00000001)    |
| QUALITY_HIGH_SPEED_1080P | High speed ( >= 100fps) quality level corresponding to the 1080p (1920 x 1080 or 1920x1088) resolution. | 2004          (0x000007d4) |
| QUALITY_HIGH_SPEED_2160P | High speed ( >= 100fps) quality level corresponding to the 2160p (3840 x 2160) resolution. | 2005          (0x000007d5) |

可以通过如下代码获取到所支持的编码尺寸

```
public static EncoderProfiles getAll (String cameraId, 
                int quality)
```

```
这个文件实际对应camera video 调用关系.

系统启动后，通过CamcorderProfile.java：static{ } 块，初始化并解析好。以供上层获取。

进入录像模式后：VideoMode.java：initRecorder——>configRecoderSpec——>getProfile，去获取摄像头或录像的默认配置。

native层主要是在 /frameworks/av/media/libmedia/MediaProfiles.cpp 文件里加载和检查参数。
```




