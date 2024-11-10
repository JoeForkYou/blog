---
title: mtk_fd参数说明
tags: camera
categories: camera
abbrlink: 42935
date: 2024-11-10 14:49:47
---
# 1 参数说明 
vendor/mediatek/proprietary/custom/mt6765/hal/camera/camera_custom_fd.cpp
```cpp

#include "camera_custom_fd.h"

void get_fd_CustomizeData(FD_Customize_PARA  *FDDataOut)
{
    FDDataOut->FDThreadNum = 1;
    FDDataOut->FDThreshold = 256;
    FDDataOut->MajorFaceDecision = 1;
    FDDataOut->OTRatio = 1088;
    FDDataOut->SmoothLevel = 8;
    FDDataOut->Momentum = 0;
    FDDataOut->MaxTrackCount = 10;
    FDDataOut->FDSkipStep = 2;
    FDDataOut->FDRectify = 10;
    FDDataOut->FDRefresh = 3;
    FDDataOut->SDThreshold = 69;
    FDDataOut->SDMainFaceMust = 1;
    FDDataOut->SDMaxSmileNum = 3;
    FDDataOut->GSensor = 1;
    FDDataOut->FDModel = 1;
    FDDataOut->OTFlow = 1;  //0:Original Flow (FDRefresh:60)  , 1:New Flow (FDRefresh:3)
    FDDataOut->UseCustomScale = 1;
    FDDataOut->FDSizeRatio = 0.0;  // float:0~1
    FDDataOut->SkipPartialFD = 0;
    FDDataOut->SkipAllFD = 0;
}
```
比较常客制化的一些值及其意义：

**FDThreshold** ： tune FD detection rate and false positive rate 。值越大代表检测的越严格。

**MajorFaceDecision** ： 决定 face 排列方式。 value = 0 or 1，value = 0，则以Face Size的大小作为检测标准，也就是优先检测最大的人脸。value = 1，优先检测在画面中心的人脸。Face AE,AF会参考Major Face资讯 。

**SmoothLevel**：  决定人脸框的移动速度。value: 8~16 。值越大，跟随感越慢。 会把前面value的值平均作为下一次移动的参考。value值越大，人脸框移动会smooth，若人脸移动速度太快，则会出现人脸框跳动的情形。

**MaxTrackCount**： 当人脸 lose 时，会用 tracking 机制继续 keep 的帧数。 

**GSensor**： 是否使用 GSensor 资讯(AP带下来)。如果为0，则会做四个角度轮流侦测，initial detection time 会变慢。

**OTFlow**： 只能是 1 且必须是 1 。

**FDSizeRatio**： 用来设置过滤图中某个比例 以下的人脸。

**FDThreadNum**：value 值增大时时会加大cpuloading，相对的检测Face的速度也快，该Thread主要跑的是FD Algo 。

**OTRatio**:value越大，当周围环境change时，人脸框越不容易fail，缺点是可能追踪到不是人脸的物体 。

**Momentum**:值可以是 0~3 。 0 = force to project direction ；3 = no reference project direction

**FDSkipStep**:跳点，为了提高SW FD的performance 。

**FDRefresh**:不是每一帧都做FD，若检测到Face后，接下来会做Face Tracking，若value = 3，则做3次FaceTracking（几毫秒可以做

一次）。

**SDMainFaceMust** :value = 0 or 1，为0则会检测前面三张Face；为1，则需要根据MajorFaceDecision 的值确定，可能不会起作用 。

**FDModel** :FD的核心是用某种算法training出来的model，不同的database或参数就会training出不同的model，也可以理解为侧重点不同。 

建议不要修改 OTRatio 、SmoothLevel 、FDRectify ，会影响 tracking。