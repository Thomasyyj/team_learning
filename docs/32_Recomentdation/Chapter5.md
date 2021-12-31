# Task5:推荐系统流程

![](material/ch1-overview.png)

本次任务主要介绍推荐系统的流程构建，主要分为线下部分和线上部分，每个部分中又分了两套推荐逻辑，分别对应热门页和推荐页面。

总体逻辑：召回+排序+重排；

线下部分总结：是根据已有的数据，为不同用户对应的热门页和推荐页列表。同时生成冷启动页面存入redis中；

线上部分总结：对于新用户：拉取冷启动页面+重排（对重复曝光去重），对于老用户：拉取推荐列表+重排，最后展示到前端。

个人注意到，在目前的项目架构中，线上只有拉取线下计算的内容然后去重这一个步骤。但是我们仍把线上和线下分成两个大块，因为在后续加入具体算法模型后，线上与线下会分别选择不同的模型（比如线上为了运算速度会牺牲模型复杂度减少计算资源的消耗，线下则会运用复杂的模型计算）。

## 1.Offline

### 1.1 热门页构建

1. 流程：

   - 基于物料画像（离线物料系统生成的画像，`MongoDB`的`NewsRecSys`库的`FeatureProtail`集合数据）以及交互反馈数据（阅读，收藏，点赞）计算热度值

     热度值计算公式：

     ```python
     `news_hot_value = (news_likes_num * 0.6 + news_collections_num * 0.3 + news_read_num * 0.1) * 10 / (1 + time_hour_diff / 72)
     
     ```

     **注：** 上述公式源自魔方秀公式： 魔方秀热度 = (总赞数 * 0.7 + 总评论数 * 0.3) * 1000 / (公布时间距离当前时间的小时差+2) ^ 1.2

     最初的公式为Hacker News算法： Score = (P-1) / (T+2)^G 其中P表示文章得到的投票数，需要去掉文章发布者的那一票；T表示时间衰减（单位小时）；G为权重，默认值为1.8

   - 将所有新闻按照类别，将热门列表存入缓存中，提供给`Online`的热门页列表

2. 代码逻辑

   代码位于`recprocess/recall/hot_recall.py`。完整代码如下。

    ```python
    def get_hot_list(self, user_id):
            """热门页列表结果"""
            hot_list_key_prefix = "user_id_hot_list:"
            hot_list_user_key = hot_list_key_prefix + str(user_id)
    
            user_exposure_prefix = "user_exposure:"
            user_exposure_key = user_exposure_prefix + str(user_id)
    
            # 当数据库中没有这个用户的数据，就从热门列表中拷贝一份 
            if self.reclist_redis_db.exists(hot_list_user_key) == 0: # 存在返回1，不存在返回0
                print("copy a hot_list for {}".format(hot_list_user_key))
                # 给当前用户重新生成一个hot页推荐列表， 也就是把hot_list里面的列表复制一份给当前user， key换成user_id
                self.reclist_redis_db.zunionstore(hot_list_user_key, ["hot_list"])
    
            # 一页默认10个item, 但这里候选20条，因为有可能有的在推荐页曝光过
            article_num = 200
    
            # 返回的是一个news_id列表   zrevrange排序分值从大到小
            candiate_id_list = self.reclist_redis_db.zrevrange(hot_list_user_key, 0, article_num-1)
    
            if len(candiate_id_list) > 0:
                # 根据news_id获取新闻的具体内容，并返回一个列表，列表中的元素是按照顺序展示的新闻信息字典
                news_info_list = []
                selected_news = []   # 记录真正被选了的
                cou = 0
    
                # 曝光列表
                print("self.reclist_redis_db.exists(key)",self.exposure_redis_db.exists(user_exposure_key))
                if self.exposure_redis_db.exists(user_exposure_key) > 0:
                    exposure_list = self.exposure_redis_db.smembers(user_exposure_key)
                    news_expose_list = set(map(lambda x: x.split(':')[0], exposure_list))
                else:
                    news_expose_list = set()
    
                for i in range(len(candiate_id_list)):
                    candiate = candiate_id_list[i]
                    news_id = candiate.split('_')[1]
    
                    # 去重曝光过的，包括在推荐页以及hot页
                    if news_id in news_expose_list:
                        continue
    
                    # TODO 有些新闻可能获取不到静态的信息，这里应该有什么bug
                    # bug 原因是，json.loads() redis中的数据会报错，需要对redis中的数据进行处理
                    # 可以在物料处理的时候过滤一遍，json无法load的新闻
                    try:
                        news_info_dict = self.get_news_detail(news_id)
                    except Exception as e:
                        with open("/home/recsys/news_rec_server/logs/news_bad_cases.log", "a+") as f:
                            f.write(news_id + "\n")
                            print("there are not news detail info for {}".format(news_id))
                        continue
                    # 需要确认一下前端接收的json，key需要是单引号还是双引号
                    news_info_list.append(news_info_dict)
                    news_expose_list.add(news_id)
                    # 注意，原数的key中是包含了类别信息的
                    selected_news.append(candiate)
                    cou += 1
                    if cou == 10:
                        break
                
                if len(selected_news) > 0:
                    # 手动删除读取出来的缓存结果, 这个很关键, 返回被删除的元素数量，用来检测是否被真的被删除了
                    removed_num = self.reclist_redis_db.zrem(hot_list_user_key, *selected_news)
                    print("the numbers of be removed:", removed_num)
    
                # 曝光重新落表
                self._save_user_exposure(user_id,news_expose_list)
                return news_info_list 
            else:
                #TODO 临时这么做，这么做不太好
                self.reclist_redis_db.zunionstore(hot_list_user_key, ["hot_list"])
                print("copy a hot_list for {}".format(hot_list_user_key))
                # 如果是把所有内容都刷完了再重新拷贝的数据，还得记得把今天的曝光数据给清除了
                self.exposure_redis_db.delete(user_exposure_key)
                return  self.get_hot_list(user_id)
    ```

   - `update_hot_value()`方法：主要用于更新物料中所有新闻的热度值，通过遍历`MongoDB`的`NewsRecSys`库的`FeatureProtail`集合数据，选取最近3天发布的新闻，进行热度值的计算，并更新`hot_value`字段（初始值为1000）

   - `group_cate_for_news_list_to_redis()`方法：主要用于将物料库（`FeatureProtail`集合），通过遍历各类新闻，按照下面形式存入Redis[0]的`hot_list_news_cate`中：

     ```
     hot_list_news_cate:<新闻类别标识>: {<新闻类别名>_<新闻id>:<热度值>}
     
     ```

### 1.2 推荐页构建

1. 流程

   - 针对注册用户，建立冷启动的推荐页列表数据，通过用户年龄、性别指定用户群，自定义用户的新闻类别，根据新闻热度值，生成冷启动的推荐列表

   - 针对老用户，提供个性化推荐，目前该系统没有提供千人千面的推荐功能，需要使用机器学习流程来完成这一部分功能

2. 代码逻辑

   - `generate_cold_user_strategy_templete_to_redis_v2()`方法：按照用户类别、新闻类别提供各类新闻的热度值；通过自定义配置将用户分为4类，遍历各类别中的新闻类型，将物料库（`MongoDB`的`NewsRecSys`库的`FeatureProtail`集合数据）中对应的新闻热度值按照下面形式存入Redis[0]的`cold_start_group`中:

     ```python
     def generate_cold_user_strategy_templete_to_redis_v2(self):
             """冷启动用户模板，总共分成了四类人群
             每类人群都把每个类别的新闻单独存储
             """
             for k, item in self.user_group.items():
                 for cate in item:
                     cate_cnt = 0
                     cate_id = self.name2id_cate_dict[cate]
                     # k 表示人群分组
                     redis_key = "cold_start_group:{}:{}".format(str(k), cate_id)
                     for news_info in self.feature_protrail_collection.find({"cate": cate}):
                         cate_cnt += 1
                         self.reclist_redis.zadd(redis_key, {news_info['cate'] + '_' + news_info['news_id']: news_info['hot_value']}, nx=True)
                     print("类别 {} 的 新闻数量为 {}".format(cate, cate_cnt))
     
     ```

   - `user_news_info_to_redis()`方法：

     1. 按照用户、新闻分类提供用户冷启动的推荐模板；遍历所有用户，针对每个用户分类，从Redis[0]的`cold_start_group`中取出新闻热度值，按照以下形式存入Redis[0]的`cold_start_user`中：

        ```
        cold_start_user:<用户ID>:<新闻类别标识>: {<新闻类别名>_<新闻id>:<热度值>}
        
        ```

     2. 按照用户提供用户的新闻类别标识；针对每个用户，根据自定义的用户新闻类别（即每个用户分类具有哪些新闻类别），按照以下形式存入Redis[0]的`cold_start_user_cate_set`中：

        ```
        cold_start_user_cate_set:<用户ID>: {<新闻类别标识>}
        
        ```

   - 冷启动代码

   ```python
   def generate_cold_start_news_list_to_redis_for_register_user(self):
           """给已经注册的用户制作冷启动新闻列表
           """
           for user_info in self.register_user_sess.query(RegisterUser).all():
               if int(user_info.age) < 23 and user_info.gender == "female":
                   redis_key = "cold_start_group:{}".format(str(1))
                   self.copy_redis_sorted_set(user_info.userid, redis_key)
               elif int(user_info.age) >= 23 and user_info.gender == "female":
                   redis_key = "cold_start_group:{}".format(str(2))
                   self.copy_redis_sorted_set(user_info.userid, redis_key)
               elif int(user_info.age) < 23 and user_info.gender == "male":
                   redis_key = "cold_start_group:{}".format(str(3))
                   self.copy_redis_sorted_set(user_info.userid, redis_key)
               elif int(user_info.age) >= 23 and user_info.gender == "male":
                   redis_key = "cold_start_group:{}".format(str(4))
                   self.copy_redis_sorted_set(user_info.userid, redis_key)
               else:
                   pass 
           print("generate_cold_start_news_list_to_redis_for_register_user."
   ```

## 2.Online

### 2.1 热门页列表构建

1. 流程

   - 获取用户曝光列表，得到所有的新闻ID
   - 针对新用户，从之前的离线热门页列表中，为该用户生成一份热门页列表，针对老用户，直接获取该用户的热门页列表
   - 对上述热门列表，每次选取10条新闻，进行页面展示
   - 对已选取的10条新闻，更新曝光记录

2. 代码

   代码位于`recprocess/online.py`

   - `_get_user_expose_set()`方法：主要获取用户曝光列表，得到所有的新闻ID；该数据存储在Redis[3]中，数据格式如下：

     ```
     user_exposure:<用户ID>: {<新闻ID>:<曝光时间>} Copy to clipboardErrorCopied
     ```

   - `_judge_and_get_user_reverse_index()`方法：用于判断用户是否存在热门页列表，如果用户是新用户，根据Redis[0]的`hot_list_news_cate`中的数据，复制创建该用户的热门页列表，具体数据格式如下：

     ```
     user_id_hot_list:<用户ID>:<新闻分类标识>: {<新闻类别名>_<新闻id>:<热度值>}Copy to clipboardErrorCopied
     ```

   - `_get_polling_rec_list()`方法：通过轮询方式获取用户的展示列表，每次只取出10条新闻；通过while循环方式，从Redis[0]的`user_id_hot_list:<用户ID>:<新闻分类标识>`中选取新闻，之后删除已选取的新闻，并把选取的10条新闻内容放到用户新闻（`user_news_list`）数组中，新闻ID放到曝光列表（`exposure_news_list`）中

   - `_save_user_exposure()`方法：将曝光新闻数据存储到Redis[3]中；设置曝光时间，删除重复的曝光新闻，并按照下列格式存储到Redis[3]的`user_exposure`中：

     ```
     user_exposure:<用户ID>: {<新闻ID>:<曝光时间>} Copy to clipboard
     ```

### 2.2 推荐页列表构建

1. 流程

   - 获取用户曝光列表，得到所有的新闻ID
   - 针对新用户，从之前的离线推荐页列表中，为该用户生成一份推荐页列表，针对老用户，直接获取该用户的推荐页列表
   - 对上述热门列表，每次选取10条新闻，进行页面展示
   - 对已选取的10条新闻，更新曝光记录

2. 代码

   代码位于`recprocess/online.py`

   - `_get_user_expose_set()`方法：主要获取用户曝光列表，得到所有的新闻ID；该数据存储在Redis[3]中，数据格式如下：

     ```
     user_exposure:<用户ID>: {<新闻ID>:<曝光时间>} Copy to clipboardErrorCopied
     ```

   - `_judge_and_get_user_reverse_index()`方法：用于判断用户是否存在冷启动列表，如果用户是新用户，获取分组ID，根据用户DI和分组ID，从Redis[0]的`cold_start_group`中的数据，复制创建该用户的推荐列表，具体数据格式如下：

     ```
     cold_start_user:<用户ID>:<新闻分类标识>: {<新闻类别名>_<新闻id>:<热度值>}Copy to clipboardErrorCopied
     ```

     将用户当前的新闻类别标识，存放到`cold_start_user_cate_set`中，具体格式如下：

     ```
     cold_start_user_cate_set:<用户ID>: {<新闻ID>}Copy to clipboardErrorCopied
     ```

   - `_get_polling_rec_list()`方法：通过轮询方式获取用户的展示列表，每次只取出10条新闻；通过while循环方式，从Redis[0]的`cold_start_user:<用户ID>:<新闻分类标识>`中选取新闻，之后删除已选取的新闻，并把选取的10条新闻内容放到用户新闻（`user_news_list`）数组中，新闻ID放到曝光列表（`exposure_news_list`）中

   - `_save_user_exposure()`方法：将曝光新闻数据存储到Redis[3]中；设置曝光时间，删除重复的曝光新闻，并按照下列格式存储到Redis[3]的`user_exposure`中：

     ```
     user_exposure:<用户ID>: {<新闻ID>:<曝光时间>} 
     ```



