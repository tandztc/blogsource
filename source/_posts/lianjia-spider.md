---
title: 链家二手房成交爬虫
date: 2017-05-26 20:04:41
tags:
  - python
  - toolbox
  - web-crawler
categories: 稀有
---

逐渐有了买房的想法，研究一段时间之后，发现各大网站都没有给出一个完整的房价统计数据和走势。好在链家网的每一条二手房成交记录都有对应的网页。如果能把每一套房的成交信息（面积，单价，总价，成交时间，户型，版块，行政区等等）拿到，存入db或者excel中，那么要分析历史走势就容易多了。此程序就是能够抓取链家网二手房成交记录的爬虫
<!-- more -->

## 获取所有成交记录url
以成都为例，打开https://cd.lianjia.com/chengjiao/ 可以看到所有已经成交的二手房。每一页显示30个记录，点击记录的标题或者图片就能来到此房源的详细页面。处理完一页翻页即可，似乎很顺利。然而事情并不像这么简单。因为此处最多有第100页的按钮，即使从url中手动输入比100更大的数字，比如（https://cd.lianjia.com/chengjiao/pg101/ ）也是返回第100页的内容。也就是说只能爬到3000套成交记录，比页面上显示的十几万套记录少多了。

观察发现，这些成交记录可以按照行政区进行分类浏览。点击[`锦江区`](https://cd.lianjia.com/chengjiao/jinjiang/)后，  列出了锦江区的成交记录，共1w多套，还是超过了3000，不过已经减少了。更进一步，可以选择锦江区的下的版块分类，比如[`川师`](https://cd.lianjia.com/chengjiao/chuanshi/)。这时候，成交记录就只剩下700多了。因此，只需要获取所有的行政区，再获取行政区的所有版块。根据版块来翻页，得到该版块的成交记录，最后汇总起来，就是链家成都的所有历史成交记录了。担心有漏网之鱼，可以在爬去每个版块的时候，判断该版块的成交房源数量是否大于3000，如果大于再想办法解决

用Python实现，`requests`获取页面，`lxml`处理页面，`xpath`语法定位所需元素。代码如下
``` python
def spider_get_xml(url):
    try:
        print("processing {}".format(url))
        time.sleep(random.uniform(0.5, 1.6))
        ret = requests.get(url, headers={'User-Agent': ua.chrome}, cookies=get_cookie(), timeout=5)
        return etree.HTML(ret.content)
    except Exception as e:
        return etree.HTML("<None/>")


def get_regions(suffix):
    while True:
        page = spider_get_xml(root_url + suffix)
        hrefs = page.xpath("//div[@data-role='ershoufang']//a")

        regions = {}
        for href in hrefs:
            regions[href.attrib['href']] = href.text
        if len(regions) > 0:
            return regions


def get_section(url):
    while True:
        page = spider_get_xml(root_url + url)
        hrefs = page.xpath("//div[@data-role='ershoufang']/div[2]//a")

        sections = {}
        for href in hrefs:
            sections[href.attrib['href']] = href.text
        if len(sections) > 0:
            return sections
```

## 流量异常，被系统制裁了！
经过多次调试，爬虫终于可以顺利地展开工作了。为了防止被发现，我设置了每次web请求采用随机的useragent，并且每次请求过后随机暂停1-2秒的时间。慢点无所谓，只要能爬出所有结果，48小时都OK。然而事情还是没有想象的这么简单。仅仅过了几分钟，爬虫在获取的网页里面就找不到目标元素。我意识到是被对面识别了。用chrome打开链家一看，果然被重定向到一个网页，提示流量异常，争取处理验证问题之后可继续访问。

用chrome的开发者模式能够看到，回答验证码之后，访问链家网时cookie中增加了一些信息。因此只需把这些信息加入爬虫，就能继续抓数据了。
``` python
def get_cookie():
    with open('cookie', 'r') as f:
        cookies = {}
        for line in f.read().split(';'):
            name, value = line.strip().split('=', 1)  # 1代表只分割一次
            cookies[name] = value
        return cookies

```
这并非万事大吉，即使使用了cookie，连续访问大约十几分钟以后，也会继续警告流量异常。我想了个最简单的方法来解决：获取的页面没有目标元素时，自动用chrome打开此页面，并且暂停程序。通常这页面会直接重定向到验证页面。手动通过验证，让爬虫程序继续，就度过这段危机啦
``` python
def process_traded_section(url):
    while True:
        page = spider_get_xml(root_url + url)
        data = page.xpath("//*[@class='total fl']/span")
        if len(data) > 0:
            total_num = int(data[0].text)
            page_num = int(math.ceil(total_num / 30))
            # print('{0} has {1} house(s)'.format(section_name, total_num))
            break

    for i in range(1, page_num + 1):
        process_traded_page(root_url + url + "pg{0}/".format(i))


def process_traded_page(url):
    # record_url(url)
    # return
    index = 16
    while index > 0:
        page = spider_get_xml(url)
        data = page.xpath("//*[@class='total fl']/span")
        if len(data) > 0:
            break
        index -= 1
        open_browser(url)
        input("看看弹出的浏览器信息，处理之后按任意键继续爬")

    if index == 0:
        print("get stuck in page {}", url)
        return
    house_refs = page.xpath("//ul[@class='listContent']/li")
    for house_ref in house_refs:
        process_traded_house(house_ref)


def process_traded_house(house_ref):
    try:
        title = house_ref.xpath("div[@class='info']/div[@class='title']/a")[0]
        house_url = title.attrib['href']
        xiaoqu, huxing, area = title.text.split(' ')
        area = area[:-2]
        deal_date = house_ref.xpath(".//div[@class='dealDate']")[0].text
        price = float(house_ref.xpath(".//div[@class='totalPrice']/span")[0].text)
        unit_price = float(house_ref.xpath(".//div[@class='unitPrice']/span")[0].text)
        info = "{0}\t{1:g}\t{2:g}\t{3}\t{4}\t{5}\t{6}\t{7}".format(house_url, price, unit_price, huxing, area,
                                                                   current_section_name, xiaoqu, deal_date)
        record_data(info)
    except Exception as e:
        pass
```

## 任务完成
最终，经过几个小时的一边coding一边解决验证码，终于拿到了成都市和杭州市的十几万套链家二手房成交记录。导入excel制表分析之

PS：
* 链家对爬虫还算友好，没有封ip，弹验证码的频率也不算高
* 总算明白网上找人专门点验证码的兼职有何作用了，真是产业啊
* 北京成交60w套，成都12w，杭州1w5。看来链家在杭州起步比较晚

### 完整源码
https://github.com/tandztc/maifang
### 使用方法
* onsell功能还未完成
```
usage: ershoufang.py [-h] [-c CITY] [-t {onsell,traded}]

optional arguments:
  -h, --help            show this help message and exit
  -c CITY, --city CITY  城市拼音首字母，bj for 北京
  -t {onsell,traded}, --type {onsell,traded}
                        在售房源或者历史成交记录
```
