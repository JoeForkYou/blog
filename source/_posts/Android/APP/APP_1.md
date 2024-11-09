---
title: Andoird camera app从零养成计划【一】
date: 2024-11-10 00:41:32
tags: [Android, APP]
---

要创建一个基本的Camera App demo，我们将使用Android Studio和Java来编写一个应用，该应用能够打开相机预览，拍照，并保存照片到设备的存储中。这里将使用Android的Camera2 API，因为它提供了更丰富的功能和更好的性能，尽管它比Camera API（已弃用）更复杂一些。

### 步骤 1: 创建一个新的Android项目

1. 打开Android Studio，选择“Start a new Android Studio project”。
2. 选择“Empty Activity”，然后点击“Next”。
3. 填写你的应用名称（如 `CameraDemo`），选择你的保存位置，语言选择Java，最小API级别设置为21（因为Camera2 API在API 21（Android 5.0）上引入）。
4. 点击“Finish”创建项目。

### 步骤 2: 添加权限

在你的 `AndroidManifest.xml` 文件中添加必要的权限：

```xml
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="你的包名">

    <uses-permission android:name="android.permission.CAMERA"/>
    <uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE"/>
    <uses-feature android:name="android.hardware.camera" android:required="true"/>
    <uses-feature android:name="android.hardware.camera.autofocus"/>

    <application
        ...
        >
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

### 步骤 3: 布局文件

修改 `res/layout/activity_main.xml` 文件来添加必要的视图控件（如TextureView用于显示相机预览，Button用于拍照）：

```xml
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".MainActivity">

    <TextureView
        android:id="@+id/textureView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:layout_above="@+id/button_capture" />

    <Button
        android:id="@+id/button_capture"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:layout_centerHorizontal="true"
        android:text="拍照" />

</RelativeLayout>
```

### 步骤 4: 编写MainActivity

由于Camera2 API较为复杂，这里不会详细展开全部代码，但会概述主要步骤和关键代码片段。

1. **初始化Camera2 API**：打开相机，设置预览大小，创建CaptureSession等。
2. **设置TextureView显示预览**。
3. **处理拍照和保存**：在点击按钮时，捕获图像并保存到存储。

你需要创建多个类来处理Camera2的不同部分，如CameraStateCallback、CaptureRequest等。

### 示例代码片段（MainActivity部分）

```java
public class MainActivity extends AppCompatActivity {
    private TextureView textureView;
    private CameraDevice cameraDevice;
    private CaptureRequest.Builder previewRequestBuilder;
    // 其他变量...

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        textureView = findViewById(R.id.textureView);
        // 初始化相机，设置TextureView显示预览等...
    }

    // 拍照按钮点击事件
    public void onCaptureButtonClick(View view) {
        // 拍照逻辑...
    }

    // 相机打开，关闭等状态的回调
    private CameraDevice.StateCallback stateCallback = new CameraDevice.StateCallback() {
        @Override
        public void onOpened(@NonNull CameraDevice camera) {
            // 相机成功打开
            cameraDevice = camera;
        }

        @Override
        public void onDisconnected(@NonNull CameraDevice camera) {
            // 相机被断开
            cameraDevice.close();
        }

        @Override
        public void onError(@NonNull CameraDevice camera, int error) {
            // 相机发生错误
            cameraDevice.close();
            cameraDevice = null;
        }
    };

    // 其他方法...
}
```

### 步骤 5: 运行时权限请求

由于Android 6.0（API 级别 23）及以上版本需要在运行时请求权限，你需要检查并在必要时请求权限。

### 总结

这里只提供了一个基本的框架和思路。Camera2 API 涉及很多复杂的步骤和概念，如处理相机状态、创建和管理CaptureRequests、SurfaceTexture等。为了完整实现功能，你需要深入研究Camera2 API的文档和示例代码。


