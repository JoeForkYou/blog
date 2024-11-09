---
title: Linux 命令之AWK过滤提取需要的信息
tags: linux
abbrlink: 30124
date: 2024-11-10 03:07:29
categories: linux
---
# 1 概述
AWK是一个优良的文本处理工具，Linux及Unix环境中现有的功能最强大的数据处理引擎之一。这种编程及数据操作语言（其名称得自于它的创始人阿尔佛雷德·艾侯、彼得·温伯格和布莱恩·柯林汉姓氏的首个字母）的最大功能取决于一个人所拥有的知识。awk经过改进生成的新的版本nawk,gawk，现在默认linux系统下日常使用的是gawk，用命令可以查看正在应用的awk的来源（ls -l /bin/awk ）

# 2 基本用法

```bash
awk '{pattern + action}' <file>
```

pattern表示在数据中要查找的内容

action是要执行的一系列的命令

awk 通过指定分隔符，将一行分为多个字段，依次用 $1、$2 ... $n 表示第一个字段、第二个字段... 第n个字段。

举例有以下一个文件。我们已经知道的格式如下。想过滤的PSS和RSS之后的文件，对应的字段是3和6。通过以下命令即可过滤出第3个字段和第6个字段的内容。

```
           TOTAL PSS:   102206            TOTAL RSS:   127132      TOTAL SWAP (KB):        0
           TOTAL PSS:   102438            TOTAL RSS:   127364      TOTAL SWAP (KB):        0
           TOTAL PSS:   102494            TOTAL RSS:   127420      TOTAL SWAP (KB):        0
           TOTAL PSS:   102578            TOTAL RSS:   127504      TOTAL SWAP (KB):        0
           TOTAL PSS:   102610            TOTAL RSS:   127536      TOTAL SWAP (KB):        0
           TOTAL PSS:   102558            TOTAL RSS:   127484      TOTAL SWAP (KB):        0
           TOTAL PSS:   102378            TOTAL RSS:   127304      TOTAL SWAP (KB):        0
           TOTAL PSS:   102594            TOTAL RSS:   127520      TOTAL SWAP (KB):        0
           TOTAL PSS:   102554            TOTAL RSS:   127480      TOTAL SWAP (KB):        0
           TOTAL PSS:   102262            TOTAL RSS:   127188      TOTAL SWAP (KB):        0
```

```
awk '{print $3, $6}' hal_PSS.txt
```

## 2.1 分隔符

awk默认分割符是空格和制表符,上面的例子中,若希望把逗号去掉则加 -F即可

```
awk -F ':|,' '{print $3 $6}' hal_PSS.txt
```

这里制定冒号（:）和逗号（,）作为作为分割符号

## 2.2 条件判断

将第三个字段满足小于102262的数字真与否打印出来

```
awk '{print $3<102262}' hal_PSS.txt
```
打印结果为

```shell
1
0
0
0
0
0
0
0
0
0
```

将第三个字段满足小于102262的那一行的信息打印出来

```
awk '$3 <102262 {print $0}' hal_PSS.txt
```

打印结果如下

```
TOTAL PSS:   102206            TOTAL RSS:   127132      TOTAL SWAP (KB):        0
```

## 2.3 统计计算

### 最大值

```
awk 'BEGIN {max=0} {if($3>max) max=$3} END {print "max PSS:", max}' hal_PSS.txt
```

### 最小值

```
awk 'BEGIN {min=102262} {if($3<min) min=$3} END {print "min PSS:", min}' hal_PSS.txt
```

### 平均值

```
awk 'BEGIN {sum=0} {sum+=$3} END {print "mean steer:", sum/NR}' hal_PSS.txt
```


