title: Django 1.10 celery worker 启动报错
date: 2016-10-23 10:49:56
tags:
- Django
- Celery
categories:
- Django
---

```
AttributeError: type object 'BaseCommand' has no attribute 'option_list'
```
<!--more-->

没耐心的直接看文章最后有解决办法。

Django的最新的release版本已经到 1.10.2 了，django-celery我也安装了最新的版本 3.1.17 。

在确保环境正确，安装和配置无误后，我使用如下命令启动 celery worker ：
```
python manage.py celery worker -l info
```

然而结果并没有像我期望的那样，它报错了：
```
(python-django) zhouyi@aliyun:~/myproject/clip$ python manage.py celery worker -l info
Traceback (most recent call last):
  File "manage.py", line 22, in <module>
    execute_from_command_line(sys.argv)
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/django/core/management/__init__.py", line 367, in execute_from_command_line
    utility.execute()
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/django/core/management/__init__.py", line 359, in execute
    self.fetch_command(subcommand).run_from_argv(self.argv)
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/django/core/management/__init__.py", line 208, in fetch_command
    klass = load_command_class(app_name, subcommand)
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/django/core/management/__init__.py", line 40, in load_command_class
    module = import_module('%s.management.commands.%s' % (app_name, name))
  File "/usr/lib/python2.7/importlib/__init__.py", line 37, in import_module
    __import__(name)
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/djcelery/management/commands/celery.py", line 6, in <module>
    from djcelery.management.base import CeleryCommand
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/djcelery/management/base.py", line 59, in <module>
    class CeleryCommand(BaseCommand):
  File "/home/zhouyi/python-django/local/lib/python2.7/site-packages/djcelery/management/base.py", line 60, in CeleryCommand
    options = BaseCommand.option_list
AttributeError: type object 'BaseCommand' has no attribute 'option_list'
```

`BaseCommand` 没有 `option_list` 属性？我之前我用 Django 1.8.3的使用一切正常啊。

把 Django 1.8.3 和 Django 1.10.2 的源码都找出来，看看这个 `BaseCommand` 做了哪些修改。

找到 `django/core/management/base.py` 中 `BaseCommand` 的代码，果然在 Django 1.8.3 中有这么一段注释：
```
    ``option_list``
        This is the list of ``optparse`` options which will be fed
        into the command's ``OptionParser`` for parsing arguments.
        Deprecated and will be removed in Django 1.10.
        Use ``add_arguments`` instead.
```
原来 `option_list` 属性在 Django 1.8 中就已经过时并且在 Django 1.10 中被完全移除了。

再去看看 django-celery 3.1.17 中是怎么写的：

```
# djcelery/management/base.py 

class CeleryCommand(BaseCommand):
    options = BaseCommand.option_list
    skip_opts = ['--app', '--loader', '--config', '--no-color']
    requires_model_validation = VALIDATE_MODELS
    keep_base_opts = False
    stdout, stderr = sys.stdout, sys.stderr
    ...
```

啊哈，怪不得会报错。最新的 django-celery 3.1.17 没能赶上最新的 Django 1.10 的改变。

怎么办，自己改吗？ Django 1.10 也发布有一段时间了，先去 Github 上看看大家怎么说。

Github 上 django-celery 的最新版本是 3.2.0a1，还是测试版。翻翻看看，哈，management 模块有一条两个月前的注释：Fix Django 1.10+ 。
看起来靠谱！快看看代码。
```
# djcelery/management/base.py 

class CeleryCommand(BaseCommand):
    options = ()
    if hasattr(BaseCommand, 'option_list'):
        options = BaseCommand.option_list
    else:
        def add_arguments(self, parser):
        ...
```
果然是改了这里！感谢前辈们的无私贡献啊！

### 解决办法：
以下三种办法应该都可以，我用的是第一种。
1. 从 Github 下载 django-celery 3.2.0a1 的源码，把 management 模块拷贝到已安装的 3.1.17版本目录下替换就ok了。
2. 整个安装 django-celery 3.2.0a1。
3. 把 Django 的版本降到 1.8。

期待 django-celery 发布最新的release版。
