title: Python中的一点小发现
date: 2016-5-31 21:01:56
tags:
- Python
categories:
- Python
---
使用Python快一年了，回头看看Python基础，还能时不时的发现一些小惊喜，在这里分享一下。
<!--more-->

### 字典生成式
假设现在我通过某种方法查询到了这样一个数据dept_data：
```
dept_data = [{'id' : 1, 'name' : 'xxx'}, {'id' : 2, 'name' : 'yyy'}, {'id' : 3, 'name' : 'zzz'}, {'id' : 2, 'name' : 'yyy'}]
```
这是一个list，它每一个元素都是一个dict，每个dict都有一个名为id的key，list的元素可能会有重复。
现在我想把dept_data中的元素去重，并且希望可以通过id来获取相应的元素。
一开始我是这么做的：
```
dept_map = {}
for data in dept_data:
    dept_map[data['id']] = data
print dept_map
```

后来我发现dict()可以把二元组的list转为dict，于是上面的代码就改为：
```
dept_map = dict([(data['id'], data) for data in dept_data])
print dept_map
```
嗯，用列表生成式加dict()省了两行代码，不过还够好看。

今天偶然想起了Python不仅有列表生成式，还有字典生成式，于是：
```
dept_map = {data['id'] : data for data in dept_data}
print dept_map
```
比第二种方式更加清晰直观，也更有效率。


### or的用法
有时我们会遇到这样的情况：
```
if user.name:
    name = user.name
else:
    name = '未知'
```

更简单的办法是三元运算符，在C/C++或是Java中是condition?a:b,在Python中则这样写：
```
name = user.name if user.name else '未知'
```
user.name写了两遍，似乎不够优雅。今天我发现原来还可以这么写：
```
name = user.name or '未知'
```
or 操作符会从左到右运算并返回第一个真值。
相应的，and 操作符会返回第一个假值，如果都为真，则返回最后一个值。