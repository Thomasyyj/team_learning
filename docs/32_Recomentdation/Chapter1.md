# Task1:熟悉推荐系统的流程与搭建环境

![](material/ch1-overview.png)

整体的推荐系统流程如图所示，整个推荐流程分为离线部分与在线部分，整个流程不具有商用价值，只是对推荐系统的整个流程作熟悉。

## 1. 整理流程

## 1.1 离线部分

### 1.1.1 物料爬取与画像处理

- 爬取物料主要采用scrapy爬虫框架，在每天23点晚上将数据爬取到MongoDB的SinaNews数据库中，代码地址：

  ```
  codes/news_recsys/news_rec_server/materials/news_scrapy/sinanews/run.py
  ```

- 物料画像处理：将当天的新闻内容通过特征画像提取从MongoDB存入Redis中

  动态画像更新：实时更新对新闻的交互（这里主要实现点赞，阅读和收藏行为），存入到Redis中，代码地址：

  ```
  codes/news_recsys/news_rec_server/materials/process_material.py
  ```

  用到的核心代码部分如下:

  ```python
  # 画像处理
  protrail_server = NewsProtraitServer()
  # 处理最新爬取新闻的画像，存入特征库
  protrail_server.update_new_items()
  # 更新新闻动态画像, 需要在redis数据库内容清空之前执行
  protrail_server.update_dynamic_feature_protrail()
  # 生成前端展示的新闻画像，并在mongodb中备份一份
  protrail_server.update_redis_mongo_protrail_data()
  ```

- 前端新闻展示：主要将Redis中的新闻动态画像数据提供给前端进行展示，代码地址：

  ```
  codes/news_recsys/news_rec_server/materials/update_redis.py	
  ```

### 1.1.2 用户与用户画像处理

- 用户注册信息记录：用户通过前端注册页面进行用户注册，用户信息会存入MySQL的用户注册信息表（register_user）中

- 用户行为记录：（阅读、点赞及收藏等行为) 存入MySQL的用户阅读信息表（user_read）、用户点赞信息表（user_likes）和用户收藏信息表（user_collections）

- 用户画像更新：根据当天的新注册用户基本信息及其行为数据构造用户画像，存入MongoDB中的`UserProtrail`，代码地址：

  ````
  materials/process_user.py
  ````

### 1.1.3 热门页列表与推荐页列表

- 