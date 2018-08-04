title: SQLAlchemy初体验
date: 2016-04-16 17:27:35
tags:
- Python
- SQLAlchemy
categories:
- Python
---
使用Django已经有一段时间了，对Django的ORM印象深刻，它简单易用，也足够灵活。但是在自己做一些小实验小例子时，仅为了ORM而使用Django这么一个web框架显然不太合适。于是我找到了SQLAlchemy。从[官方](http://www.sqlalchemy.org/)介绍来看，SQLAlchemy是Python下的一款强大且灵活的ORM框架及SQL工具集。那么我就从[官方教程](http://docs.sqlalchemy.org/en/rel_1_0/orm/tutorial.html)入手，一边学习，一边感受一下吧。
<!--more-->

## 安装
使用pip或easy_install即可安装最新版本的SQLAlchemy。我安装的是当前的最新版本1.0.12。
```
pip install sqlalchemy
或者
easy_install sqlalchemy
```

## 版本检查 Install
```
>>> import sqlalchemy
>>> sqlalchemy.__version__
'1.0.12'
```

## 连接数据库 Connecting
教程中使用了仅在内存中的SQLite数据库(in-memory-only SQLite database)。使用```create_engine()```来连接：
```
>>> from sqlalchemy import create_engine
>>> engine = create_engine('sqlite:///:memory:', echo=True)
```
`echo`参数用来设置是否开启SQLAlchemy日志，此功能基于标准的Python`logging`模块。开启日志，我们可以看到SQLAlchemy生成的SQL语句。如果你不想看到那么多输出，把它设置为`False`就好了。

`create_engine()`的返回值是`Engine`的一个实例，它代表了到数据库的核心接口，适配不同的数据库和[DBAPI](http://docs.sqlalchemy.org/en/rel_1_0/glossary.html#term-dbapi)。

**懒连接 (Lazy Connecting)**
需要注意，使用`create_engine()`创建出`Engine`时，并没有真正的连接数据库，直到调用`Engine.execute()`或`Engine.connect()`等方法时，`Engine`才会真正建立一个连接到数据库，然后发送SQL脚本。

### 数据库连接字符串
`create_engine()`方法基于一个URL来创建`Engine`对象。URL遵循[RFC-1738](http://rfc.net/rfc1738.html)，通常包含username,password,hostname,database name等可选参数。典型的URL是这样的：
```
dialect+driver://username:password@host:port/database
```
使用`create_engine()`连接几种不同数据库的例子：

**Postgresql**
Postgresql dialect使用psycopg2作为默认DBAPI，也可以用纯python的pg8000作为替代：
```
# default
engine = create_engine('postgresql://scott:tiger@localhost/mydatabase')

# psycopg2
engine = create_engine('postgresql+psycopg2://scott:tiger@localhost/mydatabase')

# pg8000
engine = create_engine('postgresql+pg8000://scott:tiger@localhost/mydatabase')
```

**MySQL**
MySQL dialect使用mysql-python作为默认DBAPI。MySQL有许多可用的DBAPI，包括MySQL-connector-python和OurSQL：
```
# default
engine = create_engine('mysql://scott:tiger@localhost/foo')

# mysql-python
engine = create_engine('mysql+mysqldb://scott:tiger@localhost/foo')

# MySQL-connector-python
engine = create_engine('mysql+mysqlconnector://scott:tiger@localhost/foo')

# OurSQL
engine = create_engine('mysql+oursql://scott:tiger@localhost/foo')
```

**Oracle**
Oracle dialect使用cx_oracle作为默认的DBAPI：
```
engine = create_engine('oracle://scott:tiger@127.0.0.1:1521/sidname')

engine = create_engine('oracle+cx_oracle://scott:tiger@tnsname')
```

**Microsoft SQL Server**
SQL Server dialect使用pyodbc作为默认的DBAPI，pymssql也是可用的：
```
# pyodbc
engine = create_engine('mssql+pyodbc://scott:tiger@mydsn')

# pymssql
engine = create_engine('mssql+pymssql://scott:tiger@hostname:port/dbname')
```

**SQLite**
SQLite连接文件数据，默认使用Python内建的`sqlite3`。
由于SQLite连接本地文件，所以URL格式略有不同。
相对路径：
```
# sqlite://<nohostname>/<path>
# where <path> is relative:
engine = create_engine('sqlite:///foo.db')
```
绝对路径：
```
#Unix/Mac - 4 initial slashes in total
engine = create_engine('sqlite:////absolute/path/to/foo.db')
#Windows
engine = create_engine('sqlite:///C:\\path\\to\\foo.db')
#Windows alternative using raw string
engine = create_engine(r'sqlite:///C:\path\to\foo.db')
```
使用SQLite内存数据库：
```
engine = create_engine('sqlite://')
```

## 声明映射 Declare a Mapping
使用ORM时，配置过程从描述数据表开始，然后定义我们将要映射到这些表的类。在现代的SQLAlchemy中，这两个任务通常被放在一起。使用[Declarative](http://docs.sqlalchemy.org/en/rel_1_0/orm/extensions/declarative/index.html)系统，可以让我们创建包含了表描述的类。
使用Declarative System的类映射被定义在*declarative base class*中，它包含了类和表的关系。我们的项目中通常只需要一个该类的实例。我们通过`declarative_base()`来创建base类，像下面这样：
```
>>> from sqlalchemy.ext.declarative import declarative_base

>>> Base = declarative_base()
```
有了"base"，我们就可以基于它定义其他需要映射的类了。
下面的示例展示了把`User`类映射到`users`数据表，类的定义包含了表明，列名及数据类型：
```
>>> from sqlalchemy import Column, Integer, String
>>> class User(Base):
...     __tablename__ = 'users'
...
...     id = Column(Integer, primary_key=True)
...     name = Column(String)
...     fullname = Column(String)
...     password = Column(String)
...
...     def __repr__(self):
...        return "<User(name='%s', fullname='%s', password='%s')>" % (
...                             self.name, self.fullname, self.password)
```
*Tip*
`__repr__()`方法的定义是可选的，我们实现它仅仅是为了以友好的格式展示`User`对象。`

## Session
`Session`用来处理ORM与数据库之间的交流。当我们第一次建立起项目时，我们需要定义一个`Session`类用来创建新的`Session`对象：
```
>>> from sqlalchemy.orm import sessionmaker
>>> Session = sessionmaker(bind=engine)
```
如果你还没有`Engine`对象，你也可以在创建`Engine`对象后把它连接到`Session`：
```
>>> Session = sessionmaker()
>>> Session.configure(bind=engine)  # once engine is available
```
通过`Session`类可以创建出绑定到我们数据库的`Session`对象。
```
>>> session = Session()
```
这个`session`对象关联了我们的`engine`，但它还没有打开任何连接。当它第一次被使用的时候，它会从`engine`维护的连接池中获取一个连接，它保持连接直到我们提交了(commit)所有的更改或者关闭了session对象。

## 添加和更新对象
为了持久化我们的`User`对象，我们使用`add()`方法把它加入到`Session`中：
```
>>> ed_user = User(name='ed', fullname='Ed Jones', password='edspassword')
>>> session.add(ed_user)
```
这时，我们称这个对象为**pending**状态。现在还没有SQL脚本被发送到数据库，这个对象也没有被存到数据库中。`Session`会在需要的时候发出SQL把对象保存到数据库中，这一过程叫做**flush**。当我们在数据库中查询`Ed Jones`时，所有pending状态的信息会首先flushed，然后立刻执行查询。
举个例子，我们创建一个查询，查到的结果与我们添加的对象是相等的：
```
>>> our_user = session.query(User).filter_by(name='ed').first() # doctest:+NORMALIZE_WHITESPACE
>>> our_user
<User(name='ed', fullname='Ed Jones', password='edspassword')>
>>> ed_user is our_user
True
```

我们还可以使用`add_all()`方法一次添加多个`User`对象：
```
>>> session.add_all([
...     User(name='wendy', fullname='Wendy Williams', password='foobar'),
...     User(name='mary', fullname='Mary Contrary', password='xxg527'),
...     User(name='fred', fullname='Fred Flinstone', password='blah')])
```
现在我们觉得Ed的密码不够安全，我们来改一下：
```
>>> ed_user.password = 'f8s7ccs'
```
当你做出了修改，`Session`是知道的：
```
>>> session.dirty
IdentitySet([<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>])
```
还有三个新的`User`对象处于pending状态：
```
>>> session.new  # doctest: +SKIP
IdentitySet([<User(name='wendy', fullname='Wendy Williams', password='foobar')>,
<User(name='mary', fullname='Mary Contrary', password='xxg527')>,
<User(name='fred', fullname='Fred Flinstone', password='blah')>])
```
通过`commit()`方法告知`Session`我们想要保存修改并提交此次transaction。`Session`会使用`UPDATE`语句更改"ed"的密码，并用`INSERT`语句添加我们新创建的三个`User`对象：
```
>>> session.commit()
```
`commit()`会flush所有的改变保存到数据库并提交此次transaction。现在session所引用的连接资源重新回到了连接池中。这个session接下来的操作会维持在一个新的transaction中，在需要的时候会重新获取连接资源。

我们查看一下Ed的`id`属性，之前还是`None`,现在已经有值了：
```
>>> ed_user.id
1
```

## 回滚 Rolling Back
因为`Session`的工作使用了transaction，所以我们可以回滚所做的改变。
现在我们改变`ed_user`的name，然后再添加一个错误的user对象：
```
>>> ed_user.name = 'Edwardo'
>>> fake_user = User(name='fakeuser', fullname='Invalid', password='12345')
>>> session.add(fake_user)
```
查询session，我们可以看到刚才的改变已经被flushed到当前的transaction中了：
```
>>> session.query(User).filter(User.name.in_(['Edwardo', 'fakeuser'])).all()
[<User(name='Edwardo', fullname='Ed Jones', password='f8s7ccs')>, <User(name='fakeuser', fullname='Invalid', password='12345')>]
```
现在回滚，我们可以看到`ed_user`的name已经改回了`ed`，`fake_user`也从session中剔除了：
```
>>> session.rollback()

>>> ed_user.name
u'ed'
>>> fake_user in session
False
```
使用SELECT来看看我们到底对数据库做了哪些改变：
```
>>> session.query(User).filter(User.name.in_(['ed', 'fakeuser'])).all()
[<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>]
```

## 查询 Querying
使用`Session`的`query()`方法可以创建一个`Query`的对象。这个方法可以接受多个参数，可以是classes和class的描述符。下面我们指定一个加载了`User`实例的`Query`对象。查询结果会返回`User`列表：
```
>>> for instance in session.query(User).order_by(User.id):
...     print instance.name, instance.fullname
ed Ed Jones
wendy Wendy Williams
mary Mary Contrary
fred Fred Flinstone
```
`query()`还可以接受基于列的实体作为参数，返回值是tuples的列表：
```
>>> for name, fullname in session.query(User.name, User.fullname):
...     print name, fullname
ed Ed Jones
wendy Wendy Williams
mary Mary Contrary
fred Fred Flinstone
```
还可以这样：
```
>>> for row in session.query(User, User.name).all():
...    print row.User, row.name
<User(name='ed', fullname='Ed Jones', password='f8s7ccs')> ed
<User(name='wendy', fullname='Wendy Williams', password='foobar')> wendy
<User(name='mary', fullname='Mary Contrary', password='xxg527')> mary
<User(name='fred', fullname='Fred Flinstone', password='blah')> fred
```
[官方教程](http://docs.sqlalchemy.org/en/rel_1_0/orm/tutorial.html#querying)这么介绍：
> The tuples returned by `Query` are named tuples, supplied by the `KeyedTuple` class, and can be treated much like an ordinary Python object. The names are the same as the attribute’s name for an attribute, and the class name for a class

**我的疑问**
上面列子中的`row`用法的确同官方教程中介绍的`KeyedTuple`一样，但我在试验中得出的是下面的结果：
```
from sqlalchemy.util._collections import KeyedTuple
for row in session.query(User, User.name).all():
    print type(row), isinstance(row, KeyedTuple)

# 输出
<class 'sqlalchemy.util._collections.result'> False
<class 'sqlalchemy.util._collections.result'> False
<class 'sqlalchemy.util._collections.result'> False
<class 'sqlalchemy.util._collections.result'> False
```
而且我在源码中并没有找到result这个类，对此表示疑惑。希望知道原因的朋友能指点一下！

你还可以使用`label()`来控制查询结果每一列的名字：
```
>>> for row in session.query(User.name.label('name_label')).all():
...    print(row.name_label)
ed
wendy
mary
fred
```
看了SQL就会发现其实就是AS语句：
```
#SQL
SELECT users.name AS name_label
FROM users
()
```

还可以给整个实体例如`User`起别名，使用`aliased()` :
```
>>> from sqlalchemy.orm import aliased
>>> user_alias = aliased(User, name='user_alias')

>>> for row in session.query(user_alias, user_alias.name).all():
...    print(row.user_alias)
<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>
<User(name='wendy', fullname='Wendy Williams', password='foobar')>
<User(name='mary', fullname='Mary Contrary', password='xxg527')>
<User(name='fred', fullname='Fred Flinstone', password='blah')>
```

`Query`的基本操作包含了LIMIT和OFFSET，但更简单的方法是使用python的切片slices：
```
session.query(User.name).offset(1).limit(2)
session.query(User.name)[1:3]

print type(session.query(User.name).offset(1).limit(2)) # <class 'sqlalchemy.orm.query.Query'>
print type(session.query(User.name)[1:3]) # <type 'list'>
```
这两种语句的查询结果是一样的，不同的是前者返回的是一个`Query`对象，后者是一个list。
毫无疑问，前者使用了SQL中的LIMIT和OFFSET语句。那么后者是从数据库中取出全部结果后再做切片的吗？这还有待考证。

过滤有两种方法，使用`filter_by()`或者`filter()`。
`filter_by()`使用关键字参数：
```
>>> for name, in session.query(User.name).filter_by(fullname='Ed Jones'):
...    print(name)
ed
```
`filter()`使用Python表达式：
```
>>> for name, in session.query(User.name).filter(User.fullname=='Ed Jones'):
...    print(name)
ed
```

`Query`对象上的大部分方法调用返回的仍是`Query`对象，这样你可以添加更多条件，比如过滤两次或更多，它们使用`AND`关系连接：
```
>>> for user in session.query(User).filter(User.name=='ed').filter(User.fullname=='Ed Jones'):
...    print(user)
<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>
```
但是如果你在`offset()`和`limit()`之后调用`filter()`,会引发错误：
```
session.query(User.name).offset(1).limit(2).filter(User.name=="wendy")

# output
sqlalchemy.exc.InvalidRequestError: Query.filter() being called on a Query which already has LIMIT or OFFSET applied. To modify the row-limited results of a  Query, call from_self() first.  Otherwise, call filter() before limit() or offset() are applied.
```
错误信息解释的也很明白了，你只需要先使用`from_self()`方法，像这样：
```
session.query(User.name).offset(1).limit(2).from_self().filter(User.name=="wendy")
```

### 常见过滤操作符

下面是用在`filter()`中的一些常见操作符：
+ equals
	```
	query.filter(User.name == 'ed')
	```
+ not equals
	```
	query.filter(User.name != 'ed')
	```
+ LIKE
	```
	query.filter(User.name.like('%ed%'))
	```
+ IN
	```
	query.filter(User.name.in_(['ed', 'wendy', 'jack']))

	# works with query objects too:
	query.filter(User.name.in_(
	        session.query(User.name).filter(User.name.like('%ed%'))
	))
	```
+ NOT IN
	```
	query.filter(~User.name.in_(['ed', 'wendy', 'jack']))
	```
+ IS NULL
	```
	query.filter(User.name == None)

	# alternatively, if pep8/linters are a concern
	query.filter(User.name.is_(None))
	```
+ IS NOT NULL
	```
	query.filter(User.name != None)

	# alternatively, if pep8/linters are a concern
	query.filter(User.name.isnot(None))
	```
+ AND
	```
	# use and_()
	from sqlalchemy import and_
	query.filter(and_(User.name == 'ed', User.fullname == 'Ed Jones'))

	# or send multiple expressions to .filter()
	query.filter(User.name == 'ed', User.fullname == 'Ed Jones')

	# or chain multiple filter()/filter_by() calls
	query.filter(User.name == 'ed').filter(User.fullname == 'Ed Jones')
	```
+ OR
	```
	from sqlalchemy import or_
	query.filter(or_(User.name == 'ed', User.name == 'wendy'))
	```
+ MATCH
	```
	query.filter(User.name.match('wendy'))
	```
	**注意**
	`match()`使用数据库指定的`MATCH`或`CONTAINS`函数。它的行为因数据库后端不同而不同。在某些数据库中是不能用的，比如SQLite。

### 返回Lists和Scalars
`Query`对象上有一些方法会立刻执行SQL并返回结果。
+ `all()` returns a list:
	```
	>>> query = session.query(User).filter(User.name.like('%ed')).order_by(User.id)
	>>> query.all()
	[<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>,
	      <User(name='fred', fullname='Fred Flinstone', password='blah')>]
    ```
+ `first()` 返回查询结果中的第一项：
	```
	>>> query.first()
	<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>
	```
+ `one()` 取所有的行(rows)，如果结果不止一个，会报错`MultipleResultsFound`。如果结果一个也没有，会报错`NoResultFound`。
+ `one_or_none()` 同`one()`相似，只是没有结果是返回`None`而不报错。
+ `scalar()` 会先调用`one()`，如果没有报错，会返回行的第一列：
	```
	>>> query = session.query(User.id).filter(User.name == 'ed').order_by(User.id)
	>>> query.scalar()
	1
	```

### 使用SQL字符串
`Query`的大多数方法还可以通过使用`text()`来接受字符串作为参数。例如`filter()`和`order_by()`：
```
>>> from sqlalchemy import text
>>> for user in session.query(User).filter(text("id<224")).order_by(text("id")).all():
...     print(user.name)
ed
wendy
mary
fred
```
SQL字符串还可以通过`params()`方法指定参数：
```
>>> session.query(User).filter(text("id<:value and name=:name")).params(value=224, name='fred').order_by(User.id).one()
<User(name='fred', fullname='Fred Flinstone', password='blah')>
```
要使用完整的SQL字符串表达式，使用`from_statement()`方法：
```
>>> session.query(User).from_statement(
...                     text("SELECT * FROM users where name=:name")).params(name='ed').all()
[<User(name='ed', fullname='Ed Jones', password='f8s7ccs')>]
```

### 计数 Counting
`Query`有个便捷的计数方法`count()`：
```
>>> session.query(User).filter(User.name.like('%ed')).count()
2
```
来看看这行代码产生的SQL:
```
SELECT count(*) AS count_1
FROM (SELECT users.id AS users_id,
                users.name AS users_name,
                users.fullname AS users_fullname,
                users.password AS users_password
FROM users
WHERE users.name LIKE ?) AS anon_1
('%ed',)
```
在某些情况下，使用`SELECT count(*) FROM table`会更简单。但现在新版本的SQLAlchemy总是用更加精确的SQL来表达更加准确的含义。

有时我们需要指定对哪些项来计数，可以使用`func.count()`表达式：
```
>>> from sqlalchemy import func
>>> session.query(func.count(User.name), User.name).group_by(User.name).all()
[(1, u'ed'), (1, u'fred'), (1, u'mary'), (1, u'wendy')]
```

要产生`SELECT count(*) FROM table`这样简单的SQL，我们可以这样做：
```
>>> session.query(func.count('*')).select_from(User).scalar()
4
```
更简单的做法，可以不使用`select_from()`:
```
>>> session.query(func.count(User.id)).scalar()
4
```

## 建立关系 Building a Relationship
现在来想想我们的第二张表(table)吧，它与`User`关联，可以被映射和查询。假设在我们的系统中用户可以保存多个email地址，那么，我们要将`users`表通过一对多的关系关联到一个叫做`addresses`的新表上。我们通过定义`Address`类来映射出这张表：
```
>>> from sqlalchemy import ForeignKey
>>> from sqlalchemy.orm import relationship

>>> class Address(Base):
...     __tablename__ = 'addresses'
...     id = Column(Integer, primary_key=True)
...     email_address = Column(String, nullable=False)
...     user_id = Column(Integer, ForeignKey('users.id'))
...
...     user = relationship("User", back_populates="addresses")
...
...     def __repr__(self):
...         return "<Address(email_address='%s')>" % self.email_address

>>> User.addresses = relationship(
...     "Address", order_by=Address.id, back_populates="user")
```

> **注意**
`relationship.back_populates`参数是SQLAlchemy中`relationship.backref`的更新版本。`relationship.backref`仍然可以使用。`relationship.back_populates`除了一点冗长和更易操作，其实是一样的。

与Django的模型定义相比，SQLAlchemy中的写法似乎稍微麻烦了一些，我想应该有它的道理吧，再深入学习也许能体会到。用过Django的同学应该不难看懂。这里[官方教程](http://docs.sqlalchemy.org/en/rel_1_0/orm/tutorial.html#building-a-relationship)介绍的很详细，我觉得是有点啰嗦了，有兴趣的同学可以仔细看看。

现在我们需要在数据库中创建出`addresses`表，因此我们会从metadata中发起一个新的CREATE语句。不必担心，已经创建的表将会被跳过：
```
>>> Base.metadata.create_all(engine)
```

## 使用关联对象 Working with Related Objects
现在当我们创建一个`User`对象，它会有一个空的`addresses`容器(collection)。collection可以有多种类型，比如集合set或是字典dictionary，默认情况下是Python的列表list。
```
>>> jack = User(name='jack', fullname='Jack Bean', password='gjffdd')
>>> jack.addresses
[]
```
我们可以自由的对`User`对象添加`Address`对象。
```
>>> jack.addresses = [
...                 Address(email_address='jack@google.com'),
...                 Address(email_address='j25@yahoo.com')]
```

添加了关系的元素可以向下面这样双向引用，这种行为通过Python实现而不需要任何SQL：
```
>>> jack.addresses[1]
<Address(email_address='j25@yahoo.com')>

>>> jack.addresses[1].user
<User(name='jack', fullname='Jack Bean', password='gjffdd')>
```
我们把`User`对象`Jack Bean`commit到数据库，通过叫做级联(cascading)的过程，两个`Address`对象也会被保存到数据库中。
```
>>> session.add(jack)
>>> session.commit()
```
现在查询Jack，我们仅仅取回了Jack，没有任何有关Jack的addresses的SQL：
```
>>> jack = session.query(User).filter_by(name='jack').one()
>>> jack
<User(name='jack', fullname='Jack Bean', password='gjffdd')>

# SQL
BEGIN (implicit)
SELECT users.id AS users_id,
        users.name AS users_name,
        users.fullname AS users_fullname,
        users.password AS users_password
FROM users
WHERE users.name = ?
('jack',)
```
查看`addresses`容器时，才会发出查询`addresses`的SQL：
```
>>> jack.addresses
[<Address(email_address='jack@google.com')>, <Address(email_address='j25@yahoo.com')>]

# SQL
SELECT addresses.id AS addresses_id,
        addresses.email_address AS
        addresses_email_address,
        addresses.user_id AS addresses_user_id
FROM addresses
WHERE ? = addresses.user_id ORDER BY addresses.id
(5,)
```
这是关系懒加载(lazy loading relationship)的一个例子。

## 关联查询 Querying with Joins
现在我们有两张表了，可以演示`Query`更多的特性了。
下面我们同时加载`User`和`Address`，使用`Query.filter()`时可以看作它们的列已经关联在一起了：
```
>>> for u, a in session.query(User, Address).\
...                     filter(User.id==Address.user_id).\
...                     filter(Address.email_address=='jack@google.com').\
...                     all():
...     print(u)
...     print(a)
<User(name='jack', fullname='Jack Bean', password='gjffdd')>
<Address(email_address='jack@google.com')>
```

使用`Query.join()`时最简单的办法来达到标准的SQL JOIN语法：
```
>>> session.query(User).join(Address).filter(Address.email_address=='jack@google.com').all()
[<User(name='jack', fullname='Jack Bean', password='gjffdd')>]

# SQL
SELECT users.id AS users_id,
        users.name AS users_name,
        users.fullname AS users_fullname,
        users.password AS users_password
FROM users JOIN addresses ON users.id = addresses.user_id
WHERE addresses.email_address = ?
('jack@google.com',)
```

`Query.join()`知道该如何连接`User`和`Address`因为它们之间有且仅有一个外键关系。如果没有外键或是有多个，用下面的几种形式之一，`Query.join()`会更好的工作：
```
query.join(Address, User.id==Address.user_id)    # explicit condition
query.join(User.addresses)                       # specify relationship from left to right
query.join(Address, User.addresses)              # same, with explicit target
query.join('addresses')                          # same, using a string
```

正如你所期望的，使用`outerjoin()`可以达到外关联(outer join)：
```
query.outerjoin(User.addresses)   # LEFT OUTER JOIN
```
对数据库的各种关联关系我了解的很少，也是需要加强的地方。更详细的文档可以看[这里](http://docs.sqlalchemy.org/en/rel_1_0/orm/query.html#sqlalchemy.orm.query.Query.join)。


### 使用别名 Using Aliases
多表查询时，有时同一张表会被引用不止一次，这时就需要使用别名(Aliases)。下面我们把`Address`关联了两次，来查询两个email地址不一样的用户：
```
>>> from sqlalchemy.orm import aliased
>>> adalias1 = aliased(Address)
>>> adalias2 = aliased(Address)
>>> for username, email1, email2 in \
...     session.query(User.name, adalias1.email_address, adalias2.email_address).\
...     join(adalias1, User.addresses).\
...     join(adalias2, User.addresses).\
...     filter(adalias1.email_address=='jack@google.com').\
...     filter(adalias2.email_address=='j25@yahoo.com'):
...     print username, email1, email2
jack jack@google.com j25@yahoo.com
```

### Using EXISTS
在SQL中EXISTS关键字是一个布尔运算符，当给定的表达式包含任何行时，就会返回True。在许多情况下它可以用来代替joins。
下面是EXISTS的一个使用实例：
```
>>> from sqlalchemy.sql import exists
>>> stmt = exists().where(Address.user_id==User.id)
>>> for name, in session.query(User.name).filter(stmt):
...     print(name)
jack
```

`Query`的几个操作符也会自动的调用EXISTS。上面的例子也可以用下面的方式表达：
```
>>> for name, in session.query(User.name).\
...         filter(User.addresses.any()):
...     print(name)
jack
```

`any()`也可以用条件来做限制：
```
>>> for name, in session.query(User.name).\
...     filter(User.addresses.any(Address.email_address.like('%google%'))):
...     print(name)
jack
```

`has()`和`any()`类似，它是多对一关系的操作符（注意`~`代表NOT）：
```
>>> session.query(Address).\
...         filter(~Address.user.has(User.name=='jack')).all()
[]
```

### 常用关系运算符 Common Relationship Operators
+ __eq__() (many-to-one “equals” comparison):
```
query.filter(Address.user == someuser)
```

+ __ne__() (many-to-one “not equals” comparison):
```
query.filter(Address.user != someuser)
```

+ IS NULL (many-to-one comparison, also uses __eq__()):
```
query.filter(Address.user == None)
```

+ contains() (used for one-to-many collections):
```
query.filter(User.addresses.contains(someaddress))
```

+ any() (used for collections):
```
query.filter(User.addresses.any(Address.email_address == 'bar'))

# also takes keyword arguments:
query.filter(User.addresses.any(email_address='bar'))
```

+ has() (used for scalar references):
```
query.filter(Address.user.has(name='ed'))
```

+ Query.with_parent() (used for any relationship):
```
session.query(Address).with_parent(someuser, 'addresses')
```

## 删除 Deleting
我们来试着删除`jack`看看会发生什么。
```
>>> session.delete(jack)
>>> session.query(User).filter_by(name='jack').count()
0
```
目前为止，一切正常。那Jack的那些`Address`对象呢？
```
>>> session.query(Address).filter(
...     Address.email_address.in_(['jack@google.com', 'j25@yahoo.com'])
...  ).count()
2
```
看，它们还在！分析SQL我们可以看到每个address的`user_id`列被设置为NULL，而此address行并没有删除。除非你明确的指定，SQLAlchemy不会做级联删除。

### 配置级联删除 Configuring delete/delete-orphan Cascade
我们可以配置`User.addresses`关系上的**cascade**选项来改变这种行为。虽然SQLAlchemy允许你在任何时候添加新的属性或是关系，但是这样需要删除已经存在的关系。所以我们需要推翻之前的关系映射重来。现在关闭`Session`：
```
session.close()
```
现在我们重新声明`User`和`Address`类：
```
class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    name = Column(String)
    fullname = Column(String)
    password = Column(String)

    addresses = relationship("Address", back_populates='user', cascade="all, delete, delete-orphan")

    def __repr__(self):
       return "<User(name='%s', fullname='%s', password='%s')>" % (
                            self.name, self.fullname, self.password)

class Address(Base):
    __tablename__ = 'addresses'
    id = Column(Integer, primary_key=True)
    email_address = Column(String, nullable=False)
    user_id = Column(Integer, ForeignKey('users.id'))
    user = relationship("User", back_populates="addresses")
    def __repr__(self):
        return "<Address(email_address='%s')>" % self.email_address     
```
注意，这次我们把关系定义在`User`类中。
现在，删除Jack的同时也会删除与这个user关联的`Address`对象：
```
>>> session.delete(jack)

>>> session.query(User).filter_by(name='jack').count()
0

>>> session.query(Address).filter(
...    Address.email_address.in_(['jack@google.com', 'j25@yahoo.com'])
... ).count()
0
```

## 创建多对多的关系 Building a Many To Many Relationship
现在来建立一个多对多的关系。假设我们有一个博客应用，文章`BlogPost`和关键词`Keyword`就是多对多关系。

我们需要创建一个没有映射的`Table`作为中间表。
```
from sqlalchemy import Table, Text
# association table
post_keywords = Table('post_keywords', Base.metadata,
    Column('post_id', ForeignKey('posts.id'), primary_key=True),
    Column('keyword_id', ForeignKey('keywords.id'), primary_key=True)
)
```
然后我们定义`BlogPost`和`Keyword`：
```
class BlogPost(Base):
    __tablename__ = 'posts'

    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'))
    headline = Column(String(255), nullable=False)
    body = Column(Text)

    # many to many BlogPost<->Keyword
    keywords = relationship('Keyword',
                            secondary=post_keywords,
                            back_populates='posts')

    def __init__(self, headline, body, author):
        self.author = author
        self.headline = headline
        self.body = body

    def __repr__(self):
        return "BlogPost(%r, %r, %r)" % (self.headline, self.body, self.author)


class Keyword(Base):
    __tablename__ = 'keywords'

    id = Column(Integer, primary_key=True)
    keyword = Column(String(50), nullable=False, unique=True)
    posts = relationship('BlogPost',
                         secondary=post_keywords,
                         back_populates='keywords')

    def __init__(self, keyword):
        self.keyword = keyword
```
现在我们还希望`BlogPost`类能有一个`author`字段：
```
BlogPost.author = relationship(User, back_populates="posts")
User.posts = relationship(BlogPost, back_populates="author", lazy="dynamic")
```

创建新表：
```
Base.metadata.create_all(engine)
```

添加一些数据：
```
wendy = session.query(User).filter_by(name='wendy').one()
post = BlogPost("Wendy's Blog Post", "This is a test", wendy)
session.add(post)


post.keywords.append(Keyword('wendy'))
post.keywords.append(Keyword('firstpost'))
```

下面是几种不同的查询示例：
```
>>> session.query(BlogPost).filter(BlogPost.keywords.any(keyword='firstpost')).all()
[BlogPost("Wendy's Blog Post", 'This is a test', <User(name='wendy', fullname='Wendy Williams', password='foobar')>)]
```

```
>>> session.query(BlogPost).\
...             filter(BlogPost.author==wendy).\
...             filter(BlogPost.keywords.any(keyword='firstpost')).\
...             all()
[BlogPost("Wendy's Blog Post", 'This is a test', <User(name='wendy', fullname='Wendy Williams', password='foobar')>)]
```

```
>>> wendy.posts.filter(BlogPost.keywords.any(keyword='firstpost')).all()
[BlogPost("Wendy's Blog Post", 'This is a test', <User(name='wendy', fullname='Wendy Williams', password='foobar')>)]
```

## 更多参考 Further Reference
[SQLAlchemy 1.0 Documentation](http://docs.sqlalchemy.org/en/rel_1_0/index.html)

## 后记
这是我第一次接触SQLAlchemy，文章是我一边学习一边记录下来的。文章中的代码大部分来自SQLAlchemy的官方文档，我都亲自做了实验并运行成功。我尽力保证文章的准确性，既是对自己的要求，也不希望因我的错误而误导读者。但因个人技术水平有限，文章中难免会有不准确的地方，希望发现错误的朋友能慷慨向我指出。学习SQLAlchemy还是请以官方文档为准。