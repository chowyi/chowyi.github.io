title: Django Model的三种继承方式
permalink: three-types-of-inheriting-django-model/
date: 2017-9-5 19:03:53
tags:
- Django
- Python
categories:
- Django
---
使用 Django 的时间也不短了，这两天发现了一个之前没有注意到的有关 Django Model 继承方面的细节。[Django 的官方文档](https://docs.djangoproject.com/en/1.11/topics/db/models/)中有很详细的解释，查阅之后，在此记录分享。
<!--more-->

## 问题背景
在 Django 项目中我建立了三个模型：Server(服务器)，PMServer(物理机)，VMServer(虚拟机)。PMServer 和 VMServer 都继承自 Server。Server作为一个抽象的概念，我把它定义为抽象类。简化的代码如下：
```
from django.db import models

class Server(models.Model):
    class Meta:
        abstract = True

    hostname = models.CharField('主机名', max_length=50)
    ...

class PMServer(Server):
    frame = models.CharField('机架号', max_length=50)
    ...

class VMServer(Server):
    image_id = models.CharField('镜像ID', max_length=50)
```

这样做可以实现 PMServer 和 VMServer 均拥有 Server 下定义的字段，还可以分别定义它们所特有的字段。执行 Django 命令`makemigrations`和`migrate`会在数据库中分别创建两张表，两张表相对独立，都完整的包含了自己本身及父类的字段。

## 问题提出
上面的代码看起来没什么问题，但在实际开发过程中，如果需要查询某服务器，而不区分是 PMServer 还是 VMServer，使用如下语句查询：
```
Server.objects.all()
```

就会报错：`AttributeError: type object 'Server' has no attribute 'objects'`

## 问题分析
因为在 Server 的 Meta 下定义`abstract = True`,所以 Django ORM 并没有在数据库中为 Server 建立数据表，Server 类仅在代码层面起到包含公共字段的作用。从数据库层面来看，和直接定义完整的 PMServer 和 VMServer 而不使用父类没有什么区别。

## 尝试解决
删除父类 Server 中定义的`abstract = True`，重做`makemigrations`和`migrate`。
查看一下 Django 生成的 migration 文件中创建 VMServer的这一部分：
```
migrations.CreateModel(
            name='VMServer',
            fields=[
                ('server_ptr', models.OneToOneField(auto_created=True, on_delete=django.db.models.deletion.CASCADE, parent_link=True, primary_key=True, serialize=False, to='resourcedb.Server')),
                ('instance_name', models.CharField(max_length=50, verbose_name='\u5b9e\u4f8b\u540d\u79f0')),
                ('instance_id', models.CharField(max_length=50, verbose_name='\u5b9e\u4f8bid')),
                ('image_id', models.CharField(max_length=50, verbose_name='\u955c\u50cfid')),
                ('instance_charge_type', models.CharField(max_length=50, verbose_name='\u5b9e\u4f8b\u4ed8\u8d39\u65b9\u5f0f')),
                ('internet_charge_type', models.CharField(max_length=50, verbose_name='\u7f51\u7edc\u8ba1\u8d39\u7c7b\u578b')),
                ('created_time', models.DateTimeField(blank=True, null=True, verbose_name='\u521b\u5efa\u65f6\u95f4')),
                ('expired_time', models.DateTimeField(blank=True, null=True, verbose_name='\u8fc7\u671f\u65f6\u95f4')),
                ('status', models.CharField(max_length=50, verbose_name='\u5b9e\u4f8b\u72b6\u6001')),
            ],
            options={
                'verbose_name': '\u865a\u62df\u673a',
                'verbose_name_plural': '\u865a\u62df\u673a',
            },
            bases=('resourcedb.server',),
        )
```

在 fields 中第一行，Django 为 VMServer 添加了一个类型为 OneToOneField 的 server_ptr字段与 Server 做一对一关联。

现在数据库中会有三张表`resourcedb_server`,`resourcedb_pmserver`,`resourcedb_vmserver`。
查看一下表结构：
```
mysql> desc resourcedb_vmserver;
+----------------------+-------------+------+-----+---------+-------+
| Field                | Type        | Null | Key | Default | Extra |
+----------------------+-------------+------+-----+---------+-------+
| server_ptr_id        | int(11)     | NO   | PRI | NULL    |       |
| instance_name        | varchar(50) | NO   |     | NULL    |       |
| instance_id          | varchar(50) | NO   |     | NULL    |       |
| image_id             | varchar(50) | NO   |     | NULL    |       |
| instance_charge_type | varchar(50) | NO   |     | NULL    |       |
| internet_charge_type | varchar(50) | NO   |     | NULL    |       |
| created_time         | datetime    | YES  |     | NULL    |       |
| expired_time         | datetime    | YES  |     | NULL    |       |
| status               | varchar(50) | NO   |     | NULL    |       |
+----------------------+-------------+------+-----+---------+-------+
```

可以看到在 pmserver 和 vmserver 表中都包含一个`server_ptr_id`字段指向 server 表中的一条记录。PMServer 和 VMServer 的公共字段都保存在 server 表中，它们特有的字段分别保存在其相对应的表中。

此时就可以使用如下代码进行查询了：
```
Server.objects.all()
Server.objects.get(name='xxx')
...
```
PMServer 和 VMServer 均可以看做 Server 进行查询，它们的 id 共用同一个自增长序列。

## 文档解释

Django Model 中的继承基本和 Python 中的继承一致，只是所有的类都要继承自`django.db.models.Model`。
你只需要考虑你是否希望父类拥有独立的一张数据表，还是仅仅为子类提供公共字段。所有的信息都由子类提供。
Django 中有三种继承关系：
1. 通常你只是需要一个父类来保存子类所共有的字段而避免重复的在多个子类中定义，这个父类永远不打算单独使用，这种情况你可以选择 Abstract base classes。
2. 如果你要继承一个已经存在的 model，并且每个 model 都有其独立的数据表，你可以选择 Multi-table inheritance。
3. 如果你只想改变一个 model Python-level 的行为而不修改其字段，你可以选择 Proxy Model。

### Abstract base classes
如果你有一些 models 都包含一些公共的信息，Abstract base classes 就会很有用。你只要定义一个 base class 并把 Meta 下设置`abstract=True`即可。它本身不会被创建数据表，它的字段会被添加在子类的数据表中。要注意它与子类不能有同名的字段，否则 Django 会抛出异常。
例如：
```
from django.db import models

class CommonInfo(models.Model):
    name = models.CharField(max_length=100)
    age = models.PositiveIntegerField()

    class Meta:
        abstract = True

class Student(CommonInfo):
    home_group = models.CharField(max_length=5)
```
Student model 会有三个字段：name, age, home_group。CommonInfo 作为一个 abstract base model 不会像一个普通的 Django Model那样生成数据表，也没有 manager，也不能被实例化或直接的保存。它只在 Python-level 提供公共信息，在 database-level 仅为每个子类创建数据表。

### Multi-table inheritance
Django Model 支持的第二种继承方式是每一个 model 都自成一个模型，都拥有自己的数据表，可以被独立的查询或是创建。子类和父类之间通过一个一对一（OneToOneField）的关系关联。
例如：
```
from django.db import models

class Place(models.Model):
    name = models.CharField(max_length=50)
    address = models.CharField(max_length=80)

class Restaurant(Place):
    serves_hot_dogs = models.BooleanField(default=False)
    serves_pizza = models.BooleanField(default=False)
```
`Place`中的字段在`Restaurant`中都是可用的，及时数据存在不同的数据表中。以下的查询都是可以的：
```
>>> Place.objects.filter(name="Bob's Cafe")
>>> Restaurant.objects.filter(name="Bob's Cafe")
```

如果你有一个`Place`对象同时也是`Restaurant`，你可以用小写的类名从`Place`对象得到`Restaurant`对象：
```
>>> p = Place.objects.get(id=12)
# If p is a Restaurant object, this will give the child class:
>>> p.restaurant
<Restaurant: ...>
```

要注意，如果`p`不是`Restaurant`对象，那么会抛出`Restaurant.DoesNotExist`异常。

在`Restaurant`上自动创建的用于关联`Place`的`OneToOneField`如下：
```
place_ptr = models.OneToOneField(
    Place, on_delete=models.CASCADE,
    parent_link=True,
)
```

你也可以在`Restautrant`的定义下重写`OneToOneField`，要设置`parent_link=True`。

### Proxy models
有时你只希望修改一个 model 在 python-level 的行为，比如修改默认的 manager 或是添加一个方法，你就可以用 proxy model inheritace。你对 proxy model 所做的增删改的操作都会被保存在原始的 model 中。
定义一个 proxy model 你只需要在 Meta 中定义`proxy = True`，如：
```
from django.db import models

class Person(models.Model):
    first_name = models.CharField(max_length=30)
    last_name = models.CharField(max_length=30)

class MyPerson(Person):
    class Meta:
        proxy = True

    def do_something(self):
        # ...
        pass
```

`MyPerson`和`Person`操作的是同一张数据表。任何`Person`的实例也可以通过`MyPerson`来访问，反之亦然
```
>>> p = Person.objects.create(first_name="foobar")
>>> MyPerson.objects.get(first_name="foobar")
<MyPerson: foobar>
```

顺便学个英文短语，*and vise versa*，意为*反之亦然*。
> 文档原文： In particular, any new instances of Person will also be accessible through MyPerson, and vice-versa.

[官网](https://docs.djangoproject.com/en/1.11/topics/db/models/)还有关于继承 Meta 的介绍以及其他详细的说明。

## 结语
没事多看文档。
本次问题就出现在想当然的把 Server 配置成了`abstract = True`。
在选择方案，做配置时，一定要了解每一项选择，配置的含义。

