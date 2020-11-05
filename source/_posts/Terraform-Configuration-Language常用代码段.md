---
title: Terraform Configuration Language常用代码段
date: 2020-07-17
permalink: terraform-configuration-language-common-usages/
tags: Terraform
categories: DevOps
---


近两个月写了不少 Terraform 的 Configuration Language, 从入门到熟悉，用起来也渐渐得心应手。从名字上看这是一种配置语言，但得益于 Terraform Provider 强大的可扩展性，它不仅可以像 Golang 或是 Python （Terraform本来就是使用Golang开发的）一样处理字符串，读写本地文件，甚至还可以发起网络请求！  

这篇文章既不是教程（初学者建议先看官方文档学习），也不是进阶的知识分享。我在这里记录一些常用的代码段，希望像我一样的新手可以少走些弯路。

<!--more-->

## Random Password

场景描述：使用 Terraform 生成随机密码，密码长度为`16`, 特殊字符`_%@`。

Terraform 官方提供了名为 `random` 的 provider，[文档](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/password)。

在找到这个 provider 之前，我曾尝试过使用 `timestamp`,`sha1`,`uuid`,`replace`等 built-in function 来生成密码，既麻烦又不安全。

```
# main.tf

provider "random" {}

resource "random_password" "password" {
  length = 16
  special = true
  override_special = "_%@"
}

resource "aws_db_instance" "example" {
  password = random_password.password.result
  # ...省略部分无关参数...
}
```


## Shuffle

场景描述：从已定义的变量 `locals.default_zone` 中随机选择一个 `zone_id`，然后再随机选择一个该 zone 对应的 `vswitch_id`。

使用了`random` 的 provider 中的 `random_shuffle`，[文档](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/shuffle)。

本段代码展示了如何随机选取列表中的项目，并且将第一次随机的输出作为第二次随机的输入。

```
# main.tf

provider "random" {
}

resource "random_shuffle" "zone_id" {
  input        = keys(local.default_zone)
  result_count = 1
}

resource "random_shuffle" "vswitch_id" {
  input        = local.default_zone[random_shuffle.zone_id.result[0]]
  result_count = 1
}

resource "alicloud_db_instance" "instance" {
  vswitch_id    = random_shuffle.vswitch_id.result[0]
  # ...省略部分无关参数...
}
```

```
# variables.tf

locals {
  default_zone = {
    cn-hangzhou-a = ["vsw-xxxxxxxxx001", "vsw-xxxxxxxxx002"]
    cn-hangzhou-b = ["vsw-xxxxxxxxx003", "vsw-xxxxxxxxx004"]
    cn-hangzhou-c = ["vsw-xxxxxxxxx005", "vsw-xxxxxxxxx006"]
  }
}
```


## Generate File

场景描述：需要把一些变量以 json 格式保存在本地文件中。生成文件的同时，还可以指定文件的权限。

这里使用了官方提供的 `local` Provider，[文档](https://registry.terraform.io/providers/hashicorp/local/latest/docs)。

```
# main.tf

resource "local_file" "file" {
  sensitive_content = jsonencode({
    "key_1" : local.value_1,
    "Key_2" : {
      "address" : aws_db_instance.rds.address,
      "username" : aws_db_instance.rds.username,
    }
  })
  filename             = "./example.json"
  file_permission      = "0700"
  directory_permission = "0700"
}
```

## Loop

场景描述：为 AWS ElasticCache 集群的每一个节点创建 CloudWatch Alarm，使用集群的节点数量来控制循环。

```
# main.tf

resource "aws_cloudwatch_metric_alarm" "cpu_utilization" {
  count      = aws_elasticache_replication_group.replication_group.number_cache_clusters

  alarm_name = aws-redis-${aws_elasticache_replication_group.replication_group.replication_group_id}-00${count.index + 1}"
  # ...省略部分无关参数...
}
```

## If Statement

场景描述：为生产环境的资源创建阿里云CloudMonitor Alarm，其他环境的资源则不需要。  

Terraform 本身没有提供 if 语句，这里使用`count`循环指令来控制，`count=1`即`if true`, `count=0`即`if false`。

```
# main.tf

resource "alicloud_cms_alarm" "cpu_usage" {
  count   = var.env == "prod" ? 1 : 0

  project = "acs_rds_dashboard"
  name    = "CPU使用率"
  metric  = "CpuUsage"
  # ...省略部分无关参数...
}
```


## Built-in Function Example

场景描述：创建 AWS DB Instance 时，只有生产环境且高配置实例开启 performance_insight，small及micro规格实例不开启。  

这一语句中使用`split`分隔实例规格字符串，使用`reverse`翻转列表配合`[0]`取最末元素，使用`&&`进行逻辑与运算，使用三元表达式计算最终结果。整个表达式写起来很有python的感觉。  

Terraform 提供了很多内置函数，[文档](https://www.terraform.io/docs/configuration/functions.html)。

```
# main.tf

resource "aws_db_instance" "instance" {
  performance_insights_enabled = (var.env == "prod" && (reverse(split(".", var.instance_class))[0] != "small") && (reverse(split(".", var.instance_class))[0] != "micro")) ? true : false
  # ...省略部分无关参数...
}
```

```
# variables.tf

variable "env" {
  description = "environment"
}

variable "instance_class" {
  default     = ""
  description = "RDS Instance class"
}
```

## Reference

- **Terraform 官方文档** https://www.terraform.io/docs/configuration/index.html
- **The Gruntwork Blog** https://blog.gruntwork.io/terraform-tips-tricks-loops-if-statements-and-gotchas-f739bbae55f9

