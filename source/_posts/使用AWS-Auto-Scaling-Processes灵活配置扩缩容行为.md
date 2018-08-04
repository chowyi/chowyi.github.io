---
title: 使用AWS Auto Scaling Processes灵活配置扩缩容行为
date: 2018-07-31 18:09:55
tags:
- AWS
- AutoScaling
categories:
- DevOps
---

## 背景

近期在工作中大量用到了 AWS Auto Scaling，经过多次试验，学习文档并与AWS的技术支持沟通，在我对 Auto Scaling 各项功能不断了解的同时，也深感 AWS 的强大。遇到的种种场景、需求，AWS 似乎早就已经替你考虑好了。
今天我遇到的问题是，在对线上服务器进行变更发版等操作时，要求不能发生自动扩缩容，否则可能导致意外的错误（扩容时，新机器使用的是老镜像。缩减时，发版操作将会中断）。要实现这一点，我想到了很多方案。

<!--more-->

## 解决方案

1. 方案一（不推荐）
    在发版构建前，使用脚本调用 AWS API 删除扩展策略（Scaling Policies）。在发版完成之后，再调用 AWS API 重新添加扩展策略。
    优点：简单粗暴？？？(╯‵□′)╯︵┻━┻
    缺点：扩展策略需要配置在脚本中，修改麻烦。在 AWS Console 上对扩展策略的修改，将在下一次发版后被重置。

2. 方案二（不推荐）
    在发版构建前，使用脚本调用 AWS API 暂停扩展流程（Suspend Scaling Processes）：**Terminated、Launch、ReplaceUnhealthy**。在发版完成后，再调用 AWS API 恢复这些流程。
    优点：不用在脚本中配置扩展策略，但也有类似的缺点。
    缺点：在 AWS Console 上对暂停扩展流程的配置，将在下一次发版后被重置。

3. 方案三
    与方案二类似，是仔细看了 AWS 文档后确定的。通过暂停扩展流程：**AlarmNotification** 即可实现暂停自动扩缩容动作。同样通过 AWS API 在发版前后停用及恢复此流程。
    优点：仅需要修改这一个扩展流程，且该流程不会在平时配置，影响极小。
    缺点：暂未想到，目前自用的方案。

## 扩展流程（Scaling Processes）详解

Amazon EC2 Auto Scaling 目前支持 8 中扩展流程（Scaling Processes）：
以下摘自 [Amazon EC2 Auto Scaling 用户指南](https://docs.aws.amazon.com/zh_cn/autoscaling/ec2/userguide/as-suspend-resume-processes.html)

- Launch
    将新的 EC2 实例添加到组，从而增加组的容量。
    **警告**
    如果您暂停 Launch，这会中断其他流程。例如，如果暂停 Launch 流程，则无法使处于备用状态的实例恢复运行，因为组无法扩展。

- Terminate
    从组中删除 EC2 实例，从而减少组的容量。
    **警告**
    如果您暂停 Terminate，这会中断其他流程。

- HealthCheck
    检查实例的运行状况。如果 Amazon EC2 或 Elastic Load Balancing 通知 Amazon EC2 Auto Scaling 实例运行状况不佳，Amazon EC2 Auto Scaling 会将该实例标记为运行状况不佳。此流程可覆盖您手动设置的实例运行状况状态。

- ReplaceUnhealthy
    终止被标记为运行状况不佳的实例，然后创建新实例将其替换。此流程与 HealthCheck 流程结合使用，并使用 Terminate 和 Launch 流程。

- AZRebalance
    在区域内的可用区之间均衡组中 EC2 实例的数量。如果从 Auto Scaling 组中删除可用区，或可用区运行状况不佳或无法使用，扩展流程会在终止运行状况不佳或无法使用的实例前，在不受影响的可用区中启动新实例。当运行状况不佳的可用区恢复正常状态时，扩展流程自动在组的可用区中重新均匀分布实例。有关更多信息，请参阅 再平衡活动。

    如果您暂停 AZRebalance 并且发生了扩大或缩小事件，扩展流程仍会尝试均衡可用区。例如，在扩展期间，它会在实例最少的可用区中启动实例。

    如果您暂停 Launch 流程，AZRebalance 不会启动新实例，也不会终止现有实例。这是因为 AZRebalance 只会在启动替换实例后才终止实例。如果您暂停 Terminate 流程，Auto Scaling 组容量可以增加到超出最大大小百分之十，因为在重新均衡活动期间允许临时发生这种情况。如果扩展流程不能终止实例，Auto Scaling 组可以保持超出其最大大小，直到您恢复 Terminate 流程。

- AlarmNotification
    接受来自与组关联的 CloudWatch 警报的通知。
    如果您暂停 AlarmNotification，Amazon EC2 Auto Scaling 不会自动执行会被警报触发的策略。如果您暂停 Launch 或 Terminate，将无法分别执行扩大或缩小策略。

- ScheduledActions
    执行您创建的计划操作。
    如果您暂停 Launch 或 Terminate，涉及启动或终止实例的计划操作会受到影响。

- AddToLoadBalancer
    在实例启动时，将其添加到附加的负载均衡器或目标组。
    如果您暂停 AddToLoadBalancer，Amazon EC2 Auto Scaling 会启动实例，但不会将其添加到负载均衡器或目标组。如果您恢复 AddToLoadBalancer 流程，该流程也会在启动实例时将其添加到负载均衡器或目标组。不过，它不会添加在此流程暂停时启动的实例。您必须手动注册这些实例。


## 总结

通过灵活地组合配置这些扩展流程，可以实现各种不同的需求，但使用时也要做到心中有数，如上所说，暂停 Launch 流程，会影响到 ReplaceUnhealthy、AZRebalance、ScheduledActions等多个流程。
灵活强大的功能就像一把双刃剑，使用得当，事半功倍，但若不能熟练掌握，就很容易给自己埋下隐患。


