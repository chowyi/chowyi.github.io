title: Django使用MySQL后端日期不能按月过滤的问题及解决方案
date: 2017-9-19 21:04:28
tags:
- Django
- MySQL
categories:
- Django
---
近期在工作中有一个需求，按创建年份统计资产的数量，一行代码就能实现，分分钟的事情。后来需求变动，要求按月份统计。我一想，这也没问题啊，对 Django 来说也就是加一个参数的事。代码改完，一调接口，咦，统计结果为什么全是 0 啊？
后面一步一步的定位问题，还挺有意思的，记录下来。
<!--more-->

## 问题描述
原项目代码较多，而且涉及工作，下面我用一个简单的例子来重现一下我遇到的问题：
开发环境：Python 2.7.11 + Django 1.11.3 + MySQL 5.5.43

新建一个 Django 工程`hello`，新建一个 Django app `frame`:
```
django-admin startproject hello
cd hello
python manage.py startapp frame
```

在 frame/models.py 中添加一个模型：
```
# -*- coding: utf-8 -*-
from __future__ import unicode_literals

from django.db import models

class Book(models.Model):
    name = models.CharField('Name', max_length=50)
    public_date = models.DateTimeField('Public_Date', auto_now=True)
```

修改配置文件 hello/settings.py，在`INSTALLED_APPS`中加入`hello`，修改`DATABASES`的配置连接到本地 MySQL 数据库，时区设置为`TIME_ZONE = 'Asia/Shanghai'`。

执行 `makemigrations` 和 `migrate` 创建数据表，然后 `python manage.py shell` 进入交互模式：
```
>>> from frame.models import Book
>>> python = Book.objects.create(name='Python')
>>> java = Book.objects.create(name='Java')
>>> python.public_date
datetime.datetime(2017, 9, 19, 6, 6, 6, 534471, tzinfo=<UTC>)
>>>
>>> Book.objects.filter(public_date__year=2017)
<QuerySet [<Book: Book object>, <Book: Book object>]>
>>>
>>> Book.objects.filter(public_date__month=9)
<QuerySet []>
```

上面的代码先创建了两个`Book`的实例，由于`public_date`字段设置了`auto_now=True`，所以默认为当前时间。然后按年份查询了一下，得到两条结果，没有问题。再按月份查询一下，结果为空，问题重现了。


## 搜索网络
能按年份查，不能按月份查，Django 这么成熟的企业级框架应该不会犯这种错误吧。网上搜索一下：
1. 发现有人遇到了同样的问题，不过只有提问，没有回答。（顺便提一下，这个问题出现在很多小网站上，应该都是用爬虫抓过去凑文章刷访问量的。在此对这种行为表示谴责。）
2. 还有人说可以用`.filter(startswith="2017-09")`这样的方式来查询。试了一下，可以实现效果，但是有几个问题：
    - 查询条件必须和数据库中保存日期的格式相同。
    - 如果要查询所有9月份而不是2017年的9月份就没有办法了，难道要用正则匹配？
    - 这实在是不够优雅，我相信应该会有真正的解决办法。

网上没有得到有用信息，自己来想办法吧。

## 问题定位
打印 Django 中 QuerySet 对象的 query 属性可以得到 Django 操作数据库的 sql 语句。
```
>>> print Book.objects.filter(public_date__year=2017).query
SELECT `frame_book`.`id`, `frame_book`.`name`, `frame_book`.`public_date` FROM `frame_book` WHERE `frame_book`.`public_date` BETWEEN 2016-12-31 16:00:00 AND 2017-12-31 15:59:59
>>>
>>> print Book.objects.filter(public_date__month=9).query
SELECT `frame_book`.`id`, `frame_book`.`name`, `frame_book`.`public_date` FROM `frame_book` WHERE EXTRACT(MONTH FROM CONVERT_TZ(`frame_book`.`public_date`, 'UTC', Asia/Shanghai)) = 9
```

看下 where 子句就发现了按年查询和按月份查询的区别。
按年份查询2017年的记录时，查询条件是：
```
WHERE `frame_book`.`public_date` BETWEEN 2016-12-31 16:00:00 AND 2017-12-31 15:59:59
```

直接判断日期是不是在2017年1月1日0点到2017年12月31日23点59分59秒之间。（因为 Django 项目设置了时区为`Asia/Shanghai`，而数据库中保存的是UTC时间，所以要提前8小时）。

而按月份查询时，要先转换时区，然后提取月份：
```
WHERE EXTRACT(MONTH FROM CONVERT_TZ(`frame_book`.`public_date`, 'UTC', Asia/Shanghai)) = 9
```

看起来没什么问题，这个时区的转换没毛病吧？到 MySQL 里试试去。
```
mysql> select convert_tz('2017-09-19 17:30:00', 'UTC', Asia/Shanghai);
ERROR 1054 (42S22): Unknown column 'Asia' in 'field list'
```

这 Asia/Shanghai 没加引号啊？（Django的问题的吗？）加上引号试试。
```
mysql> select convert_tz('2017-09-19 17:30:00', 'UTC', 'Asia/Shanghai');
+-----------------------------------------------------------+
| convert_tz('2017-09-19 17:30:00', 'UTC', 'Asia/Shanghai') |
+-----------------------------------------------------------+
| NULL                                                      |
+-----------------------------------------------------------+
1 row in set (0.00 sec)
```

看来就是这里的问题哦！时区转换失败了！

## 查找资料
convert_tz 是 MySQL 中的一个函数。查查[官方文档](https://dev.mysql.com/doc/refman/5.5/en/date-and-time-functions.html#function_convert-tz):
> CONVERT_TZ(dt,from_tz,to_tz)

> CONVERT_TZ() converts a datetime value dt from the time zone given by from_tz to the time zone given by to_tz and returns the resulting value. Time zones are specified as described in Section 10.6, “MySQL Server Time Zone Support”. This function returns NULL if the arguments are invalid.

> If the value falls out of the supported range of the TIMESTAMP type when converted from from_tz to UTC, no conversion occurs. The TIMESTAMP range is described in Section 11.1.2, “Date and Time Type Overview”. 

>```
>mysql> SELECT CONVERT_TZ('2004-01-01 12:00:00','GMT','MET');
>        -> '2004-01-01 13:00:00'
>mysql> SELECT CONVERT_TZ('2004-01-01 12:00:00','+00:00','+10:00');
>        -> '2004-01-01 22:00:00'
>```
> 
>>Note

>>To use named time zones such as 'MET' or 'Europe/Moscow', the time zone tables must be properly set up. See Section 10.6, “MySQL Server Time Zone Support”, for instructions.


这文档写的可真好，很清楚了，简单来说就是：
1. convert_tz 用于转换时区，有三个参数，分别是时间，当前时区，目标时区。
2. 时区有两种写法，一是`'+00:00'`，`'+10:00'`这样的直接写时差，二是用时区名，如`'GMT'`，`'MET'`。
3. 如果时区是无效的，结果返回`NULL`。
4. 如果不支持时区名可以看[Section 10.6, “MySQL Server Time Zone Support”](https://dev.mysql.com/doc/refman/5.5/en/time-zone-support.html)。

这答案已经呼之欲出了！不用时区名，直接用时差的写法试试看：
```
mysql> select convert_tz('2017-09-19 17:30:00', '+00:00', '+08:00');
+-------------------------------------------------------+
| convert_tz('2017-09-19 17:30:00', '+00:00', '+08:00') |
+-------------------------------------------------------+
| 2017-09-20 01:30:00                                   |
+-------------------------------------------------------+
1 row in set (0.00 sec)
```

问题就出在查询不支持时区名上！


## 解决方案
问题找到了，解决就很容易了。上面也提到，MySQL 的文档[Section 10.6, “MySQL Server Time Zone Support”](https://dev.mysql.com/doc/refman/5.5/en/time-zone-support.html)中有解决方案：
> The mysql_tzinfo_to_sql program is used to load the time zone tables. On the command line, pass the zoneinfo directory path name to mysql_tzinfo_to_sql and send the output into the mysql program. For example:
```
shell> mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root mysql
```

在命令行中执行上面的命令，除了两个警告，看起来没什么问题。
```
(python-django)vagrant@precise64:~/test/hello$ mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -uroot mysql -p
Enter password: 
Warning: Unable to load '/usr/share/zoneinfo/iso3166.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone.tab' as time zone. Skipping it.
```

在 MySQL 中验证一下：
```
mysql> select convert_tz('2017-09-19 17:30:00', 'UTC', 'Asia/Shanghai');
+-----------------------------------------------------------+
| convert_tz('2017-09-19 17:30:00', 'UTC', 'Asia/Shanghai') |
+-----------------------------------------------------------+
| 2017-09-20 01:30:00                                       |
+-----------------------------------------------------------+
1 row in set (0.00 sec)
```

在 Django 中验证一下：
```
>>> from frame.models import Book
>>> Book.objects.filter(public_date__month=9)
<QuerySet [<Book: Book object>, <Book: Book object>]>
>>> 
```

完全正常，问题解决了！
修改配置文件，换用自带的 sqlite 数据库，同样的测试，没有发现此问题。

对了，上面提到少引号的问题：
```
>>> print Book.objects.filter(public_date__month=9).query
SELECT `frame_book`.`id`, `frame_book`.`name`, `frame_book`.`public_date` FROM `frame_book` WHERE EXTRACT(MONTH FROM CONVERT_TZ(`frame_book`.`public_date`, 'UTC', Asia/Shanghai)) = 9
```

上面那句中`Asia/Shanghai`没有加引号，直接拷贝 sql 到 MySQL 执行是会报错的，而在 Django 中没有问题。我猜想可能是 Django 中 Query 对象的`__str__`方法有问题。看了一下源码，果然是这里有玄机：
```
# django/db/models/sql/query.py 
def __str__(self):
    """
    Returns the query as a string of SQL with the parameter values
    substituted in (use sql_with_params() to see the unsubstituted string).

    Parameter values won't necessarily be quoted correctly, since that is
    done by the database interface at execution time.
    """
    sql, params = self.sql_with_params()
    return sql % params
```
这里写了，参数的引用可能不准确，在执行数据库接口时才会被完成。


## 我的总结
这次定位问题，解决问题的思路还是挺清晰的，整个过程也很有趣。
以后要加强下 SQL 方面的知识。
