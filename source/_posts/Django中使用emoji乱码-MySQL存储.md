title: Django中使用emoji乱码 MySQL存储
date: 2016-02-04 19:40:36
tags:
- Django
- emoji
- Python
categories:
- Django
---
严格来说，emoji的乱码跟django并没有什么关系。这个最后再说吧。
我在尝试使用django来创建一个自己个人博客，在评论功能中，我使用emoji-picker插件给web页面上的输入框加入了输入emoji表情的功能。而然在MySQL中储存emoji时发生了乱码，在网络上查资料后解决了问题，记录一下。
<!--more-->

## 为什么会出现乱码？
utf-8是我们常用的字符编码，特别是在开发工作中，我们常常把程序源代码，html页面，数据库字符集等设置为utf-8。utf-8是一种变长编码,英文字母和数字编码为一个字节，汉字大部分编码为3个字节, 少量汉字编码为的2个字节。emoji表情被编码为4个字节。而Mysql的utf8编码最多3个字节，因此保存emoji表情是就会出现乱码问题。

## 如何解决？
**将 MySQL 的编码从 utf8 转换成 utf8mb4 。**
*注意：mysql-5.5.3之前的版本都不支持4个byte的utf8，之后的版本才有utf8mb4支持4个byte的utf8字符*
### 如何修改MySQL的编码：
我从网上找到的解决办法是这样的：
>[原文 CSDN博客](http://blog.csdn.net/likendsl/article/details/7530979)
>1.修改my.cnf
>
>[mysqld]
>character-set-server=utf8mb4
>[mysql]
>default-character-set=utf8mb4
>修改后重启Mysql
>
>2.以root身份登录Mysql，修改环境变量，将character_set_client,character_set_connection,character_set_database,character_set_results,character_set_server 都修改成utf8mb4
>
>3.将已经建好的表也转换成utf8mb4
>命令：alter table TABLE_NAME convert to character set utf8mb4 collate utf8mb4_bin; （将TABLE_NAME替换成你的表名）

实际过程中我做完第一步就已经ok了！
P.S 如何查看MySQL环境变量：
`show variables`

至此，MySQL已经可以正确的存取emoji表情了。

### 在Django中的设置：
除了以上对MySQL的设置，如果使用django的ORM来操作数据库，还需要在 project/settings.py 中为数据库连接设置添加一项OPTIONS：
```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'blog',
        'USER': 'root',
        'PASSWORD': 'root',
        'HOST': '127.0.0.1',
        'PORT': '3306',
        'OPTIONS': {'charset':'utf8mb4'},
    }
}
```
现在已经可以正常的通过django的在MySQL中存取emoji表情了！