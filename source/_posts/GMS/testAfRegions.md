---
title: testAfRegions
tags:
  - cts
  - Android
  - GMS
categories: [Android, GMS,cts]
abbrlink: 61038
date: 2024-11-10 07:46:56
---

# 1 测试流程
测试命令:
```shell
adb shell am instrument -e class android.hardware.camera2.cts.StillCaptureTest#testAfRegions[1] --abi arm64-v8a  -w android.camera.cts/androidx.test.runner.AndroidJUnitRunner 
```
测试代码的位置如下:

cts/tests/camera/src/android/hardware/camera2/cts/StillCaptureTest.java

```java
    /**
     * Test Af region for still capture.
     */
    @Test
    public void testAfRegions() throws Exception {
        for (String id : mCameraIdsUnderTest) {
            try {
                Log.i(TAG, "Testing AF regions for Camera " + id);
                openDevice(id);

                boolean afRegionsSupported = isRegionsSupportedFor3A(MAX_REGIONS_AF_INDEX);
                Log.i(TAG,"afRegionsSupported="+afRegionsSupported); //可以加这个log. true则继续往下测试,反之直接跳过这个camera
                if (!afRegionsSupported) {
                    continue;
                }

                ArrayList<MeteringRectangle[]> afRegionTestCases = get3ARegionTestCasesForCamera();//看下述这个函数的说明
                for (MeteringRectangle[] afRegions : afRegionTestCases) { //遍历afRegionTestCases的内容去测试
                    takePictureTestByCamera(/*aeRegions*/null, /*awbRegions*/null, afRegions);
                }
            } finally {
                closeDevice();
                closeImageReader();
            }
        }
    }
```

```java
 /**
     * Get 5 3A region test cases, each with one square region in it.
     * The first one is at center, the other four are at corners of
     * active array rectangle.
     *
     * @return array of test 3A regions
     */
    private ArrayList<MeteringRectangle[]> get3ARegionTestCasesForCamera() 
        
        ...
        Log.v(TAG, "Generated test regions are: " + sb.toString()); //这边可以把这个log打开看这五个区域的坐标分别是多少. 这五个区域的坐标会返回到上述afRegionTestCases,然后遍历去测试takePictureTestByCamera
```

主要执行的测试内容在这个函数takePictureTestByCamera

```
Step 1: trigger an auto focus run, and wait for AF locked.
Step 2: AF is already locked, wait for AWB converged, then lock it.
Step 3: trigger an AE precapture metering sequence and wait for AE converged.
Step 4: take a picture when all 3A are in good state
```

我们遇到的报错内容如下:

```java
mCollector.expectMeteringRegionsAreSimilar(
                    "AF regions in result and request should be similar",
                    afRegions,
                    resultAfRegions,
                    METERING_REGION_ERROR_PERCENT_DELTA);
        }
```

afRegions和resultAfRegions内容不一致.

resultAfRegions是重新下发上报的: afRegions则上述遍历中传递过来的值.

```java
MeteringRectangle[] resultAfRegions =
                    getValueNotNull(result, CaptureResult.CONTROL_AF_REGIONS);
```

# 2 hal 上报

上报路径:

vendor/mediatek/proprietary/hardware/mtkcam/aaa/source/common/hal3a/v1.0/Hal3AAdapter3.cpp

关注这个值的下发:CaptureResult.CONTROL_AF_REGIONS, mtk平台对andorid 机制的处理都会映射转成对应的mtk标准.

这个值对应的就是MTK_CONTROL_AF_REGIONS.

详细的内容见如下的文件:

vendor/mediatek/proprietary/hardware/mtkcam/include/mtkcam/utils/metadata/client/TagMap.h

```c
     _IMP_TAGCONVERT_(    ANDROID_CONTROL_AF_MODE,    MTK_CONTROL_AF_MODE)\
     _IMP_TAGCONVERT_(    ANDROID_CONTROL_AF_REGIONS,    MTK_CONTROL_AF_REGIONS)\
```

看文件Hal3AAdapter3.cpp内容:这是解析meta的内容.

```c++
MUINT8
Hal3AAdapter3::
parseMeta(const vector<MetaSet_T*>& requestQ){
    ...
        if ( (!u1RepeatTag) || ReparseMetaForDummy) // not repeating tag, parse app meta
    ...
            case MTK_CONTROL_AF_REGIONS:
    ...
        mUpdateMetaResult.push({MTK_CONTROL_AF_REGIONS, entryNew}); //这里是上报上去的位置.将entryNew的内容上报上去
}
```

往上翻看entryNew的内容从哪里来,从rArea处获取.

```c++
entryNew.push_back(rArea.i4Left,   Type2Type<MINT32>());
entryNew.push_back(rArea.i4Top,    Type2Type<MINT32>());
entryNew.push_back(rArea.i4Right,  Type2Type<MINT32>());
entryNew.push_back(rArea.i4Bottom, Type2Type<MINT32>());
entryNew.push_back(rArea.i4Weight, Type2Type<MINT32>());
```
rArea又是根据rArea和rSclCrop存在以下的逻辑关系.
```c++
rArea.i4Left   = MIN(MAX(rArea.i4Left, rSclCrop[0]), rSclCrop[2]);
rArea.i4Top    = MIN(MAX(rArea.i4Top, rSclCrop[1]), rSclCrop[3]);
rArea.i4Right  = MAX(MIN(rArea.i4Right, rSclCrop[2]), rSclCrop[0]);
rArea.i4Bottom = MAX(MIN(rArea.i4Bottom, rSclCrop[3]), rSclCrop[1]);
```

检查rSclCrop的内容.

```c++
        mi4AppfgCrop = QUERY_ENTRY_SINGLE(_appmeta, MTK_SCALER_CROP_REGION, rSclCropRect);
        fgCrop = mi4AppfgCrop;

        mAppCropRegion.p.x = rSclCropRect.p.x;
        mAppCropRegion.p.y = rSclCropRect.p.y;
        mAppCropRegion.s.w = rSclCropRect.s.w;
        mAppCropRegion.s.h = rSclCropRect.s.h;
        rSclCrop[0] = rSclCropRect.p.x;
        rSclCrop[1] = rSclCropRect.p.y;
        rSclCrop[2] = rSclCropRect.p.x + rSclCropRect.s.w;
        rSclCrop[3] = rSclCropRect.p.y + rSclCropRect.s.h;
```

该问题出现的原因是有人误删了 rSclCrop相关的内容导致该相关内容呈现如下:

```c++
         mi4AppfgCrop = QUERY_ENTRY_SINGLE(_appmeta, MTK_SCALER_CROP_REGION, rSclCropRect);
         fgCrop = mi4AppfgCrop;
         rSclCrop[3] = rSclCropRect.p.y + rSclCropRect.s.h;
```


