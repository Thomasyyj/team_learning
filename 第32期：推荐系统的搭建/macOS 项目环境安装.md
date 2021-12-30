# fun-rec macOS 环境安装

## 1. 相关软件安装

- MySQL：

  - 下载链接：[下载 MySQL Community Server](https://dev.mysql.com/downloads/mysql/)
  - 配置教程：[Mac 系统下安装配置 MySQL 的方法 - 知乎](https://zhuanlan.zhihu.com/p/27960044)
  - 配置好后启动 MySQL：`mysql -u root -p`

- MongoDB：

  - 下载链接：[Try MongoDB Products | MongoDB](https://www.mongodb.com/try#community)

  - 配置教程：[在 macOS 上安装 MongoDB 社区版](https://docs.mongoing.com/install-mongodb/install-mongodb-community-edition/install-on-macos)

  - 推荐使用 homebrew 安装：
    - ``````shell
      brew tap mongodb/brew
      brew install mongodb-community@4.4
      ``````

- conda（控制 Python 版本）：

  - 配置教程：[mac 下 anaconda 的安装及简单使用](https://blog.csdn.net/lq_547762983/article/details/81003528)

  - 创建新的 Python 环境

    ```shell
    conda create --prefix venv python=3.8
    ```

## 2. 后端环境安装和启动

> 注意：下文中 `XXX` 为存放 `fun-rec` 项目的文件夹目录

- 修改后端提供给前端的端口号。将 `XXX/fun-rec/codes/news_recsys/news_rec_server/server.py` 中第 237 行的 IP 和端口号。将 `app.run(debug=True, host='0.0.0.0', port=3000, threaded=True)` 修改为 `app.run(debug=True, host='127.0.0.1', port=5000, threaded=True)`。
- 修改路径。将 `XXX/fun-rec/codes/news_recsys/news_rec_server/conf/proj_path.py` 中第 4 行的 `proj_path = home_path + "/fun-rec/codes/news_recsys/news_rec_server/"` 改为 `proj_path = os.path.join(sys.path[1], '')`

然后开始安装依赖

```shell
cd XXX/fun-rec/codes/news_recsys/news_rec_server
# 安装依赖包
pip install -r requirements.txt
```

启动雪花：

```shell
cd XXX/fun-rec/codes/news_recsys/news_rec_server
# 启动雪花
snowflake_start_server --address=127.0.0.1 --port=8910 --dc=1 --worker=1
```

启动项目（启动之前需要启动 SQL 并创建表）：

```shell
cd XXX/fun-rec/codes/news_recsys/news_rec_server
python server.py
```

## 前端环境安装

- 先修改 `node-sass` 版本。将 `XXX/fun-rec/codes/news_recsys/news_rec_web/Vue-newsinfo/package.json` 文件中的 `"node-sass": "^4.12.0",` 修改为 `"node-sass": "^6.0.1",`。

- 修改前端端口号。将 `XXX/fun-rec/codes/news_recsys/news_rec_web/Vue-newsinfo/package.json` 文件中的 `"dev": "webpack-dev-server --open --port 8686 --contentBase src --hot --host 0.0.0.0",` 修改为 `"dev": "webpack-dev-server --open --port 8686 --contentBase src --hot --host 127.0.0.1",`。
- 修改后端端口号。将 `XXX/fun-rec/codes/news_recsys/news_rec_web/Vue-newsinfo/src/main.js` 文件中第 22 行的 IP 和端口号。将 `axios.defaults.baseURL = "http://47.108.56.188:3000";` 修改为 `axios.defaults.baseURL = "http://127.0.0.1:5000";`。
- 然后再开始安装依赖。

```shell
## XXX 是存放 fun-rec 的文件夹目录
cd XXX/fun-rec/codes/news_recsys/news_rec_web/Vue-newsinfo
# 安装依赖包
npm install
```

