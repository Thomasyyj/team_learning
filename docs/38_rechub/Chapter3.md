# Task3:离线物料系统的构建

![](material/ch1-overview.png)

本节中我们主要来了解线下部分，离线物料系统是如何构建出来的。

## 1.离线流程总览

- 物料爬取

  主要使用scrapy对新闻（静态数据）进行爬取，存放于MongoDB的SinaNews数据库中。

- 物料画像构建

  - 更新当天新闻动态画像，将用户对前一天新闻的交互（阅读、点赞和收藏等）存入Redis中（动态数据）
  - 将静态数据和动态数据分别存入对应的Redis中。

- 用户画像构建

  - 用户通过前端注册页面注册后，将用户信息存入MySQL的用户注册信息表（register_user）中；
  - 阅读、点赞及收藏等行为会被记录，这些数据存入MySQL的用户阅读信息表（user_read）、用户点赞信息表（user_likes）和用户收藏信息表（user_collections）；
  - 将当天的新注册用户基本信息及其行为数据构造用户画像，存入MongoDB中的`UserProtrai`集合中。

- 整合画像

  根据物料画像和用户画像整合，构建自动化推荐流程

## 2. 物料爬取部分详解

项目整体模块如下：

```
sinanews--------------------------------------项目python模块
+---__init__.py-------------------------------模块初始化文件
+---items.py----------------------------------items文件，用于定义类对象
+---middlewares.py----------------------------中间件，用于配置请求头、代理、cookie、会话维持等
+---pipelines.py------------------------------管道文件，用于将爬取的数据进行持久化存储
+---run.py------------------------------------单独运行项目的脚本，用于自动化爬取
+---settings.py-------------------------------配置文件，用于配置数据库
+---spiders-----------------------------------爬取新闻模块
|   +---__init__.py---------------------------爬取新闻模块初始化文件
|   +---sina.py-------------------------------爬取新浪新闻的具体逻辑，解析网页内容
scrapy.cfg------------------------------------项目配置文件
```

### 2.1 scrapy基础

- scrapy项目结构：

  默认情况下，所有scrapy项目的项目结构都是相似的，在指定目录对应的命令行中输入如下命令，就会在当前目录创建一个scrapy项目

  ```
  scrapy startproject myproject
  ```

  项目的目录结构如下：

  ```
  myproject--------------------------------------项目python模块
  +---__init__.py-------------------------------模块初始化文件
  +---items.py----------------------------------items文件，用于定义类对象
  +---middlewares.py----------------------------中间件，用于配置请求头、代理、cookie、会话维持等
  +---pipelines.py------------------------------管道文件，用于将爬取的数据进行持久化存储
  +---settings.py-------------------------------配置文件，用于配置数据库
  +---spiders-----------------------------------爬取新闻模块
  |   +---__init__.py---------------------------爬取新闻模块初始化文件
  scrapy.cfg------------------------------------项目配置文件
  ```

- Spider:

  spider是定义一个特定站点（或一组站点）如何被抓取的类，包括如何执行抓取（即跟踪链接）以及如何从页面中提取结构化数据（即抓取项）。换言之，spider是为特定站点（或者在某些情况下，一组站点）定义爬行和解析页面的自定义行为的地方。

  爬行器是自己定义的类，Scrapy使用它从一个网站(或一组网站)中抓取信息。它们必须继承 `Spider` 并定义要做出的初始请求，可选的是如何跟随页面中的链接，以及如何解析下载的页面内容以提取数据。

  对于spider来说，抓取周期是这样的：

  1. 首先生成对第一个URL进行爬网的初始请求，然后指定一个回调函数，该函数使用从这些请求下载的响应进行调用。要执行的第一个请求是通过调用 `start_requests()` 方法，该方法(默认情况下)生成 `Request` 中指定的URL的 `start_urls` 以及 `parse` 方法作为请求的回调函数。
  2. 在回调函数中，解析响应(网页)并返回 [item objects](https://www.osgeo.cn/scrapy/topics/items.html#topics-items) ， `Request` 对象，或这些对象的可迭代。这些请求还将包含一个回调(可能相同)，然后由Scrapy下载，然后由指定的回调处理它们的响应。
  3. 在回调函数中，解析页面内容，通常使用 [选择器](https://www.osgeo.cn/scrapy/topics/selectors.html#topics-selectors) （但您也可以使用beautifulsoup、lxml或任何您喜欢的机制）并使用解析的数据生成项。
  4. 最后，从spider返回的项目通常被持久化到数据库（在某些 [Item Pipeline](https://www.osgeo.cn/scrapy/topics/item-pipeline.html#topics-item-pipeline) ）或者使用 [Feed 导出](https://www.osgeo.cn/scrapy/topics/feed-exports.html#topics-feed-exports) .

  下面是官网给出的Demo:

  ```python
  import scrapy
  
  class QuotesSpider(scrapy.Spider):
      name = "quotes" # 表示一个spider 它在一个项目中必须是唯一的，不同的spider不能被设置相同的名称。
  	
      # 必须返回请求的可迭代(您可以返回请求列表或编写生成器函数)，spider将从该请求开始爬行。后续请求将从这些初始请求中相继生成。
      def start_requests(self):
          urls = [
              'http://quotes.toscrape.com/page/1/',
              'http://quotes.toscrape.com/page/2/',
          ]
          for url in urls:
              yield scrapy.Request(url=url, callback=self.parse) # 注意，这里callback调用了下面定义的parse方法
  	
      # 将被调用以处理为每个请求下载的响应的方法。Response参数是 TextResponse 它保存页面内容，并具有进一步有用的方法来处理它。
      def parse(self, response):
          # 下面是直接从response中获取内容，为了更方便的爬取内容，后面会介绍使用selenium来模拟人用浏览器，并且使用对应的方法来提取我们想要爬取的内容
          page = response.url.split("/")[-2]
          filename = f'quotes-{page}.html'
          with open(filename, 'wb') as f:
              f.write(response.body)
          self.log(f'Saved file {filename}')
  ```

- Xpath:XPath 是一门在 XML 文档中查找信息的语言，XPath 可用来在 XML 文档中对元素和属性进行遍历。在爬虫的时候使用xpath来选择我们想要爬取的内容是非常方便的，这里提一下xpath中需要掌握的重点：
  1. **xpath路径表达式：**XPath 使用路径表达式来选取 XML 文档中的节点或者节点集。这些路径表达式和我们在常规的电脑文件系统中看到的表达式非常相似。节点是通过沿着路径 (path) 或者步 (steps) 来选取的。
  2. **了解如何使用xpath语法选取我们想要的内容，所以需要熟悉xpath的基本语法**

### 2.2 新闻爬取实战

### 2.2.1 环境准备：

需要的环境：**MongoDB数据库，python中scrapy, pymongo包**

### 2.2.2 项目运行基本逻辑：

1. 每天定时从新浪新闻网站上爬取新闻数据存储到mongodb数据库中，并且需要监控每天爬取新闻的状态（比如某天爬取的数据特别少可能是哪里出了问题，需要进行排查）
2. 为了防止重复，仅爬取当天日期的新闻，并加入去重检查的代码
3. 实现三个核心功能，分别是爬虫（sina.py），抽取数据的规范化字段以及定义（items.py），把数据写入数据库（pipelines.py）

### 2.2.3 实战部分（给出核心步骤和代码）：

1. 创建scrapy项目

   ```python
   scrapy startproject sinanews
   ```

2. 实现抽取规范字段items.py

   ```python
   # Define here the models for your scraped items
   #
   # See documentation in:
   # https://docs.scrapy.org/en/latest/topics/items.html
   
   import scrapy
   from scrapy import Item, Field
   
   # 定义新闻数据的字段
   class SinanewsItem(scrapy.Item):
       """数据格式化，数据不同字段的定义
       """
       title = Field() # 新闻标题
       ctime = Field() # 新闻发布时间
       url = Field() # 新闻原始url
       raw_key_words = Field() # 新闻关键词（爬取的关键词）
       content = Field() # 新闻的具体内容
       cate = Field() # 新闻类别
   ```

3. 实现爬虫sina.py

   注：这里爬取的网址为https://news.sina.com.cn/roll/#pageid=153&lid=2509&k=&num=50&page=1

   ```python
   # -*- coding: utf-8 -*-
   import re
   import json
   import random
   import scrapy
   from scrapy import Request
   from ..items import SinanewsItem
   from datetime import datetime
   
   
   class SinaSpider(scrapy.Spider):
       # spider的名字
       name = 'sina_spider'
   
       def __init__(self, pages=None):
           super(SinaSpider).__init__()
   
           self.total_pages = int(pages)
           # base_url 对应的是新浪新闻的简洁版页面，方便爬虫，并且不同类别的新闻也很好区分
           self.base_url = 'https://feed.mix.sina.com.cn/api/roll/get?pageid=153&lid={}&k=&num=50&page={}&r={}'
           # lid和分类映射字典
           self.cate_dict = {
               "2510":  "国内",
               "2511":  "国际",
               "2669":  "社会",
               "2512":  "体育",
               "2513":  "娱乐",
               "2514":  "军事",
               "2515":  "科技",
               "2516":  "财经",
               "2517":  "股市",
               "2518":  "美股"
           }
   
       def start_requests(self):
           """返回一个Request迭代器
           """
           # 遍历所有类型的新闻
           for cate_id in self.cate_dict.keys():
               for page in range(1, self.total_pages + 1):
                   lid = cate_id
                   # 这里就是一个随机数，具体含义不是很清楚
                   r = random.random()
                   # cb_kwargs 是用来向解析函数parse中传递参数的
                   yield Request(self.base_url.format(lid, page, r), callback=self.parse, cb_kwargs={"cate_id": lid})
       
       def parse(self, response, cate_id):
           """解析网页内容，并提取网页中需要的内容
           """
           json_result = json.loads(response.text) # 将请求回来的页面解析成json
           # 提取json中我们想要的字段
           # json使用get方法比直接通过字典的形式获取数据更方便，因为不需要处理异常
           data_list = json_result.get('result').get('data')
           for data in data_list:
               item = SinanewsItem()
   
               item['cate'] = self.cate_dict[cate_id]
               item['title'] = data.get('title')
               item['url'] = data.get('url')
               item['raw_key_words'] = data.get('keywords')
   
               # ctime = datetime.fromtimestamp(int(data.get('ctime')))
               # ctime = datetime.strftime(ctime, '%Y-%m-%d %H:%M')
   
               # 保留的是一个时间戳
               item['ctime'] = data.get('ctime')
   
               # meta参数传入的是一个字典，在下一层可以将当前层的item进行复制
               yield Request(url=item['url'], callback=self.parse_content, meta={'item': item})
       
       def parse_content(self, response):
           """解析文章内容
           """
           item = response.meta['item']
           content = ''.join(response.xpath('//*[@id="artibody" or @id="article"]//p/text()').extract())
           content = re.sub(r'\u3000', '', content)
           content = re.sub(r'[ \xa0?]+', ' ', content)
           content = re.sub(r'\s*\n\s*', '\n', content)
           content = re.sub(r'\s*(\s)', r'\1', content)
           content = ''.join([x.strip() for x in content])
           item['content'] = content
           yield item 
   ```

4. 实现数据持久化piplines.py

   这里需要注意的就是实现SinanewsPipeline类的时候，里面很多方法都是固定的，不是随便写的，不同的方法又不同的功能，这个可以参考scrapy官方文档。

   ```python
   # Define your item pipelines here
   #
   # Don't forget to add your pipeline to the ITEM_PIPELINES setting
   # See: https://docs.scrapy.org/en/latest/topics/item-pipeline.html
   # useful for handling different item types with a single interface
   import time
   import datetime
   import pymongo
   from pymongo.errors import DuplicateKeyError
   from sinanews.items import SinanewsItem
   from itemadapter import ItemAdapter
   
   
   # 新闻item持久化
   class SinanewsPipeline:
       """数据持久化：将数据存放到mongodb中
       """
       def __init__(self, host, port, db_name, collection_name):
           self.host = host
           self.port = port
           self.db_name = db_name
           self.collection_name = collection_name
   
       @classmethod    
       def from_crawler(cls, crawler):
           """自带的方法，这个方法可以重新返回一个新的pipline对象，并且可以调用配置文件中的参数
           """
           return cls(
               host = crawler.settings.get("MONGO_HOST"),
               port = crawler.settings.get("MONGO_PORT"),
               db_name = crawler.settings.get("DB_NAME"),
               # mongodb中数据的集合按照日期存储
               collection_name = crawler.settings.get("COLLECTION_NAME") + \
                   "_" + time.strftime("%Y%m%d", time.localtime())
           )
   
       def open_spider(self, spider):
           """开始爬虫的操作，主要就是链接数据库及对应的集合
           """
           self.client = pymongo.MongoClient(self.host, self.port)
           self.db = self.client[self.db_name]
           self.collection = self.db[self.collection_name]
           
       def close_spider(self, spider):
           """关闭爬虫操作的时候，需要将数据库断开
           """
           self.client.close()
   
       def process_item(self, item, spider):
           """处理每一条数据，注意这里需要将item返回
           注意：判断新闻是否是今天的，每天只保存当天产出的新闻，这样可以增量的添加新的新闻数据源
           """
           if isinstance(item, SinanewsItem):
               try:
                   # TODO 物料去重逻辑，根据title进行去重，先读取物料池中的所有物料的title然后进行去重
   
                   cur_time = int(item['ctime'])
                   str_today = str(datetime.date.today())
                   min_time = int(time.mktime(time.strptime(str_today + " 00:00:00", '%Y-%m-%d %H:%M:%S')))
                   max_time = int(time.mktime(time.strptime(str_today + " 23:59:59", '%Y-%m-%d %H:%M:%S')))
                   if cur_time > min_time and cur_time <= max_time:
                       self.collection.insert(dict(item))
               except DuplicateKeyError:
                   """
                   说明有重复
                   """
                   pass
           return item
   ```

5. 配置文件setting.py

   ```python
   # Scrapy settings for sinanews project
   #
   # For simplicity, this file contains only settings considered important or
   # commonly used. You can find more settings consulting the documentation:
   #
   #     https://docs.scrapy.org/en/latest/topics/settings.html
   #     https://docs.scrapy.org/en/latest/topics/downloader-middleware.html
   #     https://docs.scrapy.org/en/latest/topics/spider-middleware.html
   
   from typing import Collection
   
   BOT_NAME = 'sinanews'
   
   SPIDER_MODULES = ['sinanews.spiders']
   NEWSPIDER_MODULE = 'sinanews.spiders'
   
   
   # Crawl responsibly by identifying yourself (and your website) on the user-agent
   #USER_AGENT = 'sinanews (+http://www.yourdomain.com)'
   
   # Obey robots.txt rules
   ROBOTSTXT_OBEY = True
   
   # Configure maximum concurrent requests performed by Scrapy (default: 16)
   #CONCURRENT_REQUESTS = 32
   
   # Configure a delay for requests for the same website (default: 0)
   # See https://docs.scrapy.org/en/latest/topics/settings.html#download-delay
   # See also autothrottle settings and docs
   # DOWNLOAD_DELAY = 3
   # The download delay setting will honor only one of:
   #CONCURRENT_REQUESTS_PER_DOMAIN = 16
   #CONCURRENT_REQUESTS_PER_IP = 16
   
   # Disable cookies (enabled by default)
   #COOKIES_ENABLED = False
   
   # Disable Telnet Console (enabled by default)
   #TELNETCONSOLE_ENABLED = False
   
   # Override the default request headers:
   #DEFAULT_REQUEST_HEADERS = {
   #   'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8',
   #   'Accept-Language': 'en',
   #}
   
   # Enable or disable spider middlewares
   # See https://docs.scrapy.org/en/latest/topics/spider-middleware.html
   #SPIDER_MIDDLEWARES = {
   #    'sinanews.middlewares.SinanewsSpiderMiddleware': 543,
   #}
   
   # Enable or disable downloader middlewares
   # See https://docs.scrapy.org/en/latest/topics/downloader-middleware.html
   #DOWNLOADER_MIDDLEWARES = {
   #    'sinanews.middlewares.SinanewsDownloaderMiddleware': 543,
   #}
   
   # Enable or disable extensions
   # See https://docs.scrapy.org/en/latest/topics/extensions.html
   #EXTENSIONS = {
   #    'scrapy.extensions.telnet.TelnetConsole': None,
   #}
   
   # Configure item pipelines
   # See https://docs.scrapy.org/en/latest/topics/item-pipeline.html
   # 如果需要使用itempipline来存储item的话需要将这段注释打开
   ITEM_PIPELINES = {
      'sinanews.pipelines.SinanewsPipeline': 300,
   }
   
   # Enable and configure the AutoThrottle extension (disabled by default)
   # See https://docs.scrapy.org/en/latest/topics/autothrottle.html
   #AUTOTHROTTLE_ENABLED = True
   # The initial download delay
   #AUTOTHROTTLE_START_DELAY = 5
   # The maximum download delay to be set in case of high latencies
   #AUTOTHROTTLE_MAX_DELAY = 60
   # The average number of requests Scrapy should be sending in parallel to
   # each remote server
   #AUTOTHROTTLE_TARGET_CONCURRENCY = 1.0
   # Enable showing throttling stats for every response received:
   #AUTOTHROTTLE_DEBUG = False
   
   # Enable and configure HTTP caching (disabled by default)
   # See https://docs.scrapy.org/en/latest/topics/downloader-middleware.html#httpcache-middleware-settings
   #HTTPCACHE_ENABLED = True
   #HTTPCACHE_EXPIRATION_SECS = 0
   #HTTPCACHE_DIR = 'httpcache'
   #HTTPCACHE_IGNORE_HTTP_CODES = []
   #HTTPCACHE_STORAGE = 'scrapy.extensions.httpcache.FilesystemCacheStorage'
   
   MONGO_HOST = "127.0.0.1"
   MONGO_PORT = 27017
   DB_NAME = "SinaNews"
   COLLECTION_NAME = "news"
   ```

6. 监控脚本monitor_news.py

   ```python
   # -*- coding: utf-8 -*-
   import sys, time
   import pymongo
   import scrapy 
   from sinanews.settings import MONGO_HOST, MONGO_PORT, DB_NAME, COLLECTION_NAME
   
   if __name__ == "__main__":
       news_num = int(sys.argv[1])
       time_str = time.strftime("%Y%m%d", time.localtime())
   
       # 实际的collection_name
       collection_name = COLLECTION_NAME + "_" + time_str
       
       # 链接数据库
       client = pymongo.MongoClient(MONGO_HOST, MONGO_PORT)
       db = client[DB_NAME]
       collection = db[collection_name]
   
       # 查找当前集合中所有文档的数量
       cur_news_num = collection.count()
   
       print(cur_news_num)
       if cur_news_num < news_num:
           print("the news nums of {}_{} collection is less then {}".\
               format(COLLECTION_NAME, time_str, news_num))
   ```

7. 运行脚本run_scrapy_sina.sh

   ```python
   # -*- coding: utf-8 -*-
   """
   新闻爬取及监控脚本
   """
   
   # 设置python环境
   python="/home/recsys/miniconda3/envs/news_rec_py3/bin/python"
   
   # 新浪新闻网站爬取的页面数量
   page="1"
   min_news_num="1000" # 每天爬取的新闻数量少于500认为是异常
   
   # 爬取数据
   scrapy crawl sina_spider -a pages=${page}  
   if [ $? -eq 0 ]; then
       echo "scrapy crawl sina_spider --pages ${page} success."
   else   
       echo "scrapy crawl sina_spider --pages ${page} fail."
   fi
   
   # 检查今天爬取的数据是否少于min_news_num篇文章，这里也可以配置邮件报警
   python monitor_news.py ${min_news_num}
   if [ $? -eq 0 ]; then
       echo "run python monitor_news.py success."
   else   
       echo "run python monitor_news.py fail."
   fi
   ```

   

