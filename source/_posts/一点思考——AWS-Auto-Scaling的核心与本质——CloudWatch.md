---
title: 一点思考——AWS Auto Scaling的核心与本质——CloudWatch
date: 2018-09-04 16:36:24
permalink: thoughts-about-the-core-and-essence-of-aws-auto-scaling
tags:
- AWS
- AutoScaling
categories:
- DevOps
---


在了解了目标跟踪扩展策略（Target Tracking Scaling Policy）的机制之后，突然就对 AWS Auto Scaling 有了新的理解。它最核心的部分其实是 CloudWatch！

想明白了这一点之后，AWS Auto Scaling 就不再那么神奇与神秘了。我甚至可以利用 CloudWatch 和 EC2 的相关 API 自己来实现一套 Auto Scaling！

（当然，这并不能否认 AWS Auto Scaling 的强大，无论是在细节上还是用户体验上它都打磨的非常完美了。）
<!--more-->

## 思考
了解了目标跟踪扩展策略的机制之后，我突然想到，触发扩缩容动作的实际上就是 CloudWatch Alarm 嘛！
当 CloudWatch Alarm 被触发后，调用 EC2 的 API 用指定的镜像或启动模板去创建或者销毁机器，再把新创建的机器挂载到 ELB 上，这样就实现了基本功能。
种种的扩展策略，都是可以通过控制程序逻辑围绕 CloudWatch 的监控指标来实现。
AWS Auto Scaling 实际上是把 AMI，LaunchTemplate，ELB，SNS，CloudWatch等功能巧妙的融合在了一起。猜想一下，也许还用到了 AWS Lambda。

## 核心
CloudWatch 作为 AWS Auto Scaling 的核心，提供了触发自动扩容，缩减的原始事件。
设置扩展策略，实际上就是设置 CloudWatch Alarm。
通过灵活编辑 CloudWatch Alarm，我们完全可以实现出各种各样的扩展策略。

## 本质
按下面列出的流程来说明吧：
1. 新建 Auto Scaling Group
本质：选择启动镜像，选择要关联的 ELB
2. 添加扩展策略
本质：添加 CloudWatch Alarm
3. 达到扩展策略条件
本质：CloudWatch Alarm 发生报警
4. 扩出新实例
本质：使用预设的镜像启动新实例，挂载到预设的ELB上
5. 缩减实例
本质：释放实例
6. 实例保护
本质：对实例做标记，缩减时跳过该实例


## 总结
想通了发现没什么好写的，就这样吧。
