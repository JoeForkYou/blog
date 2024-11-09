---
title: Andoird camera app从零养成计划【二】
date: 2024-11-10 02:29:38
tags: [Android, APP]
---

# 1 API1

AndroidManifest.xmlAndroidManifest官方解释是应用清单（manifest意思是货单），每个应用的根目录中都必须包含一个，并且文件名必须一模一样。这个文件中包含了APP的配置信息，系统需要根据里面的内容运行APP的代码，显示界面。示例如下:

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.newcamera">
    <uses-permission android:name="android.permission.CAMERA"/>   //使用camera的权限
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/> //写文件的权限
    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"  //apk图标
        android:supportsRtl="true"
        android:theme="@style/Theme.NewCamera">
        <activity
            android:name=".PreviewActivity"
            android:exported="false" />
        <activity
            android:name=".MainActivity"     //告知打开apk的主Activity的入口
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

对于API 1 来说.打开camera 的对象已经封装好.在android/hardware/Camera.java中.

Camera API 中主要涉及以下几个关键类:

- Camera:操作和管理相机资源.支持相机资源切换.设置预览和拍摄尺寸.设置光圈,曝光等相关参数.

- SurfaceView:用于绘制相机预览图像.提供实时预览的图像

- SurfaceHolder:用于控制Surface的一个抽象接口.它可以控制Surface的尺寸,格式与像素等.并可以监视Surface的变化.

- SurfaceHolder.Callback:用于监听Surface状态变化的接口



SurfaceView和普通的View相比有什么区别呢？

普通View都是共享一个Surface的,所有的绘制也都在UI线程中进行.因为UI线程还要处理其他逻辑.因此对View的更新速度和绘制帧率无法保证.这显然不适合相机实时预览这种情况.因而SurfaceView持有一个单独Surface.它负责管理这个Surface的格式.尺寸以及显示位置,它的Surface绘制也在单独的线程中进行,因而拥有更高的绘制效率和帧率。



SurfaceHolder.Callback接口里定义了三个函数:

- **surfaceCreated(SufaceHolder holder)**;当Surface第一次创建的时候调用.可以在这个方法里调用camera.open(),camera.setPreviewDisplay()来实现打开相机以及连接Camera与Surface等操作
- **surfaceChanged(SurfaceHolder holder,int format,int width,int height)**;当Surface的size,format等发生变化的时候调用,可以在这个方法里调用camera.startPreview()开启预览
- **surfaceDestroyed(SurfaceHolder holder)**;

在打开相机前,我们需要获取到相机的一些基础信息

```java
    private void getCameraInfo() {
        //有多少个摄像头
        int numberOfCameras = Camera.getNumberOfCameras();

        for (int i = 0; i < numberOfCameras; ++i) {
            final Camera.CameraInfo cameraInfo = new Camera.CameraInfo();

            Camera.getCameraInfo(i, cameraInfo);
            //后置摄像头
            if (cameraInfo.facing == Camera.CameraInfo.CAMERA_FACING_BACK) {
                faceBackCameraId = i;
                faceBackCameraOrientation = cameraInfo.orientation;
            }
            //前置摄像头
            else if (cameraInfo.facing == Camera.CameraInfo.CAMERA_FACING_FRONT) {
                faceFrontCameraId = i;
                faceFrontCameraOrientation = cameraInfo.orientation;
            }
        }

        Log.e(TAG,"faceBackCameraId ="+faceBackCameraId+"\tfaceBackCameraOrientation="+faceBackCameraOrientation);
        Log.e(TAG,"faceFrontCameraId="+faceFrontCameraId+"\tfaceFrontCameraOrientation="+faceFrontCameraOrientation);
    }
```

实际上打印出来的是

```
02-09 14:05:28.078  5280  5280 E CameraXTT: faceBackCameraId =0	faceBackCameraOrientation=90
02-09 14:05:28.078  5280  5280 E CameraXTT: faceFrontCameraId=1	faceFrontCameraOrientation=270
```

## 1.1 打开相机

知道了相机的相关信息,就可以通过对应的CameraID来打开对应的cameraDevices.注意 只针对单摄.双摄打开原理不一样.示例如下:

```java
import android.hardware.Camera;

private Camera camera = null;

camera = Camera.open(0);   //open(参数),参数对应的camera id//针对单摄的情况.可以通过这个方法直接打开对应的device设备.

camera.setPreviewDisplay(sfv_preview.getHolder());//sfv_preview是定义的SurfaceView,用来呈现相机的预览.
camera.setDisplayOrientation(90);   //让相机旋转90度,相机方向不对会出现拉伸情况.
camera.startPreview();
```

打开相机后可以获取到一个camera的对象.从这个对象里可以获取和设置相机的各种参数信息.

```java
camera.getParameters();
这个后面跟对应的参数：
例如:
 camera.getParameters().getAntibanding()
 //获取预览尺寸
 Log.e(TAG,"w"+camera.getParameters().getPreviewSize().width+"h="+camera.getParameters().getPreviewSize().height);
```

## 1.2 关闭相机

关闭相机要先停止预览再release()即可

```java
camera.stopPreview();
camera.release();
```

## 1.3 拍照

拍照通过调用Camera的takePicture()方法来完成的.

```
takePicture(ShutterCallback shutter, PictureCallback raw, PictureCallback postview, PictureCallback jpeg)
```

该方法有三个参数

- ShutterCallback shutter:在拍照的瞬间被回调.这里通常可以播放"咔擦"的音效

- PictureCallback raw:返回未经压缩的图像数据

- PictureCallback postview:返回postview的图像数据

- PictureCallback jpeg:返回经过JPEG压缩的图像数据

  我们一般用的是最后一个.实现最后一个PictureCallback即可

  ```java
  private void takePic(){
          camera.takePicture(null, null, new Camera.PictureCallback() {
              @Override
              public void onPictureTaken(byte[] data, Camera camera) {
                  Bitmap bmp = BitmapFactory.decodeByteArray(data, 0 ,data.length);
                  String fileName = Environment.getExternalStorageDirectory().toString()
                                  +File.separator
                                  +"DCIM/Camera"
                                  +File.separator
                                  +"PicTest_"+System.currentTimeMillis()+".jpg";
                  Log.e(TAG,"fileName="+fileName);
                  File file = new File(fileName);
                  if(!file.getParentFile().exists()){
                      file.getParentFile().mkdir();
                  }
                  try {
                      BufferedOutputStream bos=new BufferedOutputStream(new FileOutputStream(file));
                      bmp.compress(Bitmap.CompressFormat.JPEG, 80, bos);//向缓冲区压缩图片
                      bos.flush();
                      bos.close();
                      Toast.makeText(MainActivity.this, "拍照成功，照片保存在"+fileName+"文件之中！", Toast.LENGTH_LONG).show();
                  }catch (Exception e){
                      // TODO Auto-generated catch block
                      //e.printStackTrace();
                      Toast.makeText(MainActivity.this, "拍照失败！"+e.toString(), Toast.LENGTH_LONG).show();
                  }
                  stopPreview();
                  startPreivew();
              }
          });
      }
  ```

  

# 2 API2

叫出CameraManager ，打开 CameraDevice ，拿住CameraCaptureSession，发送CaptureRequest .

Camera API2中主要涉及的以下几个关键类:

- CameraManager:摄像头管理器.用于打开和关闭系统摄像头
- CameraCharacteristics:描述摄像头的各种特性.我们可以通过CameraManager的getCameraCharacteristics(@NonNull String cameraId)方法来获取
- CameraDevice:描述系统摄像头.类似早期的Camera
- CameraCaptureSession:Session类.当需要拍照,预览等功能时,需要创建该类的实例.然后通过该实例里的方法进行控制(例如:拍照 capture())
- CaptureRequest:描述了一个操作请求,拍照,预览等操作都需要先传入CaptureRequest参数，具体的参数控制也是通过CameraRequest的成员变量来设置
- CaptureResult:描述拍照完成后的结果

开发者通过创建CaptureRequest向摄像头发起Capture请求,这些请求会排成一个队列供摄像头处理,摄像头将结果包装在CaptureMetadata中返回给开发者.整个流程建立一个CameraCaptureSession的会话中.

## 2.1 打开相机

打开相机之前,要获取到CameraManger,然后获取相机列表,进而获取各个摄像头(主要是前摄和后摄)的参数

```java
CameraManager manager = (CameraManager) getSystemService(Context.CAMERA_SERVICE);
```


