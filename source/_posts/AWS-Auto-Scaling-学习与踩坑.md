---
title: AWS Auto Scaling 学习与踩坑
permalink: aws-auto-scaling-study/
date: 2018-07-02 10:04:51
tags:
- AWS
- AutoScaling
categories:
- DevOps
---

最近工作不太顺，总踩坑，当然这是客观原因。主观因素呢，就归结于自己经验不足。

这又刚被 AWS Auto Scaling 给坑了，于是学习总结了一番，记录下来，主要是关于 Auto Scaling 的详细介绍和容易犯错踩坑的地方。方便以后自己查询，也希望能帮到看到这篇文章的人。
<!--more-->


## 什么是 Auto Scaling Group

> [Amazon EC2 Auto Scaling 用户指南](https://docs.aws.amazon.com/zh_cn/autoscaling/ec2/userguide/what-is-amazon-ec2-auto-scaling.html)
>
> Amazon EC2 Auto Scaling 帮助您确保您具有适量的 Amazon EC2 实例可用于处理您的应用程序负载。您可创建 EC2 实例的集合，称为 Auto Scaling 组 。您可以指定每个 Auto Scaling 组中最少的实例数量，Auto Scaling 会确保您的组中的实例永远不会低于这个数量。您可以指定每个 Auto Scaling 组中最大的实例数量，Auto Scaling 会确保您的组中的实例永远不会高于这个数量。如果您在创建组的时候或在创建组之后的任何时候指定了所需容量，Auto Scaling 会确保您的组一直具有此数量的实例。如果您指定了扩展策略，则 Auto Scaling 可以在您的应用程序的需求增加或降低时启动或终止实例。
>
> 例如，以下 Auto Scaling 组的最小容量为 1 个实例，所需容量为 2 个实例，最大容量为和 4 个实例。您制定的扩展策略是按照您指定的条件，在最大最小实例数范围内调整实例的数量。
>
> ![aws-auto-scaling-example.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/aws-auto-scaling-example.png)


## 创建 Auto Scaling Group

创建 Auto Scaling Group 有三种方式：
1. 使用 **EC2实例** 创建
2. 使用 **启动配置** 创建
3. 使用 **Launch Templates(启动模板)** 创建

### 使用 EC2实例 创建

使用 **EC2实例** 创建 Auto Scaling Group需要满足以下条件：
1. 实例位于您要创建 Auto Scaling 组的可用区中。
2. 实例不是其他 Auto Scaling 组的成员。
3. 实例处于 running 状态。
4. 用于启动实例的 AMI 必须仍然存在。

#### 创建步骤

1. 在 EC2 列表找到用于创建 Auto Scaling Group 的实例，操作 -> 实例设置 -> 附加到 Auto Scaling 组。
![create-asg-by-ec2-1.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-ec2-1.png)

2. 在弹出的对话框中选择附加到“**新的 Auto Scaling 组**”，设置一个组名称。
![create-asg-by-ec2-2.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-ec2-2.png)

3. 创建成功，在 Auto Scaling Group 列表中查看创建好的组，检查配置（后面详细介绍）。
![create-asg-by-ec2-3.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-ec2-3.png)

4. 在 Auto Scaling Group 属性的*实例*页签可以看到前面选择的 EC2 实例。最好为此实例设置**缩减保护**。
![create-asg-by-ec2-4.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-ec2-4.png)

#### 注意事项

1. 如果该 EC2 实例已经关联在**一个** ELB 上，则由此创建的 Auto Scaling Group 也会关联到相同的 ELB。
2. 如果该 EC2 实例已经关联在**一个以上（不包含一个）** ELB上，则由此创建的 Auto Scaling Group 不会关联任何的 ELB。


### 使用 启动配置 创建

1. 创建新的**启动配置**（或选择已有的启动配置）
![create-asg-by-launchconf-1.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-launchconf-1.png)

    1. 选择 **AMI映像**
    ![create-asg-by-launchconf-1.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-launchconf-2.png)

    2. 选择配置合适的**实例类型**
    ![create-asg-by-launchconf-2.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-launchconf-3.png)

    3. 设置启动配置的名称
    ![create-asg-by-launchconf-3.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-launchconf-4.png)

    4. 添加存储
    ![create-asg-by-launchconf-4.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-launchconf-5.png)

    5. 配置安全组
2. 创建 **Auto Scaling 组**
    1. 设置组名、组大小、网络及可用区
    ![create-asg-by-launchconf-4.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-launchconf-6.png)

    2. 设置扩展策略
    ![create-asg-by-launchconf-4.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/create-asg-by-launchconf-7.png)

    3. 配置通知
    4. 配置标签

### 使用 Launch Templates(启动模板) 创建

暂未使用，待研究。


## Auto Scaling Group 配置详解

点击创建好的 Auto Scaling Group 可以在下方的属性面板中查看它的详细配置，一个有 9 个页签，下面详细说明。

### 详细信息

![auto-scaling-group-conf-detail-1.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/auto-scaling-group-conf-detail-1.png)

1. 启动配置
    可以选择使用**启动配置**还是**启动模板**来在自动扩容时启动新的实例。两者作用相似，都可以设置自动扩容新实例所需的 **AMI、实例类型、存储、安全组**等。
    此配置还可以随时更改。

2. Classic 负载均衡器 和 目标组
    前者用于设置 ASG 与 Classic ELB(s) 关联，后者用于设置 ASG 与 Application Load Balancer(s) 或 Network Load Balancer(s) 的目标组（Target Group(s))关联。
    将 ASG 与 ELB 关联后，ASG 组内的实例都将自动的挂载到 ELB 上，新增或分离的实例也将同步在 ELB 上挂载或摘掉。
    每个 ASG 最多可以关联 10 个 ELB，可关联目标组的数量未测试。

3. 所需容量、最大、最小
    最大、最小指定了 ASG 在自动扩容或缩减时不能超过的上下范围。所需容量必须在最大、最小值之间，代表当前对实例数量的需求，ASG 会通过自动扩容或缩减将正常的实例数量维持在所需容量。

4. 运行状况检查类型
    运行状况检查类型有两个可选性：**EC2、ELB**。

    EC2： 对 EC2 实例进行状态检查，可以在实例的属性面板查看。关机和重启都会导致状态检查不通过。
    ![ec2-status-check.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/ec2-status-check.png)

    ELB： 使用 ELB 提供的健康检查。

5. 运行状况检查宽限期
    通常，刚刚投入使用的 Auto Scaling 实例需要预热才能通过 Auto Scaling 运行状况检查。Auto Scaling 等运行状况检查宽限期结束才检查实例的运行状况。为了给实例提供足够的预热时间，请确保运行状况检查宽限期包含应用程序的预期启动时间。

6. 终止策略
    终止策略有 5 个可选项：Default、OldestInstance、OldestLaunchConfiguration、NewestInstance、ClosestToNextInstanceHour。

    1. Default(默认终止策略): 默认终止策略可帮助确保您的网络架构均匀分布到多个可用区。
        > 1. 如果多个可用区中都有实例，选择有最多实例并且至少一个实例不受缩小保护的可用区。如果有多个可用区有此数目的实例，则选择使用最旧启动配置的实例所在的可用区。
        >
        > 2. 确定所选可用区中哪些不受保护的实例使用最旧启动配置。如果有一个此类实例，则终止该实例。
        >
        > 3. 如果有多个实例使用最旧启动配置，则确定哪些不受保护的实例最接近下个计费小时。(这将帮助您最大程度地使用您的 EC2 实例并管理 Amazon EC2 使用成本。)如果有一个此类实例，则终止该实例。
        >
        > 4. 如果有多个不受保护的实例最接近下个计费小时，则随机选择其中一个实例。
        >
        > ![termination-policy-default-flowchart-diagram.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/termination-policy-default-flowchart-diagram.png)

    2. OldestInstance
        终止组中最旧的实例。当您将 Auto Scaling 组中的实例升级为新的 EC2 实例类型，可以逐渐将较旧类型的实例替换为较新类型的实例时，此选项十分有用。
    3. OldestLaunchConfiguration
        终止组中最新的实例。如果要测试新的启动配置但不想在生产中保留它时，此策略非常有用。
    4. NewestInstance
        终止采用最旧启动配置的实例。如果要更新某个组并且逐步淘汰先前配置中的实例时，此策略非常有用。
    5. ClosestToNextInstanceHour
        终止最接近下个计费小时的实例。此策略将帮助您最大程度地使用您的实例并管理 Amazon EC2 使用成本。

7. 默认冷却时间
    > [Amazon EC2 Auto Scaling 用户指南：适用于 Amazon EC2 Auto Scaling 的扩展冷却时间
](https://docs.aws.amazon.com/zh_cn/autoscaling/ec2/userguide/Cooldown.html)
    >
    > 在实例投入使用之前，这些实例使用配置脚本安装和配置软件。因此，实例从启动到投入使用大约需要两到三分钟的时间。实际时间取决于诸多因素，如实例大小和是否有启动脚本要完成等。
    >
    > 现在出现了流量高峰，从而导致 CloudWatch 警报触发。该警报触发时，Auto Scaling 组会启动一个实例来帮助处理增加的需求。但是存在一个问题：该实例需要几分钟的时间才能启动。在此期间，CloudWatch 警报可能会继续触发，从而导致 Auto Scaling 组在警报每次出现时都另外启动一个实例。
    >
    > 但是，有了冷却时间，Auto Scaling 组在启动一个实例后，将暂停所有简单扩展策略或手动扩展引起的扩展活动，直至经过了指定时间量。（默认值为 300 秒。）这样，新启动的实例有时间开始处理应用程序流量。冷却时间过后，所有暂停的扩展操作都会恢复。如果 CloudWatch 警报再次触发，则 Auto Scaling 组将启动另一个实例，而冷却时间也会再次生效。不过，如果新增的实例足以将 CPU 使用率降为正常水平，则该组会保持其当前大小。

8. 暂停的进程
    > [Amazon EC2 Auto Scaling 用户指南：暂停和恢复扩展流程](https://docs.aws.amazon.com/zh_cn/autoscaling/ec2/userguide/as-suspend-resume-processes.html)
    >
    > 您可以暂停然后恢复您的 Auto Scaling 组的一个或多个扩展流程。如果需要调查配置问题或与 Web 应用程序相关的其他问题，然后在不触发扩展流程的前提下对应用程序进行更改，则此设置很有用。

    Amazon EC2 Auto Scaling 支持以下扩展流程：

    - Launch
    - Terminate
    - HealthCheck
    - ReplaceUnhealthy
    - AZRebalance
    - AlarmNotification
    - ScheduledActions
    - AddToLoadBalancer

9. 实例保护
    如果已设定好保护免于缩减，新启动的实例在默认情况下将被保护免于缩减。在缩减时 Auto Scaling 将不会选择被保护的实例成为要终止的实例。更改此一选项将不会影响现有实例。

### 活动历史记录

此页签可以看到该 Auto Scaling Group 扩容、缩减、添加、分离等活动记录。

### 扩展策略

扩展策略用于根据需求变化动态的进行扩展或缩减。扩展策略有三种：
1. 目标跟踪扩展策略
2. 简单的扩展策略
3. 带步骤的扩展策略

[Amazon EC2 Auto Scaling 用户指南：适用于 Amazon EC2 Auto Scaling 的目标跟踪扩展策略](https://docs.aws.amazon.com/zh_cn/autoscaling/ec2/userguide/as-scaling-target-tracking.html)

### 实例

*实例*页签可以看到关联在此 ASG 中的 EC2 实例。可以在这里设置**缩减保护**，分离实例等操作。

### 标签

*标签*页签既可以为此 ASG 组添加标签，也可以为此 ASG 组自动扩容创建出的实例打标签。将标签后面的**标记新实例**选项设置为**是**，则此标签将附加在自动扩展出的实例上。

![auto-scaling-group-conf-tag-1.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/auto-scaling-group-conf-tag-1.png)

### 计划的操作

待学习。

### 生命周期挂钩

待学习。


## 实例保护（缩减保护）
缩减保护可以避免在 ASG 组在所需容量减少时终止（terminated）实例。

在 ASG 组属性面板实例标签中可以为组中的实例单独添加或取消缩减保护，也可以设置 ASG 的实例保护属性为“保护免于缩减”，这样 ASG 组启动实例时会自动为实例添加实例保护。

**[注意]** 实例保护并不能针对以下情况保护 Auto Scaling 实例
1. 实例保护不能阻止手动在控制台去终止实例。（这不废话嘛……终止保护可以阻止手动终止实例，不过跟此话题无关。）
    **再注意：** 终止保护可以阻止手动终止实例，但不能阻止 ASG 缩减去终止实例！
2. **实例未通过运行状况检查的情况下的运行状况检查更换。**(这点很重要！就是说，即使设置了实例保护，去关机或重启，都会导致状况检查不通过而被terminated)
3. Spot 实例中断。

## 设为备用
这是个非常有用的功能！合理使用**设为备用**功能在需要对 ASG 组中的实例维护进行关机、重启等操作时可以省去很多麻烦。
在上一节实例保护的介绍中说了，即使设置了实例保护，当实例关机时，仍然会被 ASG 自动 terminated（危险！）。此时就可以使用**设为备用**这个操作。
将实例**设为备用**，会首先将该实例从 ASG 关联的 ELB 中取消注册，然后根据设置减小 ASG 的所需容量（不启动新实例）或启动新的实例（所需容量保持不变）。Amazon EC2 Auto Scaling 不对处于备用状态的实例执行运行状况检查。当实例处于备用状态时，其运行状况将反映您将实例置于备用状态之前，实例具有的状态，直到将实例恢复为可用。

![set-asg-instance-standby.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/set-asg-instance-standby.png)

因为此时 Auto Scaling 不会对备用实例运行状况检查，所以可以进行关机维护等操作而不必担心被自动 terminated。
维护完成后，将实例重新**设为可用**，ASG 的所需容量会自动增加，实例也会重新注册到 ASG 关联的 ELB 上。

**注意：**
1. 在将实例从备用状态**设为可用**前，请务必确定该实例已经正常运行可以通过状况检查，否则在**设为可用**后会被立刻 terminated （即使你设置了实例保护和终止保护，都没用）！
2. 如果 ASG 设置了多个可用区，**设为备用**后导致各可用区中实例数量不一致，Auto Scaling 将会重新平衡可用区，即 terminated 实例较多的可用区中的一台，在实例较少的可用区中新启动一台。要避免自动平衡可用区，可以在 ASG 中设置暂停进程 **AZRebalance**。

## 常用操作总结

### 1. 重启、关机 ASG 中的实例
    
为避免 ASG 将实例自动 terminated，需按以下步骤执行：
1. 在 ASG 属性面板*实例*页签将要维护的实例**设为备用**。（此操作会将实例从 ASG 关联的 ELB 中取消注册，如要求不影响业务，请确保仍有其他正常运行的实例）
2. 进行重启、关机等维护操作。
3. 在**设为可用**前，确保实例已启动，运行正常。
4. 将实例**设为启用**。（一定要确保检查步骤3，否则会导致**设为启用**后被立即 terminated）

### 2. 删除 Auto Scaling Group

在删除 Auto Scaling 组时，其所需值、最小值和最大值均会被设置为 0。因此 ASG 中的实例将会被 terminated。  
如仅需删除 Auto Scaling Group 而保持原有实例不变，需按一下步骤执行：
1. 在 ASG 属性面板*实例*页签将实例逐一分离。（分离操作会导致实例从 ASG 关联的 ELB 中取消注册，如要求不影响业务，需**逐一**分离，挂载 ELB）
2. 确保 ASG 中已没有实例，分离出去的实例已经重新注册到原有的 ELB 上。
3. 删除 ASG（若不需要备份，也请一并删除启动配置）。

### 3. 附加实例到 ASG 中

要附加到 ASG 中的实例需满足以下条件：
1. 实例处于 running 状态。
2. 用于启动实例的 AMI 必须仍然存在。
3. 实例不是其他 Auto Scaling 组的成员。
4. 实例与 Auto Scaling 组位于同一可用区。
5. 如果 Auto Scaling 组具有附加的传统负载均衡器，则实例和该负载均衡器必须都位于 EC2- 或同一 VPC 中。如果 Auto Scaling 组具有附加的目标组，则实例和负载均衡器必须都位于同一 VPC 中。


为避免可能出现的危险操作，在附加实例到 ASG 组中，需按一下步骤执行：
1. 检查 ASG 中的配置，是否关联了 ELB，要添加的实例是否确实要注册到这些 ELB
2. 检查 ASG 中的配置，ASG 的状况检查类型是 EC2 还是 ELB，要添加的实例是否能通过状况检查。
3. 检查 ASG 中的配置，新附加的实例是否添加了**实例保护**，根据场景判断是否需要添加**实例保护**。

### 4. 从 ASG 中分离实例
    
如果不希望使用 ASG 管理实例了，可以在 ASG 的属性面板*实例*页签将实例从 ASG 中分离。分离时，若勾选“将一个新实例添加到 Auto Scaling 组以均衡负载”，ASG 将保持所需容量不变，分离指定实例，然后新启动一台实例来代替分离出去的实例。若取消勾选此项，则 ASG 的所需容量将会自动减一（若减一后小于最小值，将不能分离）。

![detach-instance-from-asg.png](https://blog-1252856176.file.myqcloud.com/post/aws-auto-scaling-study/detach-instance-from-asg.png)

**注意：**
1. 分离出的实例将会自动从 ASG 关联的 ELB 上取消注册！
2. 如果 ASG 设置了多个可用区，分离后导致各可用区中实例数量不一致，Auto Scaling 将会重新平衡可用区，即 terminated 实例较多的可用区中的一台，在实例较少的可用区中新启动一台。要避免自动平衡可用区，可以在 ASG 中设置暂停进程**AZRebalance**。

### 5. 两种组织 ASG 实例的方式对比
    
假设系统日常需要 2 台实例支撑业务运行，高峰是需要 4 台实例。有如下两种方式设置 Auto Scaling Group。

**方式 1：**

2 台实例不使用 ASG 管理，用于支撑日常业务。设置 ASG 最小 0，最大 2，所需 2，用于支撑高峰时业务。

优点：
- 配置简单，适用于对 ASG 不熟悉的同学使用。对原有系统架构添加 Auto Scaling 功能时不影响原系统架构。
- 2 台 ASG 组外的实例不受 ASG 配置的影响，可以避免因 ASG 配置错误造成意外的实例 terminated 或在 ELB 上取消注册。

缺点：
- 无法使用目标跟踪的扩展策略。扩展策略只能监控 ASG 组中的实例。
- 不能统一的修改 ASG 组内外实例的 ELB。

**方式 2：**

实例全部附加到 ASG 管理，日常用的 2 台实例一定要设置**实例保护**。设置 ASG 最小 2， 最大 4，所需 2，配置动态扩展策略。

优点：
- 统一管理，对 ASG 添加 ELB 等设置可以批量的应用在各实例上。
- 可以应用各种动态扩展策略。

缺点：
- 需要熟知 ASG 的各项设置与功能特性。否则容易造成实例意外 terminated 或在 ELB 上取消注册，导致影响线上业务。（这个其实不能算是缺点。）


## 心得体会

1. 线上操作一定要非常小心！想清前因后果，不做不确定的操作，不要想当然。
2. 熟读文档很重要！AWS 的文档写的很棒很细致，不过本地化还不够，有些翻译有些别扭不好理解。也因此有了这篇文章。
3. AWS EC2 Auto Scaling 真的功能很多，很强大！
4. 凡事都有两面。功能强大意味着使用复杂！不熟悉的话真的有很多坑！血泪的教训！
5. 又攒经验了！
