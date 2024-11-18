---
title: Kernel加时间戳
tags: Android
categories: Android
abbrlink: 50204
date: 2024-11-18 22:23:59
---
---
添加对应的头文件和包装函数:

```cpp
#include <linux/ktime.h>
long long thermal_get_current_time_ms(void)
{
    long long temp;
    struct timespec64 t;
    ktime_get_ts64(&t);
    temp = (((long long) t.tv_sec) * 1000000 + (long)t.tv_nsec/1000);
    return (temp/1000);
}
```

c中static 受到编译器的影响 禁止使用long long类型.所以debug的时候还是去掉static的申明.
```cpp
long long time_diff,start_timestamp,end_timestamp;

start_timestamp = thermal_get_current_time_ms();

end_timestamp = thermal_get_current_time_ms();
time_diff = end_timestamp - start_timestamp;
printk("[time] timestamp_diff = %lld ms",time_diff);
```

