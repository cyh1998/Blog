#### 环境：
Ubuntu 18.04  
Python 3.6.9
#### scrapy的安装
准备工作，需要pip、python-dev
```
sudo apt-get install python3-pip //指定安装python3的pip
sudo apt-get install python3-dev //指定安装python3最新的
```
因为本人虚拟机上同时拥有python2和python3的版本，这里在安装时需要指定是python3的版本，如果不指定，默认安装python2的版本。  
安装scrapy
```
sudo pip3 install Scrapy
```
网上的教程中说需要安装一些依赖，这里提供下命令，如果环境依赖缺少，自行安装下
```
sudo apt-get install build-essential
sudo apt-get install libxml2-dev
sudo apt-get install libxslt1-dev
sudo apt-get install python-setuptools
```
最后使用命令 `scrapy --version` ，若显示scrapy版本，则安装成功
#### 创建项目
在你需要创建项目的目录执行
```
scrapy startproject xiaomiapp
```
`scrapy startproject 项目名` 这里演示的爬虫网站为小米应用商城，项目名设为xiaomiapp  
新建的项目目录为：
```
xiaomiapp/
├── scrapy.cfg
└── xiaomiapp
    ├── __init__.py
    ├── items.py
    ├── middlewares.py
    ├── pipelines.py
    ├── __pycache__
    ├── settings.py
    └── spiders
        ├── __init__.py
        └── __pycache__
```
`scrapy.cfg`：项目配置文件  
`items.py`：项目的数据容器文件，用来定义获取的数据  
`middlewares.py`：项目的中间件文件  
`pipelines.py`：项目管道文件，用于对items中的数据进行进一步的处理  
`settings.py`：设置文件，包含了爬虫项目的设置信息  
`spiders` ：存放自定义爬虫文件的目录  

进入`spiders`目录，执行  
```
scrapy genspider xiaomiapp app.mi.com/hotCatApp/10
```
`scrapy genspider 文件名 爬取网站域名`，创建自定义爬虫文件，修改其内容：
```
# -*- coding: utf-8 -*-
import scrapy
from xiaomiapp.items import XiaomiappItem

class XiaomiappPySpider(scrapy.Spider):
    name = 'xiaomiapp.py'
    allowed_domains = ['app.mi.com/hotCatApp/10']
    start_urls = ['http://app.mi.com/hotCatApp/10/']

    def parse(self, response):
        lies = response.css('.applist li')
        for li in lies:
            item = XiaomiappItem()
            item['appname'] = li.css('h5 a::text').extract()
            item['appdesc'] = li.css('.app-desc a::text').extract()
            yield item
        pass
```
说明：  
`name`：文件名  
`allowed_domains`：允许的域名  
`start_urls`：请求的地址  
结合网站的实际内容，使用`response.css()`来获取网站内容，除了CSS选择器之外，Scrapy还支持使用re方法以正则表达式提取内容，以及xpath方法以XPATH语法提取内容。`.extract()`可直接获取包含在标签内的文本  

修改`items.py`内容：
```
# -*- coding: utf-8 -*-

# Define here the models for your scraped items
#
# See documentation in:
# https://docs.scrapy.org/en/latest/topics/items.html

import scrapy

class XiaomiappItem(scrapy.Item):
    # define the fields for your item here like:
    # name = scrapy.Field()
    appname = scrapy.Field()
    appdesc = scrapy.Field()
    pass
```
item是保存爬取数据的容器，将可将获取的数据放到item中，类似于字典。
#### 运行
在项目目录执行
```
scrapy crawl xiaomiapp
```
结果如下：

![爬取网站](https://upload-images.jianshu.io/upload_images/22192996-d0f4c637e077aad3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![爬取结果](https://upload-images.jianshu.io/upload_images/22192996-39625ed2e718a20d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

#### 数据保存
将数据保存为csv、xml、json格式，可以直接使用命令：
```
scrapy crawl xiaomiapp -o xiaomiapp.csv
scrapy crawl xiaomiapp -o xiaomiapp.xml
scrapy crawl xiaomiapp -o xiaomiapp.json
```
运行后目录中会出现csv、xml、json格式文件，如果中文输出乱码，在`settings.py`中配置
```
FEED_EXPORT_ENCODING = 'utf-8'
```
再次运行，结果：

![json数据](https://upload-images.jianshu.io/upload_images/22192996-fbc2a4425d805598.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
