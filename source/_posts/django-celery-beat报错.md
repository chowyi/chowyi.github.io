title: django-celery beat 报错
permalink: periodictask-object-has-no-attr-in-django-celery-3_1_17
date: 2016-10-26 11:26:39
tags:
- Django
- Celery
categories:
- Django
---

```
AttributeError: 'PeriodicTask' object has no attribute '_default_manager'
```
<!--more-->

这是我在使用 Django 1.10.2 和 django-celery 3.1.17 做开发时遇到的问题，在使用命令:
```
python manage.py celery beat
```
启动 celery beat 一段时间后，会报错如下错误：

```
[2016-10-25 17:17:41,883: CRITICAL/MainProcess] beat raised exception <type 'exceptions.AttributeError'>: AttributeError("'PeriodicTask' object has no attribute '_default_manager'",)
Traceback (most recent call last):
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/celery/apps/beat.py", line 112, in start_scheduler
    beat.start()
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/celery/beat.py", line 490, in start
    self.sync()
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/celery/beat.py", line 493, in sync
    self.scheduler.close()
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/celery/beat.py", line 285, in close
    self.sync()
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/djcelery/schedulers.py", line 209, in sync
    self.schedule[name].save()
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/djcelery/schedulers.py", line 98, in save
    obj = self.model._default_manager.get(pk=self.model.pk)
AttributeError: 'PeriodicTask' object has no attribute '_default_manager'
```

在 stackoverflow 上查到有同学和我遇到了[相同的问题](http://stackoverflow.com/questions/39664493/using-django-celery-beat-locally-i-get-error-periodictask-object-has-no-attrib)，原来是 django-celery 的源码有些问题。

再去 Github 上看看，Github 上 django-celery 的最新版本是 3.2.0a1, 在 2016年6月30日已经有前辈修复了这个问题：

[fix django 1.10 issue: Access _default_manager on the model instance'…](https://github.com/celery/django-celery/commit/803234d03c8f03a149476365496263a921714bba)

```
# djcelery/schedulers.py 

     def save(self):
         # Object may not be synchronized, so only
         # change the fields we care about.
-        obj = self.model._default_manager.get(pk=self.model.pk)
+        obj = type(self.model)._default_manager.get(pk=self.model.pk)
         for field in self.save_fields:
             setattr(obj, field, getattr(self.model, field))
         obj.last_run_at = make_aware(obj.last_run_at)
```

## 解决办法：
办法一： 直接在 django-celery 3.1.17 的安装路径下修改源码。


办法二： 从 Github 下载最新的 django-celery 3.2.0a1 安装。 


## 启示：
1. 由于缺乏经验，我并没有仔细考虑到各软件之前版本不兼容的问题，想当然的认为最新的版本都是可用的。
django-celery 3.1.17 是2015年10月9日发布的， Django 1.10 是2016年8月1日发布的，因此会有一些不兼容的问题。
2. 在初见报错时，我也怀疑过是源码错误，但并不坚定。以后应该敢于质疑，敢于挑战，大神的代码也不能保证没有错误。当然，大多是时候还是要先排查自己的代码。
  
期待 django-celery 发布最新的release版。最后还是要感谢前辈们的贡献！

