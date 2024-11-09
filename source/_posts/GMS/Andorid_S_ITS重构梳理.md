---
title: Android_S_ITS重构梳理
date: 2024-11-10 02:26:46
tags: [Android, ITS , GMS]
---
# 1 概述

先看its环境搭建的文档.里面有基础操作说明.

然后为了更好理解梳理its的内容.我顺道整理出its 测试的脚本内容.

单刀直入.找到这个总的测试文件.我们所有的整跑命令都是从这个文件中调用起来的.

android-cts-verifier/CameraITS/tools/run_all_tests.py
main函数的入口

```python
def main():
```
设置输出的tmp文件即测试生成的图片和log 保存的路径.
```python 
  logging.basicConfig(level=logging.INFO)
  # Make output directories to hold the generated files.
  topdir = tempfile.mkdtemp(prefix='CameraITS_')
  subprocess.call(['chmod', 'g+rx', topdir])
  logging.info('Saving output files to: %s', topdir)
```
将输入的sys参数即场景拼接到scenes. 这个是针对直接输入测试cmd后面衔接场景和camera的.(exp:  python tools/run_all_tests.py camera=0 scenes=0)
```python
  # Override camera & scenes with cmd line values if available
  for s in list(sys.argv[1:]):
    if 'scenes=' in s:
      scenes = s.split('=')[1].split(',')
    elif 'camera=' in s:
      camera_id_combos = s.split('=')[1].split(',')
```

读取测试testBeds. 这边和config.yml对应起来.

```python
  # Read config file and extract relevant TestBed
  config_file_contents = get_config_file_contents()
  for i in config_file_contents['TestBeds']:
    if scenes == ['sensor_fusion']:
      if TEST_KEY_SENSOR_FUSION not in i['Name'].lower():
        config_file_contents['TestBeds'].remove(i)
    else:
      if TEST_KEY_SENSOR_FUSION in i['Name'].lower():
        config_file_contents['TestBeds'].remove(i)
```

我们的config.yml 每一个TestBeds:下面会衔接
\- Name: TEST_BED_TABLET_SCENES  # Need 'tablet' in name for tablet scenes

\- Name: TEST_BED_SENSOR_FUSION  # Need 'sensor_fusion' in name for SF tests

分别对应标准灯箱和马达灯箱

继续往下看. 如果没有直接输入测试cmd后衔接参数. 而是直接python tools/run_all_tests.py.

以下逻辑会走进去判断. 因为这几个值都没写.会直接读取config.yml的对应字符camera和scene后衔接的参数.

```python
  # Get test parameters from config file
  test_params_content = get_test_params(config_file_contents)
  if not camera_id_combos:
    camera_id_combos = str(test_params_content['camera']).split(',')
  if not scenes:
    scenes = str(test_params_content['scene']).split(',')
    scenes = [_INT_STR_DICT.get(n, n) for n in scenes]  # recover '1_1' & '1_2'
```

获取config.yml 中dut(测试机器的SN号),并且覆盖apk模式以允许写入外部存储.即让com.android.cts.verifier  拥有读写操作.  注意:我们一般把apk 下载到手机的时候 都是把相机以及其他所有权限都打开. 这两个是不一样的. 相机权限的打开是避免无法调用相机导致的fail. 而这边的读写操作主要是给apk 的各种服务开的.  使其能正常下发命令.

```python
  device_id = get_device_serial_number('dut', config_file_contents)
  # Enable external storage on DUT to send summary report to CtsVerifier.apk
  enable_external_storage(device_id)
```

然后获取测试图表的sn号. 如果TEST_KEY_TABLET 存在在对应的TestBeds里,则获取到table_id,否则为None

```python
TEST_KEY_TABLET = 'tablet'
...

config_file_test_key = config_file_contents['TestBeds'][0]['Name'].lower()
  if TEST_KEY_TABLET in config_file_test_key:
    tablet_id = get_device_serial_number('tablet', config_file_contents)
  else:
    tablet_id = None
```

获取马达舵机的控制通道

```python
  testing_sensor_fusion_with_controller = False
  if TEST_KEY_SENSOR_FUSION in config_file_test_key:
    if test_params_content['rotator_cntl'].lower() in VALID_CONTROLLERS:
      testing_sensor_fusion_with_controller = True
```

预加载场景，如果cmd line没有指出场景

```python
  # Prepend 'scene' if not specified at cmd line
  for i, s in enumerate(scenes):
    if (not s.startswith('scene') and
        not s.startswith(('sensor_fusion', '<scene-name>'))):
      scenes[i] = f'scene{s}'
```

如果用户没有制定特定的场景会跑测所有的场景.

创建子文件用于保存各个cameraID各个场景

```python
    # A subdir in topdir will be created for each camera_id. All scene test
    # output logs for each camera id will be stored in this subdir.
    # This output log path is a mobly param : LogPath
    cam_id_string = 'cam_id_%s' % (
        camera_id.replace(its_session_utils.SUB_CAMERA_SEPARATOR, '_'))
    mobly_output_logs_path = os.path.join(topdir, cam_id_string)
    os.mkdir(mobly_output_logs_path)
```

以上对config.yml的读取和检索后都重新创建一个yml文件,用于正式的跑测

```
      new_yml_file_name = get_updated_yml_file(config_file_contents)
```



# 2 跑测

上面是一些跑测的文件创建和准备

下面直面跑测的内容.

这个逻辑是用来确定单跑和整跑的逻辑.

如果输入的命令有包含tests/  则是单跑调用. 否则都是整跑，

```python
        if 'tests/' in test:
          cmd = [
              'python3',
              os.path.join(os.environ['CAMERA_ITS_TOP'], test), '-c',
              '%s' % new_yml_file_name
          ]
        else:
          cmd = [
              'python3',
              os.path.join(os.environ['CAMERA_ITS_TOP'], 'tests', s, test),
              '-c',
              '%s' % new_yml_file_name
          ]
```
创建subprocess 用于正式跑测
```python
    for num_try in range(NUM_TRIES):
          # pylint: disable=subprocess-run-check
          with open(MOBLY_TEST_SUMMARY_TXT_FILE, 'w') as fp:
            output = subprocess.run(cmd, stdout=fp)
          # pylint: enable=subprocess-run-check
```

解析mobly log 记录跑测返回的状态(skip/pass/fail),并且记录此结果

大致的跑测逻辑如上概诉，真实挂测的def run(cmd): 在这不累诉

## 2.1 加载场景

下面看这俩个内容:

```
def report_result(device_id, camera_id, results):
def load_scenes_on_tablet(scene, tablet_id):
```

加载对应场景的逻辑很简单.

就是将对应场景下的png全部push进table 图表设备中.

push路径为.该路径 必须要有push 的权限. 就算是市面上的机器，不然没法正确调用出对应场景的图片

```python
_DST_SCENE_DIR = '/mnt/sdcard/Download/'
```



```python
def load_scenes_on_tablet(scene, tablet_id):
  """Copies scenes onto the tablet before running the tests.

  Args:
    scene: Name of the scene to copy image files.
    tablet_id: adb id of tablet
  """
  logging.info('Copying files to tablet: %s', tablet_id)
  scene_dir = os.listdir(
      os.path.join(os.environ['CAMERA_ITS_TOP'], 'tests', scene))
  for file_name in scene_dir:
    if file_name.endswith('.png'):
      src_scene_file = os.path.join(os.environ['CAMERA_ITS_TOP'], 'tests',
                                    scene, file_name)
      cmd = f'adb -s {tablet_id} push {src_scene_file} {_DST_SCENE_DIR}'
      subprocess.Popen(cmd.split())
  time.sleep(LOAD_SCENE_DELAY)
  logging.info('Finished copying files to tablet.')
```
而对于这个函数用于改变场景和check
```
def check_manual_scenes(device_id, camera_id, scene, out_path):
```



## 2.2 上报结果

所有的结果都会记录到

```python
ACTION_ITS_RESULT = 'com.android.cts.verifier.camera.its.ACTION_ITS_RESULT'
```

本质上是通过这个将结果上报给apk. 会在手机目录下看到类似its_camera1_scene0.txt的文件.本质上是把这个文件读写上报给测试apk

```python
def report_result(device_id, camera_id, results):
  """Sends a pass/fail result to the device, via an intent.

  Args:
   device_id: The ID string of the device to report the results to.
   camera_id: The ID string of the camera for which to report pass/fail.
   results: a dictionary contains all ITS scenes as key and result/summary of
            current ITS run. See test_report_result unit test for an example.
  """
  adb = f'adb -s {device_id}'

  # Start ItsTestActivity to receive test results
  cmd = f'{adb} shell am start {ITS_TEST_ACTIVITY} --activity-brought-to-front'
  run(cmd)
  time.sleep(ACTIVITY_START_WAIT)

  # Validate/process results argument
  for scene in results:
    if RESULT_KEY not in results[scene]:
      raise ValueError(f'ITS result not found for {scene}')
    if results[scene][RESULT_KEY] not in RESULT_VALUES:
      raise ValueError(f'Unknown ITS result for {scene}: {results[RESULT_KEY]}')
    if SUMMARY_KEY in results[scene]:
      device_summary_path = f'/sdcard/its_camera{camera_id}_{scene}.txt'
      run('%s push %s %s' %
          (adb, results[scene][SUMMARY_KEY], device_summary_path))
      results[scene][SUMMARY_KEY] = device_summary_path

  json_results = json.dumps(results)
  cmd = (f"{adb} shell am broadcast -a {ACTION_ITS_RESULT} --es {EXTRA_VERSION}"
         f" {CURRENT_ITS_VERSION} --es {EXTRA_CAMERA_ID} {camera_id} --es "
         f"{EXTRA_RESULTS} \'{json_results}\'")
  if len(cmd) > 8000:
    logging.info('ITS command string might be too long! len:%s', len(cmd))
  run(cmd)
```

# 3 测试场景

注意SUB_CAMERA_TESTS 数组保存了 对应场景 对应的测试项目. 但是这个不是最后全部的测试项目。也有部分api 测试存在于android-cts-verifier/CameraITS/tests/its_base_test.py

```python
SUB_CAMERA_TESTS = {
    'scene0': [
        'test_burst_capture',
        'test_jitter',
        'test_metadata',
        'test_read_write',
        'test_sensor_events',
        'test_solid_color_test_pattern',
        'test_unified_timestamps',
    ],
```

另外对于以下这种场景有说明:比如场景1中的1_1,1_2 是分出来

场景2 中的不同人脸图都共有测试项

```python
    'scene2_a': [
        'test_faces',
        'test_num_faces',
    ],
```

\#   scene*_1/2/... are same scene split to load balance run times for scenes

\#   scene*_a/b/... are similar scenes that share one or more tests

# 4 ITS apK 代码

apk代码路径如下:

```
cts/apps/CtsVerifier/src/com/android/cts/verifier/camera
目录结构如下:和camera有关的有如下的测试.
├── bokeh
│   └── CameraBokehActivity.java
├── flashlight
│   └── CameraFlashlightActivity.java
├── formats
│   └── CameraFormatsActivity.java
├── fov
│   ├── CalibrationPreferenceActivity.java
│   ├── CameraPreviewView.java
│   ├── CtsTestHelper.java
│   ├── DetermineFovActivity.java
│   ├── PhotoCaptureActivity.java
│   ├── SelectableResolution.java
│   └── Size.java
├── intents
│   ├── CameraContentJobService.java
│   └── CameraIntentsActivity.java
├── its
│   ├── ItsException.java
│   ├── ItsSerializer.java
│   ├── ItsService.java
│   ├── ItsTestActivity.java
│   ├── ItsUtils.java
│   ├── Logt.java
│   └── StatsImage.java
├── orientation
│   └── CameraOrientationActivity.java
├── OWNERS
├── performance
│   ├── CameraPerformanceActivity.java
│   └── CameraTestInstrumentation.java
└── video
    └── CameraVideoActivity.java
```

我们重点看its 目录下的代码

```shell
├── its
│   ├── ItsException.java   #记录异常的接口
│   ├── ItsSerializer.java  #serialize  序列化解析json 对象
│   ├── ItsService.java  #its 围绕这个服务进行交互的
│   ├── ItsTestActivity.java #主要测试main
│   ├── ItsUtils.java #检索的文件 获取图片格式等操作都在此文件内完成
│   ├── Logt.java #传递log msg
│   └── StatsImage.java #load ctsverifier_jni 
```
ItsTestActivity.java # 下列代码是两年前谷歌加入的为了修复tests/scene0/test_metadata.py 脚本的问题而加入的默认语言检查.所以我们使用cts-verifier apk的时候 系统的默认要美国地区的英语.不然无法进行测试
```java
        // Default locale must be set to "en-us"
        Locale locale = Locale.getDefault();
        if (!Locale.US.equals(locale)) {
            String toastMessage = "Unsupported default language " + locale + "! " +
                    "Please switch the default language to English (United States) in " +
                    "Settings > Language & input > Languages";
            Toast.makeText(ItsTestActivity.this, toastMessage, Toast.LENGTH_LONG).show();
            ItsTestActivity.this.getReportLog().setSummary(
                    "FAIL: Default language is not set to " + Locale.US,
                    1.0, ResultType.NEUTRAL, ResultUnit.NONE);
            setTestResultAndFinish(false);
        }
```
这是当时谷歌的commit.  感觉谷歌的操作真的是全是堆积这种bug. 以前就没有中文语言不支持一说.
```
CtsVerifier: Fail Camera ITS in case of unsupported locale

Per CTS specification Camera ITS must only be run with default
language set to English (United States). Check this and directly
fail during ITS activity initialization and when processing ITS
commands.
The python script handling of socket errors must also be modified
to appropriately notify users about this error condition.

Bug: 149922652
Test: Manual using CtsVerifier app,
python tests/scene0/test_metadata.py

Change-Id: I3a2833e2fbc8e8863f5dd13600d39e7d77cea438
```

ITS apk部分的源码建议去谷歌官网看.

