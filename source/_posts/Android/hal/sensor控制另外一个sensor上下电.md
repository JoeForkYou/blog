---
title: sensor控制另外一个sensor上下电
tags:
  - hal
  - camera
categories:
  - hal
  - camera
abbrlink: 53434
date: 2024-11-10 07:08:27
---


# 1 描述

在另外一个sensor驱动中控制另外一个sensor的上电

# 2 打开so

vendor/sprd/modules/libcamera/sensor/sensor\_drv\_u.c

```c++
static cmr_int sensor_drv_ic_identify(struct sensor_drv_context *sensor_cxt,
                                      cmr_u32 sensor_id, cmr_u32 identify_off) {
    
  ...
    struct sensor_drv_lib libPtrsc500cs;
    struct sensor_drv_lib *libPtr = &libPtrsc500cs;
    void *(*sc500cs_power_sequence_for_creepage)(void) = NULL;   
    
  ...
        if((sensor_id == 2) && (0 == strcmp(slot_sensor_info_list[1].sensor_name, "sc500cs_frontdd"))) {
        libPtr->drv_lib_handle = dlopen("libsensor_sc500cs_frontdd.so", RTLD_NOW);
        if (!libPtr->drv_lib_handle) {
            SENSOR_LOGE("sc500cs sensor lib handle failed");
        } else {
            *(void **)&sc500cs_power_sequence_for_creepage = dlsym(libPtr->drv_lib_handle, "sc500cs_power_sequence_for_creepage");
            if (!sc500cs_power_sequence_for_creepage) {
                SENSOR_LOGE("sc500cs_power_sequence_for_creepage open lib function failed");
            } else {
                SENSOR_LOGD("sc500cs_power_sequence_for_creepage before");
                sc500cs_power_sequence_for_creepage();
            }
        }
        dlclose(libPtr->drv_lib_handle);
        libPtr->drv_lib_handle = NULL;
    }
    
}
```

# 3 调用

在打开的so中调用对应的函数

```c++
void sc500cs_power_sequence_for_creepage(void) {
    SENSOR_LOGD("E");
    struct sensor_drv_context sns_cxt;
    struct sensor_drv_context *sensor_cxt = &sns_cxt;
    struct sensor_ic_drv_cxt sensor_drv_cxt;
    struct sensor_ic_drv_cxt *sns_drv_cxt = &sensor_drv_cxt;
    cmr_handle hw_drv_handle = NULL;
    struct hw_drv_init_para input_ptr;
    cmr_int fd_sensor = SENSOR_FD_INIT;

    SENSOR_AVDD_VAL_E dvdd_val = SENSOR_AVDD_1200MV;
    SENSOR_AVDD_VAL_E avdd_val = SENSOR_AVDD_2800MV;
    SENSOR_AVDD_VAL_E iovdd_val = SENSOR_AVDD_1800MV;
    BOOLEAN reset_level = SENSOR_LOW_PULSE_RESET;

    //sensor_clean_info(sensor_cxt);
    input_ptr.sensor_id = 1;
    input_ptr.caller_handle = sensor_cxt;
    fd_sensor = hw_sensor_drv_create(&input_ptr, &hw_drv_handle);
    SENSOR_LOGD("fd sensor=%d, hw_drv_handle->fd_sensor=%d", fd_sensor, hw_drv_handle->fd_sensor);
    if ((SENSOR_FD_INIT == fd_sensor) || (NULL == hw_drv_handle)) {
        SENSOR_LOGE("sc500cs hw sensor drv create error");
    } else {
        sensor_cxt->fd_sensor = fd_sensor;
        sensor_cxt->hw_drv_handle = hw_drv_handle;
        sensor_cxt->sensor_hw_handler = hw_drv_handle;
    }

    sns_drv_cxt->hw_handle = (struct sensor_ic_drv_cxt *)hw_drv_handle;

    hw_sensor_set_reset_level(sns_drv_cxt->hw_handle, reset_level);
    hw_sensor_set_avdd_val(sns_drv_cxt->hw_handle, SENSOR_AVDD_CLOSED);
    hw_sensor_set_dvdd_val(sns_drv_cxt->hw_handle, SENSOR_AVDD_CLOSED);
    hw_sensor_set_iovdd_val(sns_drv_cxt->hw_handle, SENSOR_AVDD_CLOSED);

    usleep(2 * 1000);
    hw_sensor_set_iovdd_val(sns_drv_cxt->hw_handle, iovdd_val);
    usleep(1 * 1000);
    hw_sensor_set_dvdd_val(sns_drv_cxt->hw_handle, dvdd_val);
    usleep(1 * 1000);
    hw_sensor_set_reset_level(sns_drv_cxt->hw_handle, !reset_level);
    hw_sensor_set_avdd_val(sns_drv_cxt->hw_handle, avdd_val);
    usleep(5 * 1000);

    hw_sensor_set_reset_level(sns_drv_cxt->hw_handle, reset_level);
    usleep(1 * 1000);
    hw_sensor_set_dvdd_val(sns_drv_cxt->hw_handle, SENSOR_AVDD_CLOSED);
    usleep(1 * 1000);
    hw_sensor_set_iovdd_val(sns_drv_cxt->hw_handle, SENSOR_AVDD_CLOSED);
    usleep(2 * 1000);

    hw_sensor_drv_delete(hw_drv_handle);
    SENSOR_LOGD("X");
}
```

# 4 说明

这种方式可以直接cat 对应gpio的高低电平以及PMIC的电位变化

(GPIO 实际对应GPIO+偏移64)

例如检查GPIO41则要grep  GPIO 105的信息

```shell
while true ;do
{
adb shell cat /sys/kernel/debug/gpio |grep 105
sleep 1
adb shell cat /sys/kernel/debug/LDO_VDDCAMD1/enable
}
done
&
```

