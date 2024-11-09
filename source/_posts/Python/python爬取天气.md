---
title: python爬取天气
tags: python
abbrlink: 26103
date: 2024-11-10 01:07:35
categories: python
---

# 1 安装使用

VSCode中安装相关的插件就可以配合chatgpt来实现一些简单的爬虫操作，只能应对比较简单的场景.
安装插件
chatGPT-ChatMoss

# 2 实例

给AI发送
python 爬取无锡七日天气
回复如下:
要爬取无锡七日天气，可以使用Python中的第三方库 requests 和 Beautiful Soup。以下是详细步骤：

1. 导入必要的库

```python
import requests
from bs4 import BeautifulSoup
```

2. 发送GET请求获取网页内容

```python
url = "http://www.weather.com.cn/weather/101190201.shtml"
response = requests.get(url)
```

3. 解析网页内容

```python
soup = BeautifulSoup(response.content, "html.parser")
```

4. 找到需要的天气信息所在的标签

```python
weather_data = soup.select(".t>li")
```

5. 提取天气信息并输出

```python
for weather in weather_data:
    # 获取日期、天气状况、最高温度和最低温度
    date = weather.select_one(".date").get_text()
    condition = weather.select_one(".wea").get_text()
    high_temperature = weather.select_one(".tem span").get_text()
    low_temperature = weather.select_one(".tem i").get_text()

    # 打印天气信息
    print(date, condition, high_temperature, low_temperature)
```

完整代码如下：

```python
import requests
from bs4 import BeautifulSoup

url = "http://www.weather.com.cn/weather/101190201.shtml"

response = requests.get(url)

soup = BeautifulSoup(response.content, "html.parser")

weather_data = soup.select(".t>li")

for weather in weather_data:
    date = weather.select_one(".date").get_text()
    condition = weather.select_one(".wea").get_text()
    high_temperature = weather.select_one(".tem span").get_text()
    low_temperature = weather.select_one(".tem i").get_text()

    print(date, condition, high_temperature, low_temperature)
```

但是上述实际运行是存在一些小问题的。

打印结果如下: 这个很明显就是date这个数据为None

```powershell
None <p class="wea" title="多云">多云</p> <span>28</span> <i>17℃</i>
None <p class="wea" title="多云">多云</p> <span>27</span> <i>15℃</i>
None <p class="wea" title="多云">多云</p> <span>24</span> <i>12℃</i>
None <p class="wea" title="阴转小雨">阴转小雨</p> <span>20</span> <i>15℃</i>
None <p class="wea" title="阴">阴</p> <span>15</span> <i>13℃</i>
None <p class="wea" title="小雨">小雨</p> <span>19</span> <i>14℃</i>
None <p class="wea" title="小雨转晴">小雨转晴</p> <span>16</span> <i>8℃</i>
```

继续追问机器人:

优化下代码，没有获取到天数，并且这个get_text有报错.

机器人直接回复:

以下是优化后的代码，已解决上述问题：

```python
import requests
from bs4 import BeautifulSoup

url = "http://www.weather.com.cn/weather/101190201.shtml"

response = requests.get(url)

soup = BeautifulSoup(response.content, "html.parser")

weather_data = soup.select(".t li")

for weather in weather_data:
    # 获取日期、天气状况、最高温度和最低温度
    date = weather.select_one("h1").get_text()
    condition = weather.select_one(".wea").get_text()
    high_temperature = weather.select(".tem span")[0].get_text()
    low_temperature = weather.select(".tem i")[0].get_text()

    # 打印天气信息
    print(date, condition, high_temperature, low_temperature)
```


在优化代码时，我们对原有代码进行了以下改进：

- 将CSS选择器中的“>”删除，因为它只会选择直接子元素，导致无法获取到所有的天气信息。
- 由于天数信息并不在 ".date" 标签内，所以将日期信息从该标签中提取出来。在这里，我们使用了 h1 标签来获取日期信息。
- 使用 select() 方法来选取多个元素，并且使用索引 [0] 来定位到需要的元素，因为 .select_one() 方法只能返回单个元素，如果要选取多个元素，需要使用 select() 方法。同时，我们还修改了 ".tem span" 和 ".tem i" 的类选择器，以便正确获取最高温度和最低温度。
- 在使用 get_text() 方法时，需要确保该元素存在并且不为空。如果该元素不存在或为空，会导致 AttributeError 异常。


这一次就非常好用了.

打印出的结果如下:

```powershell
19日（今天） 多云 28 17℃
20日（明天） 多云 27 15℃
21日（后天） 多云 24 12℃
22日（周六） 阴转小雨 20 15℃
23日（周日） 阴 15 13℃
24日（周一） 小雨 19 14℃
25日（周二） 小雨转晴 16 8℃
```

