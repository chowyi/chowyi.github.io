---
title: 深入了解AWS Auto Scaling的动态扩展——目标跟踪扩展策略
permalink: aws-auto-scaling-target-tracking-scaling-policy 
date: 2018-08-31 17:17:49
tags:
- AWS
- AutoScaling
categories:
- DevOps
---

## 背景

AWS Auto Scaling 有手动扩展（Manual Scaling）、计划的扩展（Scheduled Scaling）及动态扩展（Dynamic Scaling）三种扩展方式。
手动扩展（Manual Scaling）、计划的扩展（Scheduled Scaling）两种方式使用比较简单，本文主要记录我在使用动态扩展（Dynamic Scaling）时遇到的一些疑惑及解答。

动态扩展（Dynamic Scaling）目前支持的扩展策略类型有：

1. 目标跟踪扩展策略（Target tracking scaling）
2. 步进扩展策略（Step scaling）
3. 简单扩展策略（Simple scaling）

详细的介绍可以看官网文档：[适用于 Amazon EC2 Auto Scaling 的动态扩展](https://docs.aws.amazon.com/zh_cn/autoscaling/ec2/userguide/as-scale-based-on-demand.html)

以下内容会记录一些在文档中没有写清，我通过实际测试并与AWS 技术支持沟通得到的结果。
<!--more-->

## 目标跟踪扩展策略的具体机制

### 什么是目标跟踪扩展策略
一句话简述：目标跟踪的扩展策略，可以预定义一个指标的目标值，例如：使 Auto Scaling 组的平均聚合 CPU 利用率保持在 50%，Auto Scaling 可以通过自动扩展或收缩实例数量，将指标保持在目标值附近。


### Question 1：多久可以扩容出实例？

当我们使用 Auto Scaling 时，我们最关心的问题就是：当指标到达目标值时，多长时间会启动新的机器？指标瞬间的波动会引发自动扩缩容吗？

文档中有这么一句话：
> To ensure application availability, the Auto Scaling group scales out proportionally to the metric as fast as it can, but scales in more gradually. 
> 为了确保应用程序可用性，Auto Scaling 组会针对指标尽快按比例向外扩展，但会逐渐向内扩展。

“as fast as it can" 具体有多 fast ？"in more gradually" 又是怎样的 gradually 呢？

当我们设定一个目标跟踪扩展策略时，AWS 会自动地在 CloudWatch Alarms 中新增两条以 "TargetTracking-" 开头的警报。
举例来说，当我们设定策略为 Average CPU Utilization = 50 时，我们会看到两条警报分别是：
1. CPU 利用率 (CPUUtilization) > 50 用于 3 个数据点 (位于 3 分钟 范围内)
2. CPU 利用率 (CPUUtilization) < 35 用于 15 个数据点 (位于 15 分钟 范围内)

即：
1. 警报每分钟检查一次，如果*CPU 利用率 (CPUUtilization)*在过去的3个数据点上都超过 50%，警报就会被触发，随即 Auto Scaling Group就会启动实例。
这就是文档中所说的 "as fast as it can"。
2. 警报每分钟检查一次，如果*CPU 利用率 (CPUUtilization)*在过去的15个数据点上都低于 35%，警报就会被触发，随即 Auto Scaling Group就会缩减实例。
这就是文档中所说的 "in more gradually"。其中 35% 的阈值由策略目标值（50%）的 70% 计算所得。

**注意：**
文档中有这么一段：
> We recommend that you scale on metrics with a 1-minute frequency because that ensures a faster response to utilization changes. Scaling on metrics with a 5-minute frequency can result in slower response time and scaling on stale metric data. By default, Amazon EC2 instances are enabled for basic monitoring, which means metric data for instances is available at 5-minute intervals. You can enable detailed monitoring to get metric data for instances at 1-minute frequency.
> 建议将随指标扩展的频率设置为 1 分钟，这可更快地响应使用率变化。如果将随指标扩展的频率设置为 5 分钟，可能会导致响应时间变慢，并且可能导致系统依据陈旧的指标数据进行扩展。默认情况下，Amazon EC2 实例是针对基本监控启用的，也就是说，实例的指标数据以 5 分钟的间隔提供。您可以启用详细监控，从而以 1 分钟的频率获取实例的指标数据。

上文说的警报每分钟检查一次，需要开启实例的详细监控。默认的基础监控频率为5分钟一次，详细监控频率为1分钟一次。详细监控会收取少量的额外费用。
此外，不仅要为 Auto Scaling Group 中已有的实例开启详细监控，还要注意在定义 Launch Templates / Launch Configuration (启动配置) 时勾选 Monitoring (监控)，这样新增的实例也会启用详细监控。

**结论：**
1. 当开启详细监控时，指标到达预设目标值后，最快3分钟可触发 Auto Scaling 启动新实例。
2. 当开启详细监控时，指标低于预设目标值的 70% 后，至少15分钟后会触发 Auto Scaling 缩减实例。
3. 由前两条结论可知，时间小于3分钟的指标短暂的波动，不会导致触发自动扩容。


### Question 2:扩容是一个实例一个实例的添加吗？

首先，文档中提到，目标跟踪扩展策略仅适用于指标值与 Auto Scaling 组中的实例数成比例地增加或减少的情况，以便能够利用指标数据按比例地扩展或缩减实例数量。
其次，我们要了解目标跟踪扩展策略是如何计算要扩展或缩减的实例数量的。

引用文档中的举例：
> 我们在确定需要添加或删除多少个实例时通过向上或向下取整来保守地进行操作，以免添加的实例数量不足或删除的实例数量过多。例如，假设将 CPU 利用率的目标值设置为 50%，则当 Auto Scaling 组超出目标值时，我们可以确定添加 1.5 个实例可将 CPU 使用率降至 50% 左右。由于不可能添加 1.5 个实例，我们将该值向上取整，添加两个实例。这可能会将 CPU 使用率降至 50% 以下，但可确保应用程序具有充足的支持资源。同样，如果我们确定删除 1.5 个实例可使 CPU 使用率提高到 50% 以上，我们将只删除一个实例。

由此可知，目标跟踪扩展策略会首先按比例计算要达到预设目标值需要添加或缩减的实例数量，保守的进行取整，然后执行添加或缩减实例。
也就是说，目标跟踪的扩展策略会按比例一次扩容出多台机器，而不是一台一台的扩展。这非常符合我们的实际需求。

文档中还提到：
> 对于实例较少的 Auto Scaling 组，组的使用率可能偏离目标值较远。
> 对于实例较多的 Auto Scaling 组，使用率分布在更多的实例上。添加或删除实例会导致目标值与实际指标数据点之间的差距缩小。 

**结论：**
目标跟踪的扩展策略会按比例计算出需要扩容的机器数量，然后一次扩容出来。


