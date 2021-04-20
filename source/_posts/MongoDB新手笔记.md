---
title: MongoDB 新手笔记
date: 2021-04-06 19:10:37
permalink: green-hand-notes-about-mongodb/
tags:
- MongoDB
categories:
- MongoDB
---

说来惭愧，这么多年也没真正去学习过 MongoDB，最近工作中用到了，边用边学。  

文章标题写的是**新手笔记**，但这不是一篇入门教程，而是记录我作为一个新手在使用 MongoDB 过程中遇到的一些疑惑与问题。以下内容基于 MongoDB Server 4.0 测试编写（比如 `$convert` 方法是在 MongoDB Server 4.0 才加入的），参考时需注意版本。
<!--more-->

## ObjectId

### 什么是 ObjectId？

ObjectId 是一个 12 字节的对象，其中：  
- 4 字节用来表示以秒为单位的 Unix 时间戳
- 5 字节随机值
- 3 字节自增计数器，初始值随机

在 mongo Shell 中可以使用 `ObjectId.valueOf()` 方法获取它的十六进制字符串表示，字符串长度为 24。

### 在 Mongo Shell 中使用 ObjectId

```Javascript
> db.app.insert({"name": "app_A"})
WriteResult({ "nInserted" : 1 })
> db.app.find()
{ "_id" : ObjectId("606c4ae91c1866a586e387d4"), "name" : "app_A" }

// 因为类型不匹配，所以查询无结果
> db.app.find({"_id": "606c4ae91c1866a586e387d4"})
// 将字符串转换为 ObjectId
> db.app.find({"_id": ObjectId("606c4ae91c1866a586e387d4")})
{ "_id" : ObjectId("606c4ae91c1866a586e387d4"), "name" : "app_A" }
```

### 外键存 string 还是 ObjectId？

没有特别的理由时，应该优先选择 ObjectId。  
ObjectId 的优点：
- 占用空间更少。ObjectId 占空间12字节，ObjectId 的字符串表示占24字节。
- 包含信息更多。ObjectId 包含时间戳信息。

刚上手 MongoDB 时，不了解 ObjectId 的意义，外键都存成了 string 类型，导致做关联查询时遇到了字段类型不匹配的问题。


### 在 MongoDB Go Driver 中使用 ObjectId

在 MongoDB Go Driver 中，ObjectId 与 string 类型互相转换：
```golang
// ObjectId 转 string
doc.Id.Hex()
// string 转 ObjectId
import "go.mongodb.org/mongo-driver/bson/primitive"
primitive.ObjectIDFromHex(id)
```

## $elemMatch

与大部分编程语言中的 dict/map 不同，在 MongoDB 中 document 的 fields 是有序的。在查询document 是要注意 fields 的顺序。下面举个例子。  

一个常见的场景是，我们在资源上添加标签，然后通过标签查询。现有如下数据：
```Javascript
> db.resource.find().pretty()
{
	"_id" : ObjectId("606d66b42a998f22cc841dcf"),
	"name" : "Resource_A",
	"tags" : [
		{
			"TagKey" : "app",
			"TagValue" : "app_A"
		},
		{
			"TagKey" : "module",
			"TagValue" : "module_A"
		}
	]
}
```

现在通过标签查询资源：
```Javascript
> db.resource.count({
...     "tags": {
...         $all: [
...             {"TagKey": "module", "TagValue": "module_A"},
...             {"TagKey": "app", "TagValue": "app_A"},
...         ]
...     }
... })
// 找到一条记录，正确
1
```

下面改变一下 document 中 fields 的顺序：
```Javascript
> db.resource.count({
...     "tags": {
...         $all: [
...             {"TagValue": "module_A", "TagKey": "module"},
...             {"TagKey": "app", "TagValue": "app_A"},
...         ]
...     }
... })
// 找不到记录
0
```

要处理这种情况，可以使用 [$elemMatch(query)](https://docs.mongodb.com/manual/reference/operator/query/elemMatch/)。

> 官方文档说明：  
> The $elemMatch operator matches documents that contain an array field with at least one element that matches all the specified query criteria.

```Javascript
> db.resource.count({
...     "tags": {
...         $all: [
...             {$elemMatch: {"TagValue": "module_A", "TagKey": "module"}},
...             {$elemMatch: {"TagKey": "app", "TagValue": "app_A"}},
...         ]
...     }
... })
// 找到一条记录，正确
1
```

## $aggregate

常见的场景是通过外键进行关联查询。下面举例。

app 与 set 为一对多关系，set 与 module 为一对多关系。数据如下：
```Javascript
> db.app.find()
{ "_id" : ObjectId("606c4ae91c1866a586e387d4"), "name" : "app_A" }
db.set.find()
{ "_id" : ObjectId("606d82753b659cdfed5ed935"), "name" : "set_A", "appId" : ObjectId("606c4ae91c1866a586e387d4") }
db.module.find()
{ "_id" : ObjectId("606d665c2a998f22cc841dca"), "name" : "module_A", "setId" : "606d82753b659cdfed5ed935" }
```

**注意**：  
每一条 set 记录包含一个 appId 字段，类型为 ObjectId  
每一条 module 记录包含一个 setId 字段，类型为 string  
这里把 module 记录的 setId 字段设置为 string 类型是为了再次说明两者之间的差异。  

需求，已知 moduleId，查询出与之关联的 set 和 app 的 name。
查询如下：
```Javascript
db.module.aggregate([
    // 先在 module Collection 中按指定的 moduleId 过滤
    {$match: {"_id": ObjectId("606d665c2a998f22cc841dca")}},
    // 将 module 记录的 string 类型 setId 转换为 ObjectId 作为 setOid 字段
    // 以便下一步关联查询中将 setOid 与 set._id 判断相等
    {$addFields: {"setOid": {$convert: {input: "$setId", to: "objectId"}}}},
    // 关联查询 set Collection
    {
        $lookup: {
            from: "set",
            localField: "setOid",
            foreignField: "_id",
            as: "module_set"
        }
    },
    // 在关联的结果中抽取出新的字段 setName, appId
    // $arrayElemAt 用于获取 array 字段中指定索引的元素
    // 这里也可以用 $first 代替（MongoDB 4.4 可用）
    {
        $project: {
            "_id": 1,
            "name": 1,
            "setName": {"$arrayElemAt": ["$module_set.name", 0]},
            "appId": {"$arrayElemAt": ["$module_set.appId", 0]},
        }
    },
    // 此处 localField: appId 类型为 ObjectId，可直接与 app._id 比较无需转换
    {
        $lookup: {
            from: "app",
            localField: "appId",
            foreignField: "_id",
            as: "set_app"
        }
    },
    {
        $project: {
            "_id": 1,
            "name": 1,
            "setName": 1,
            "appName": {"$arrayElemAt": ["$set_app.name", 0]},
        }
    }
])
```

### 关键知识点

1. $lookup
2. $addFields
3. $convert
4. $arrayElemAt

具体的用法和说明已体现在上面代码的注释中，不再赘述。

## 在 $lookup 的 from 字段中使用表达式

目前（2021.04） $lookup 的 from 字段仅支持传入 string，不支持动态变量。可关注 [ISSUE SERVER-22497](https://jira.mongodb.org/browse/SERVER-22497)。
