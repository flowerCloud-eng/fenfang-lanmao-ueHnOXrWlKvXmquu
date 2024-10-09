
### 搜索示例


搜索 `123`，网页地址为：[https://www.virustotal.com/gui/search/123/comments](https://github.com)


[![截图_20241008183426](https://images.cnblogs.com/cnblogs_com/blogs/803846/galleries/2346972/o_241008112644_%E6%88%AA%E5%9B%BE_20241008183426-1728383718279-2.png)](https://github.com)


### 请求接口



```
GET /ui/search?limit=20&relationships%5Bcomment%5D=author%2Citem&query=123 HTTP/1.1
Accept-Encoding: gzip, deflate, br, zstd
Accept-Ianguage: en-US,en;q=0.9,es;q=0.8
Accept-Language: zh-CN,zh;q=0.9
Cache-Control: no-cache
Connection: keep-alive
Cookie: _gid=GA1.2.1662779803.1728383656; _ga=GA1.2.686372046.1728383655; _gat=1; _ga_BLNDV9X2JR=GS1.1.1728383655.1.1.1728383759.0.0.0
DNT: 1
Host: www.virustotal.com
Pragma: no-cache
Referer: https://www.virustotal.com/
Sec-Fetch-Dest: empty
Sec-Fetch-Mode: cors
Sec-Fetch-Site: same-origin
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36
X-Tool: vt-ui-main
X-VT-Anti-Abuse-Header: MTgwNjgyNDI1ODItWkc5dWRDQmlaU0JsZG1scy0xNzI4MzgzNzYxLjMxMg==
accept: application/json
content-type: application/json
sec-ch-ua: "Google Chrome";v="129", "Not=A?Brand";v="8", "Chromium";v="129"
sec-ch-ua-mobile: ?0
sec-ch-ua-platform: "Windows"
x-app-version: v1x304x0

```

注意观察，其中发现参数：`X-VT-Anti-Abuse-Header` 需要逆向，目测是 base64 加密，这个参数的含义也很明确——“反滥用头”。在编写爬虫的代码层的实现时，尽量将 `User-Agent`、`X-Tool`、`x-app-version` 补全，因为它们是网站特有的亦或者常见的反爬识别参数。


值得注意的是，`X-Tool`、`x-app-version` 是固定值，`x-app-version` 需要定期去官网查看一下更新，不更新可能也行，自行测试。


也就是说，目前我们得到如下 Headers：



```
{
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36',
    'X-Tool': 'vt-ui-main',
    'x-app-version': 'v1x304x0',
}

```

### 开始逆向


接下来，逆向 `X-VT-Anti-Abuse-Header`，总体没啥难度，不过这个网站具备反调试措施（无法断点调试、无法从请求的启动器回溯），这个反调试我没有去解决，而是通过直接查找。


搜索 `X-VT-Anti-Abuse-Header`，可得到：


[![截图_20241008190340](https://images.cnblogs.com/cnblogs_com/blogs/803846/galleries/2346972/o_241008112644_%E6%88%AA%E5%9B%BE_20241008190340.png)](https://github.com)


直接进入该 js 文件中：


[![截图_20241008190408](https://images.cnblogs.com/cnblogs_com/blogs/803846/galleries/2346972/o_241008112644_%E6%88%AA%E5%9B%BE_20241008190408.png)](https://github.com):[悠兔机场](https://xinnongbo.com)


可以看到，我们的目标参数由方法 `r.computeAntiAbuseHeader()` 计算而来。全局搜索 `computeAntiAbuseHeader`：


[![截图_20241008190551](https://images.cnblogs.com/cnblogs_com/blogs/803846/galleries/2346972/o_241008112644_%E6%88%AA%E5%9B%BE_20241008190551.png)](https://github.com)


序号1为我们所需的，也就是函数的实现，序号2为该函数的调用，也就是上方的 `X-VT-Anti-Abuse-Header` 来源。进入 1 所在的 js 文件。


[![截图_20241008190849](https://images.cnblogs.com/cnblogs_com/blogs/803846/galleries/2346972/o_241008112644_%E6%88%AA%E5%9B%BE_20241008190849.png)](https://github.com)


可以看到，这个方法真的非常简单，不需要任何回溯和断点，可以直接的手搓出来。这个网站典型的将反爬中的`防君子不防小人`做得很好。言归正传，这个方法：


1. 首先获取当前时间的秒级时间戳
2. 然后生成一个介于 1e10 和 5e14 之间略大的随机数，如果生成的数小于 50，则返回 `"-1"`，否则返回该数的整数部分
3. 最后将生成的随机数、固定字符串 `"ZG9udCBiZSBldmls"`（意为 "不要作弊"）和当前时间戳拼接并进行 `base64` 加密


有趣的一点是：



```
> atob('ZG9udCBiZSBldmls ')
< 'dont be evil'

```

固定的字符串告诉我们不要作恶……这尼玛绝了，哈哈哈哈


### 加密实现



```
# header.py
import base64
import random
import time


def computeAntiAbuseHeader():
    e = time.time()
    n = 1e10 * (1 + random.random() % 5e4)
    raw = f'{n:.0f}-ZG9udCBiZSBldmls-{e:.3f}'
    res = base64.b64encode(raw.encode())
    return res.decode()


if __name__ == '__main__':
    print(computeAntiAbuseHeader())

```

### 结束了吗？


看到这里你以为结束了？oh\~不，还差一点点，尽管你有了上述的分析和实现，你发现你怎么请求都没用！数据还是不给你，为啥？


再次将你的视线挪移到请求接口中：



```
Accept-Ianguage: en-US,en;q=0.9,es;q=0.8
Accept-Language: zh-CN,zh;q=0.9

```

此处有个容易忽略的老6请求头：`Accept-Ianguage`，好了，到此才算结束了，看一下下方完整的代码示例吧。



```
"""
翻页采集实现
2024年9月27日 solved
https://www.virustotal.com/gui/
"""
import time
import requests
import header
from urllib.parse import urlencode, urlparse

base_url = "https://www.virustotal.com/ui/search"
initial_params = {
    "limit": 20,
    "relationships[comment]": "author,item",
    "query": "baidu"
}

proxies = {
    'http': None,
    'https': None
}


def build_url(url, params):  # ☆
    return urlparse(url)._replace(query=urlencode(params)).geturl()


def get_headers():
    return {
        'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/129.0.0.0 Safari/537.36',
        'X-Tool': 'vt-ui-main',
        'X-VT-Anti-Abuse-Header': header.computeAntiAbuseHeader(),
        'x-app-version': 'v1x304x0',
        'accept-ianguage': 'en-US,en;q=0.9,es;q=0.8'
    }


def fetch_data(url):
    response = requests.get(url, headers=get_headers(), proxies=proxies)
    return response.json()


def process_data(data):
    for item in data['data']:
        print(f"ID: {item['id']}, Type: {item['type']}")


# 主循环
next_url = build_url(base_url, initial_params)
while next_url:
    print(f"Fetching: {next_url}")
    json_data = fetch_data(next_url)

    # 检查是否有数据
    if not json_data.get('data'):
        print("No more data.")
        break

    # 处理当前页面的数据
    process_data(json_data)

    # 获取下一页的 URL
    next_url = json_data.get('links', {}).get('next')

    if not next_url:
        print("No more pages.")
        break

    time.sleep(1)

print("Finished fetching all pages.")


```

