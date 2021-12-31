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

   

2. 代码逻辑
