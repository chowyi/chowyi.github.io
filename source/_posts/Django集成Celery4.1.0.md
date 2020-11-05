title: Django集成Celery4.1.0
permalink: how-to-configure-celery-4_1_0-with-django/
date: 2017-9-1 16:45:39
tags:
- Django
- Python
- Celery
categories:
- Django
---

之前做项目用的是 Django 1.8 配合 Celery 3.1.7。
现在使用最新的版本的 Celery 4.1.0，与之前的使用方法有一些变化，做一个简要记录。
本文仅介绍 Celery 4 的配置方法，针对有 Celery 3 及 Django 使用经验的读者。
详细的使用方法请参考 [Celery官网](http://docs.celeryproject.org/en/latest/index.html)
<!--more-->

不同点：

> Celery 4.0 supports Django 1.8 and newer versions. Please use Celery 3.1 for versions older than Django 1.8.

从 Celery 4.0 开始不再支持 Django 1.8 以下的版本（不包含1.8）。Django 1.8以下的版本需要使用 Celery 3.1 。

Celery 4.0 与 Django 集成不再使用 djcelery 插件。result backend 和 celery beat 也分别独立到两个扩展中，分别为 `django-celery-results` 和 `django-celery-beat`。


## 安装
```
pip install celery
pip install django-celery-results # 用于支持使用Django ORM做result backend
pip install django-celery-beat # 用于支持定时任务
```

## 配置

Django 项目的结构如下：
> - proj/
    - manage.py
    - proj/
        - __init__.py
        - settings.py
        - urls.py 

推荐做法是创建一个 `proj/proj/celery.py` 用来定义 Celery 定义。
```
from __future__ import absolute_import, unicode_literals
import os
from celery import Celery

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'ops.settings')

celery_app = Celery('ops', result_backend='django-db')

# Using a string here means the worker doesn't have to serialize
# the configuration object to child processes.
# - namespace='CELERY' means all celery-related configuration keys
#   should have a `CELERY_` prefix.
celery_app.config_from_object('django.conf:settings', namespace='CELERY')

# Load task modules from all registered Django app configs.
celery_app.autodiscover_tasks()

celery_app.loader.override_backends['django-db'] = 'django_celery_results.backends.database:DatabaseBackend'
```

在 `proj/proj/__init__.py` 下加入以下内容以确保 celery app 在 Django 项目启动时被加载：
```
from __future__ import absolute_import, unicode_literals

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app

__all__ = ['celery_app']
```

在其他的 Django app 下创建 celery task，推荐的做法是写在 `app/tasks.py` 文件下，这样就不用再去手动设置 `CELERY_IMPORT`。

在 `proj/proj/settings.py` 中添加：
```
# Celery
CELERY_RESULT_BACKEND = 'django-db'
# 使用redis 作为任务队列
CELERY_BROKER_URL = 'redis://:@localhost:6379/0'
```

## 启动worker
```
celery worker -A proj -l info
```

## 启动beat
```
celery -A proj beat -l info --scheduler django_celery_beat.schedulers:DatabaseScheduler
```
