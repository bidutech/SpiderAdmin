# SpiderAdmin

![PyPI](https://img.shields.io/pypi/v/spideradmin.svg)

- github: https://github.com/mouday/SpiderAdmin
- pypi: https://pypi.org/project/spideradmin/


## 功能介绍
1. 对Scrapyd 接口进行可视化封装，对Scrapy爬虫项目进行删除 和 查看

2. 并没有实现修改，添加功能, 部署推荐使用
```bash
$ scrapyd-deploy -a
```
3. 对爬虫设置定时任务，支持apscheduler 的3中方式和随机延时，共计4中方式
- 单次运行 date
- 周期运行 corn
- 间隔运行 interval
- 随机运行 random

4. 基于Flask-BasicAuth 做了简单的权限校验

## 启动运行

```
$ pip3 install spideradmin

$ spideradmin init  # 初始化，可选配置，也可以使用默认配置

$ spideradmin       # 启动服务

```
访问：
http://127.0.0.1:5000/


## 页面截图
![](https://github.com/mouday/SpiderAdmin/raw/master/image/main.png)

![](https://github.com/mouday/SpiderAdmin/raw/master/image/status.png)

![](https://github.com/mouday/SpiderAdmin/raw/master/image/task.png)

![](https://github.com/mouday/SpiderAdmin/raw/master/image/time.png)

## TODO
1. ~~增加登录页面做权限校验~~
2. ~~增加定时设置的多样性~~
3. ~~增加定时随机运行~~

## 部署Scrapyd注意版本问题
- Scrapyd==1.2.0
- Scrapy==1.6.0
- Twisted==18.9.0

## 启用执行结果扩展
安装依赖
```
pip install PureMySQL
```

新建数据表
```sql
CREATE TABLE `log_spider` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `spider_name` varchar(50) DEFAULT NULL,
  `item_count` int(11) DEFAULT NULL,
  `duration` int(11) DEFAULT NULL,
  `log_error` int(11) DEFAULT NULL,
  `create_time` datetime DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='爬虫执行结果统计'
```

### 1、在scrapy项目中添加数据收集扩展文件

item_count_extension.py

```python
# -*- coding: utf-8 -*-

import logging

from scrapy import signals
from puremysql import PureMysql
from datetime import datetime


class SpiderItemCountExtension(object):
    
    # 设置为数据库链接url eg: mysql://root:123456@127.0.0.1:3306/mydata 
    ITEM_LOG_DATABASE_URL = None
    
    # 设置数据表
    ITEM_LOG_TABLE = "log_spider"

    @classmethod
    def from_crawler(cls, crawler):
        ext = cls()
        crawler.signals.connect(ext.spider_closed, signal=signals.spider_closed)
        return ext

    def spider_closed(self, spider, reason):
        stats = spider.crawler.stats.get_stats()
        scraped_count = stats.get("item_scraped_count", 0)
        dropped_count = stats.get("item_dropped_count", 0)

        log_error = stats.get("log_count/ERROR", 0)
        start_time = stats.get("start_time")
        finish_time = stats.get("finish_time")
        duration = (finish_time - start_time).seconds

        count = scraped_count + dropped_count

        logging.debug("*" * 50)
        logging.debug("* {}".format(spider.name))
        logging.debug("* item count: {}".format(count))
        logging.debug("*" * 50)

        item = {
            "spider_name": spider.name,
            "item_count": count,
            "duration": duration,
            "log_error": log_error,
            "create_time": datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        }

        mysql = PureMysql(db_url=self.ITEM_LOG_DATABASE_URL)
        table = mysql.table(self.ITEM_LOG_TABLE)
        table.insert(item)
        mysql.close()

```

2、scrapy项目启用扩展
settings.py

```python
EXTENSIONS = {
    # 'scrapy.extensions.telnet.TelnetConsole': None,
    "item_count_extension.SpiderItemCountExtension": 100
}
```

3、配置相同的db_url
default_config.py
```python
ITEM_LOG_DATABASE_URL = "mysql://root:123456@127.0.0.1:3306/mydata"
ITEM_LOG_TABLE = "log_spider"
```

## 更新日志

| 版本 | 日期 | 描述|
|- | - | -|
|0.0.17 | 2019-07-02 | 优化文件，优化随机调度，增加调度历史统计和可视化 |
|0.0.20 | 2019-10-08 | 增加执行结果统计 |
