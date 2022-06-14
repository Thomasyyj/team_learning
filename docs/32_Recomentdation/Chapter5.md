# Task5:推荐系统流程

![](material/ch1-overview.png)

本次任务主要介绍推荐系统的流程构建，主要分为线下部分和线上部分，每个部分中又分了两套推荐逻辑，分别对应热门页和推荐页面。

总体逻辑：召回+排序+重排；

线下部分总结：是根据已有的数据，为不同用户对应的热门页和推荐页列表。同时生成冷启动页面存入redis中；

线上部分总结：对于新用户：拉取冷启动页面+重排（对重复曝光去重），对于老用户：拉取推荐列表+重排，最后展示到前端。

在后续加入具体算法模型后，线上与线下会分别加入不同的模型（比如线上为了运算速度会牺牲模型复杂度减少计算资源的消耗，线下则会运用复杂的模型计算）。

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
    def update_hot_value(self):
            """更新物料库中所有新闻的热度值
            """
            # 遍历物料池里面的所有文章
            for item in self.feature_protrail_collection.find():
                news_id = item['news_id']
                news_cate = item['cate']
                news_ctime = item['ctime']
                news_likes_num = item['likes']
                news_collections_num = item['collections']
                news_read_num = item['read_num']
                news_hot_value = item['hot_value']
    
                # 时间转换与计算时间差   前提要保证当前时间大于新闻创建时间，目前没有捕捉异常
                news_ctime_standard = datetime.strptime(news_ctime, "%Y-%m-%d %H:%M")
                cur_time_standard = datetime.now()
                time_day_diff = (cur_time_standard - news_ctime_standard).days
                time_hour_diff = (cur_time_standard - news_ctime_standard).seconds / 3600
    
                # 只要最近3天的内容
                if time_day_diff > 3:
                    continue
    
                # 72 表示的是3天，
                news_hot_value = (news_likes_num * 0.6 + news_collections_num * 0.3 + news_read_num * 0.1) * 10 / (1 + time_hour_diff / 72) 
    
                # 更新物料池的文章hot_value
                item['hot_value'] = news_hot_value
                self.feature_protrail_collection.update({'news_id':news_id}, item)
    
        def get_hot_rec_list(self):
            """获取物料的点赞，收藏和创建时间等信息，计算热度并生成热度推荐列表存入redis
            """
            # 遍历物料池里面的所有文章
            for item in self.feature_protrail_collection.find():
                news_id = item['news_id']
                news_cate = item['cate']
                news_ctime = item['ctime']
                news_likes_num = item['likes']
                news_collections_num = item['collections']
                news_read_num = item['read_num']
                news_hot_value = item['hot_value']
    
                #print(news_id, news_cate, news_ctime, news_likes_num, news_collections_num, news_read_num, news_hot_value)
    
                # 时间转换与计算时间差   前提要保证当前时间大于新闻创建时间，目前没有捕捉异常
                news_ctime_standard = datetime.strptime(news_ctime, "%Y-%m-%d %H:%M")
                cur_time_standard = datetime.now()
                time_day_diff = (cur_time_standard - news_ctime_standard).days
                time_hour_diff = (cur_time_standard - news_ctime_standard).seconds / 3600
    
                # 只要最近3天的内容
                if time_day_diff > 3:
                    continue
                
                # 计算热度分，这里使用魔方秀热度公式， 可以进行调整, read_num 上一次的 hot_value  上一次的hot_value用加？  因为like_num这些也都是累加上来的， 所以这里计算的并不是增值，而是实时热度吧
                # news_hot_value = (news_likes_num * 6 + news_collections_num * 3 + news_read_num * 1) * 10 / (time_hour_diff+1)**1.2
                # 72 表示的是3天，
                news_hot_value = (news_likes_num * 0.6 + news_collections_num * 0.3 + news_read_num * 0.1) * 10 / (1 + time_hour_diff / 72) 
    
                #print(news_likes_num, news_collections_num, time_hour_diff)
    
                # 更新物料池的文章hot_value
                item['hot_value'] = news_hot_value
                self.feature_protrail_collection.update({'news_id':news_id}, item)
    
                #print("news_hot_value: ", news_hot_value)
    
                # 保存到redis中
                self.reclist_redis.zadd('hot_list', {'{}_{}'.format(news_cate, news_id): news_hot_value}, nx=True)
    
    
        def group_cate_for_news_list_to_redis(self, ):
            """将每个用户的推荐列表按照类别分开，方便后续打散策略的实现
            对于热门页来说，只需要提前将所有的类别新闻都分组聚合就行，后面单独取就可以
            注意：运行当前脚本的时候需要需要先更新新闻的热度值
            """
            # 1. 按照类别先将所有的新闻都分开存储
            for cate_id, cate_name in self.cate_dict.items():
                redis_key = "news_cate:{}".format(cate_id)
                for item in self.feature_protrail_collection.find({"cate": cate_name}):
                    self.reclist_redis.zadd(redis_key, {'{}_{}'.format(item['cate'], item['news_id']): item['hot_value']})
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

   这段代码实现的逻辑和实际流程有些不一样。我们先自定义了四组用户类型，根据四组用户类型先储存好所有的新闻（相当于做好了冷启动列表），之后再根据用户的画像把用户分成四类人，然后把对应的新闻一起储存进去。

   思考：在后续阶段，我认为自定义固定数量的用户画像是不合理的，这个应该需要通过机器学习模型来学。

   - 初始定义用户类型

     ```python
     def _set_user_group(self):
             """将用户进行分组
             1. age < 23 && gender == female  
             2. age >= 23 && gender == female 
             3. age < 23 && gender == male 
             4. age >= 23 && gender == male  
             """
             self.user_group = {
                 "1": ["国内","娱乐","体育","科技"],
                 "2": ["国内","社会","美股","财经","股市"],
                 "3": ["国内","股市","体育","科技"],
                 "4": ["国际", "国内","军事","社会","美股","财经","股市"]
             }
             self.group_to_cate_id_dict = defaultdict(list)
             for k, cate_list in self.user_group.items():
                 for cate in cate_list:
                     self.group_to_cate_id_dict[k].append(self.name2id_cate_dict[cate])
     ```

   - `generate_cold_user_strategy_templete_to_redis_v2()`方法：

     这段代码的目的是先生成冷启动的列表。按照4类自定义的用户类型，遍历各类别中的新闻类型，将物料库（`MongoDB`的`NewsRecSys`库的`FeatureProtail`集合数据）中对应的新闻热度值存入Redis[0]的`cold_start_group`中:

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

     ```python
     def user_news_info_to_redis(self):
             """将每个用户涉及到的不同的新闻列表添加到redis中去
             """
             for user_info in self.register_user_sess.query(RegisterUser).all():
                 if int(user_info.age) < 23 and user_info.gender == "female":
                     self._copy_cold_start_list_to_redis(user_info.userid, group_id="1")
                 elif int(user_info.age) >= 23 and user_info.gender == "female":
                     self._copy_cold_start_list_to_redis(user_info.userid, group_id="2")
                 elif int(user_info.age) < 23 and user_info.gender == "male":
                     self._copy_cold_start_list_to_redis(user_info.userid, group_id="3")
                 elif int(user_info.age) >= 23 and user_info.gender == "male":
                     self._copy_cold_start_list_to_redis(user_info.userid, group_id="4")
                 else:
                     pass 
     def _copy_cold_start_list_to_redis(self, user_id, group_id):
             """将确定分组后的用户的物料添加到redis中，并记录当前用户的所有新闻类别id
             """
             # 遍历当前分组的新闻类别
             for cate_id in self.group_to_cate_id_dict[group_id]:
                 group_redis_key = "cold_start_group:{}:{}".format(group_id, cate_id)
                 user_redis_key = "cold_start_user:{}:{}".format(user_id, cate_id)
                 self.reclist_redis.zunionstore(user_redis_key, [group_redis_key])
             # 将用户的类别集合添加到redis中
             cate_id_set_redis_key = "cold_start_user_cate_set:{}".format(user_id)
             self.reclist_redis.sadd(cate_id_set_redis_key, *self.group_to_cate_id_dict[group_id])
     ```

     1. 按照用户、新闻分类提供用户冷启动的推荐模板；遍历所有用户，针对每个用户分类，从Redis[0]的`cold_start_group`中取出新闻热度值，按照以下形式存入Redis[0]的`cold_start_user`中：

        ```
        cold_start_user:<用户ID>:<新闻类别标识>: {<新闻类别名>_<新闻id>:<热度值>}
        
        ```

     2. 按照用户提供用户的新闻类别标识；针对每个用户，将用户的类别集合按照以下形式存入Redis[0]的`cold_start_user_cate_set`中：

        ```
        cold_start_user_cate_set:<用户ID>: {<新闻类别标识>}
        
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



