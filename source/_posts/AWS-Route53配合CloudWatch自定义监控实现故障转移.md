---
title: AWS Route53配合CloudWatch自定义监控实现故障转移
permalink: aws-route53-failover-with-cloudwatch
date: 2018-06-25 20:57:46
tags:
- AWS
- Route53
- CloudWatch
categories:
- DevOps
---


## 背景
最近工作中遇到的一个场景：业务系统在国内及国外各部署一套，两套系统有各自的域名。当其中一套发生故障时，要求通过修改域名解析，转移流量到另一套系统进行容灾。
现状是通过脚本进行监控，产生告警后手动修改域名解析实现故障转移，待故障系统恢复后，再将域名解析修改回来。
这样的方式存在的几个明显的问题：
1. 出现故障时，人员响应不够及时。
2. 存在多个业务系统，手工操作工作量大，速度慢。
3. 手工操作易出现错误。
4. 故障恢复后，需要再次人工干预改回解析记录。

为解决如上问题，近几天搭建了一套测试环境，通过 AWS CloudWatch 自定义监控 + Route53 Failover 记录实现了故障发生时的自动切换解析，故障恢复后，自动切回解析的效果。
<!--more-->

## 方案设计

### 方案一
使用监控脚本监测到故障发生，调用 AWS Route53 的 api 批量修改解析。

优点：
- 逻辑简单

缺点：
- 需要自己编写批量修改解析记录的脚本，注意异常处理。考虑到此操作风险较高，慎用。
- 后期有脚本的维护成本
- 没有监控记录（其实打日志也是可以记录的，但效果不好）

### 方案二（本文介绍的方法）
1. 自定义监控脚本上报监控指标到 AWS CloudWatch
2. 配置 CloudWatch 的 Alarm（配置故障阈值）
3. 添加 AWS Route53 HealthCheck，监测 CloudWatch Alarm 的状态
4. 添加 AWS Route53 Failover 类型的解析记录，关联上一步创建的 HealthCheck

故障转移的流程：
脚本将监测数据上报至 CloudWatch => 异常数据触发 Alarm => HealthCheck 状态变为 Unhealthy => 域名解析到备用记录

优点：
- 使用整套 AWS 的服务，可靠性较高
- 无需编写脚本，整个方案在 AWS Console 配置实现，易于维护
- CloudWatch 可记录监控数据

缺点：
- 方案中间链路较长，故障发生到故障转移完成有较大延时（监控脚本监控间隔 + CloudWatch Alarm 统计周期 + Route53 HealthCheck监控间隔 + 解析TTL）。


## CloudWatch 自定义监控
[Amazon CloudWatch 文档](https://docs.aws.amazon.com/zh_cn/AmazonCloudWatch/latest/monitoring/PublishMetrics.html)写的很清楚了。
简单来说，CloudWatch 提供接口上报监控数据，有两种方式：既可以调用 AWS CLI 命令，也可以使用 boto3    编写脚本来调用api。

几个概念：
- Metric
    向 CloudWatch 上报的一系列数据称为一个 Metric，即指标。
- Namespace
    Namespace 即命名空间。Metric 需要划分在一个 Namespace 中。例如 AWS 中已经定义的，EC2 的监控指标就在 名为 EC2 的 Namespace 中，RDS 的监控指标就在 名为 RDS 的 Namespace 中。
- Dimension
    Dimension 即维度。在统一命名空间下，又可以把指标按不同维度分类。比如 EC2 Namespace 下，又可以按映像、实例、AutoScaling等来分类。

![metric-list.png](https://blog-1252856176.file.myqcloud.com/post/aws-route53-failover-with-cloudwatch/metric-list.png)

实际通过命令或api上报监控数据时，需要指定 MetricName、Namespace、timestamp、value（指标的值） 以及 unit（指标单位）。

我这里编写python脚本调用ping命令监控网络状况，使用 boto3 将响应时间上报 CloudWatch。使用 crontab 每分钟定时调用脚本。
之后就可以在 CloudWatch 的控制台看到又上报数据绘制成的图表了。图表还可以做很多负责的设置甚至加入数据表达式，功能很强大。

![monitor-lines.png](https://blog-1252856176.file.myqcloud.com/post/aws-route53-failover-with-cloudwatch/monitor-lines.png)

## 配置 CloudWatch Alarm
上一步已经把监控数据收集到了 CloudWatch，现在可以通过对每一项指标（Metric）在一个周期内设置一个阈值创建一个警报（Alarm）。当指标超出阈值时会引起 Alarm 状态的改变。

Alarm 有三种状态：
- OK —— 指标位于规定的阈值范围内
- ALARM —— 指标在规定的阈值范围外
- INSUFFICIENT_DATA —— 警报刚刚开始；指标不可用；或指标数据不足以判断警报状态

Alarm 状态变化时可以触发一些操作，比如：发送短信通知，Auto Scaling等。详见[Amazon CloudWatch警报文档](https://docs.aws.amazon.com/zh_cn/AmazonCloudWatch/latest/monitoring/AlarmThatSendsEmail.html)。

本方案中不需要触发任何操作，只需要使用 Alarm 的状态。

![edit-alarm.png](https://blog-1252856176.file.myqcloud.com/post/aws-route53-failover-with-cloudwatch/edit-alarm.png)


## 配置 Route53 HealthCheck
Route53 Failover 解析需要关联一个 Route53 HealthCheck，当该记录的 HealthCheck 状态为 Unhealthy 时切换到备用解析。
因此，首先要创建一个 Route53 HealthCheck。

Route53 HealthCheck 有三种监控类型：
1. Endpoint （终端（不知道怎么翻译），就是对 IP 或域名的某个端口进行监控）
2. 其他的 HealthCheck （可以对其他多个HealthCheck做一些组合条件）
3. CloudWatch Alarm （可以监控 Alarm 的状态）

这里我就需要选用第三种对 CloudWatch Alarm 的监控，然后关联上一步中创建的 CloudWatch Alarm。

![create-r53-healthcheck](https://blog-1252856176.file.myqcloud.com/post/aws-route53-failover-with-cloudwatch/create-r53-healthcheck.png)

创建完成后回到 Route53 HealthCheck 列表就可以看到 HealthCheck 当前的状态了。

![healthcheck-list.png](https://blog-1252856176.file.myqcloud.com/post/aws-route53-failover-with-cloudwatch/healthcheck-list.png)

## 配置 Route53 Failover 解析
Route53 配置故障转移有两种方案。一种是使用 Failover 解析，功能较为简单。另一种是使用 Traffic Policy 功能，功能十分强大，但每条策略要收取 50$/month 的费用。
这里对比两者及需求分析，我这里只需要简单的使用 Failover 解析就可以了。

1. 首先创建两条除解析值以外一模一样的主备两条解析（这两条解析的 Name 为中间值，可以随意选取，不能是最终的域名）。
2. 创建两条ALIAS别名解析，`Alias Target`分别为第一步中的两条主备解析。`Routing Policy`选择 Failover，`Failover Record Type`分别设置为 Primary 和 Secondary。`Evaluate Target Health`设置为 Yes，`Associate with Health Check`设置为 Yes，分别关联主备链路对应的 Route53 HealthCheck。

配置如下图：

![dns-list.png](https://blog-1252856176.file.myqcloud.com/post/aws-route53-failover-with-cloudwatch/dns-list.png)

![failover-dns.png](https://blog-1252856176.file.myqcloud.com/post/aws-route53-failover-with-cloudwatch/failover-dns.png)

## 测试
1. 按如上配置，正常时域名解析指向`www.baidu.com`，测试成功。
2. 修改主链路监控脚本，向 CloudWatch 上传一些异常数据。
3. 在 CloudWatch 控制台可以看到监控曲线异常。
4. CloudWatch Alarm 状态由 ok 变为 alarm。
5. Route53 HealthCheck 状态有 Health 变为 Unhealthy。
6. 再次访问域名，解析已经指向`www.google.com`。故障转移生效。
7. 还原主链路监控脚本，上报正常的监控数据。以上步骤依次恢复正常，域名解析还原。

测试中故障转移完成的时间大概为3-5分钟，监控正常后恢复时间较转移时略长一些。 

影响故障转移时间的因素如下：
- 监控脚本监控间隔
- CloudWatch Alarm 统计周期
- Route53 HealthCheck监控间隔
- DNS TTL

## Route53 Traffic Policy
Traffic Policy 功能非常强大，AWS 提供了一个可视化的编辑器，可以用来配置整个架构链路。
Traffic Policy 同样可以实现 Failover 解析的功能，不过显得有点大材小用，而且每条 policy 需要收取 50$/month。
更详细的内容可以查看[文档：复杂 Amazon Route 53 配置中运行状况检查的工作原理](https://docs.aws.amazon.com/zh_cn/Route53/latest/DeveloperGuide/dns-failover-complex-configs.html)。

![traffic-policy-graph.png](https://blog-1252856176.file.myqcloud.com/post/aws-route53-failover-with-cloudwatch/traffic-policy-graph.png)

## 总结
算上环境搭建，整个方案设计断断续续的研究测试了几天，非常有意思，也幸亏 AWS 的文档写的好，还算顺利。

一开始我是打算用文中提到的方案一去实现的，考虑之后感觉风险比较大，又听说了 Route53 的 Traffic Policy 功能，就去看了一下 Traffic Policy。
把 Traffic Policy 的环境搭起来测试成功了，但是需要收费。正好在查 AWS 文档的时候，看到了简单的方案可以用 Failover 记录去实现，就又花了一下午去测试。整个过程可能多花了一点时间，不过也不能算走冤枉路，都还是很有收获的。

几点感受：
1. 在生产中不要重复造轮子，先去看看有没有成熟的解决方案。
2. AWS 的整套工具真的很全很好用，是一条完整流畅的工具链。
3. 思路清晰很重要，特别涉及到链路复杂时，先梳理好流程可以事半功倍。
4. Zabbix 的监控数据可不可以上报到 CloudWatch 管理呢？嗯？
