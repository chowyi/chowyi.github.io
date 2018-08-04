title: 在Django模板中显示emoji表情
date: 2016-02-04 19:49:50
tags:
- Django
- emoji
- Python
categories:
- Django
---
现在我已经成功的在博客的评论功能中加入了输入emoji表情的功能，它们以Unicode编码的形式保存在MySQL数据库中，并使用MySQL的utf8mb4字符集解决了乱码的问题。现在要如何把它们从Unicode编码重新转换为emoji表情显示在页面上呢？<!--more-->通过搜索，对比了几个jQuery插件之后，我决定使用Github上的一个开源项目[gaqzi/django-emoji](https://github.com/gaqzi/django-emoji)。

## 如何使用[gaqzi/django-emoji](https://github.com/gaqzi/django-emoji)

### django-emoji简介
django-emoji是一款简单易用的django app， 到目前为止，最新的版本号是 django-emoji 2.1.0

#### 安装
使用如下命令安装：`pip install django-emoji`

#### 如何使用
在你的django项目setting.py的`INSTALLED_APPS`·中添加`emoji`，像这样：
```
INSTALLED_APPS = (
    ...
    'emoji',
)
```
在django模板中引入`emoji_tags`，为你可能包含emoji表情的文本添加过滤器，像这样：
```
{% load emoji_tags %}
{{ blog_post.body|emoji_replace }}
{{ blog_post.body|emoji_replace_unicode }}
{{ blog_post.body|emoji_replace_html_entities }}
```
上面展示了三种过滤器：
emoji_replace 用来把形如` : dog : `(没有空格)这样的emoji标记替换为emoji表情图标
emoji_replace_unicode 用来把标识emoji的Unicode编码替换为emoji表情图标
emoji_replace_html_entities 用来把html编码的Unicode emoji表情替换为emoji表情图标

这样就可以在django模板中愉快的使用emoji表情了，简单易用。
此外，django-emoji还提供了很多有用的api，更多功能和配置还是去看看它的[文档](https://github.com/gaqzi/django-emoji)吧！