---
title: CloudWatch Events配合AWS Lambda使用初体验
date: 2018-09-12 18:41:09
permalink: first-try-aws-lambda-with-cloudwatch-events/
tags:
- AWS
- CloudWatch
- AWS Lambda
categories:
- DevOps
---

## 背景
按照团队的标准规范，在 AWS 上新建的 EC2 实例都要添加 Name, Region, App, Env, Layer, Group 等标签。其中 Name 标签的格式是：`{Region}-{App}-{Layer}-{Env}-{Group}-{IP后两段}`，例如：`vg-store-web-release-a-12-34`。
在 Auto Scaling Group 的配置上，我设置了为新扩容出的实例自动添加 Region, App, Env, Layer, Group 这些标签，但因为 Name 标签中包含 IP 后缀，因此没办法预先设置在 Auto Scaling Group 上，只能在实例创建出来分配到私有 IP 后手动的添加 Name 标签。

为了解决这个问题，我最先想到的方案是：使用脚本调用 API 定时扫描没有 Name 标签的实例，然后获取实例的 PrivateIP 和其他标签拼装出 Name 添加到实例上。
我很快就编写出 Demo，经过测试，可以实现需求，但总觉得不够优雅：
1. 需要定时调用脚本，间隔时间过长就不能及时发现新扩容的实例，间隔过短则显得有些浪费计算资源（事实上，在我们的业务场景中发生扩容的次数很少，间隔以天为单位设置都有些多余）。
2. 需要部署维护脚本及crontab。
3. 需要找一台服务器运行脚本。

后来我想到了 Auto Scaling Group 有 LifeCycle Hook，又从它的设置界面了解到了 CloudWatch Events 这一服务。
最后通过 CloudWatch Events 和 AWS Lambda 就可以很好的实现我的需求！
<!--more-->

## 方案总览

在 AWS Lambda 中编写 Python 脚本，响应 CloudWatch Events 发送的 Auto Scaling 的 `EC2 Instance Launch Successful`事件。Python 脚本使用 boto3 对实例进行查询及创建标签等操作。
如图所示：
![solution.png](https://blog-1252856176.file.myqcloud.com/post/first-try-aws-lambda-with-cloudwatch-events/solution.png)

## 逐步配置

### AWS Lambda

#### 添加运行函数角色

在 AWS Lambda 中，运行函数需要一个 IAM Role，就好像一个 IAM User 调用 API 一样。AWS Lambda 可以通过 IAM Role 来判断是否有权限执行代码中的操作。
在 IAM 中可以创建角色并附加所需的权限策略。

![iam-role-settings.png](https://blog-1252856176.file.myqcloud.com/post/first-try-aws-lambda-with-cloudwatch-events/iam-role-settings.png)

#### 创建函数

创建一个 AWS Lambda 函数，指定函数名称，选择运行语言及上一步创建的用于运行函数的角色。

![add-aws-lambda-function.png](https://blog-1252856176.file.myqcloud.com/post/first-try-aws-lambda-with-cloudwatch-events/add-aws-lambda-function.png)

之后就可以看到如下的可视化Designer，中间的是我们的**函数**，**函数**右侧连接了所选的函数运行角色可操作的 AWS 资源。**函数**左侧还没有连接任何东西，可以从页面左侧的触发器列表中选择 CloudWatch Events。

![aws-lambda-settings.png](https://blog-1252856176.file.myqcloud.com/post/first-try-aws-lambda-with-cloudwatch-events/aws-lambda-settings.png)

从左侧的触发器列表中选择 CloudWatch Events 后，在下方配置详细的触发规则。规则类型设置为事件模式，服务选择 Auto Scaling，事件类型选择 EC2 Instance Launch Successful。

![cloudwatch-events-settings.png](https://blog-1252856176.file.myqcloud.com/post/first-try-aws-lambda-with-cloudwatch-events/cloudwatch-events-settings.png)

上面的步骤是在 AWS Lambda 创建 CloudWatch Events事件。反过来在 CloudWatch 中定义事件源再关联 Lambda 也是可以的。如下图所示：
可以注意一下页面底部的**显示示例事件**中的内容，这是一段包含了此事件详细信息的json串，会作为参数传递给 AWS Lambda。
![cloudwatch-events-settings-2.png](https://blog-1252856176.file.myqcloud.com/post/first-try-aws-lambda-with-cloudwatch-events/cloudwatch-events-settings-2.png)

设置好事件源后，就可以编写 Lambda 函数的代码了。点击 Designer 中的 Lambda 实体。在下方可以看到代码编辑器。在这里可以添加 CloudWatch Events 触发后需要运行的脚本。
脚本自动接收 event, context 两个参数。event 参数包含了触发事件的详细信息，context 参数包含了运行环境的上下文信息。通过 event 参数我们可以获取到 Auto Scaling 新扩容的实例的 InstanceId，然后就可以用 boto3 来做各种操作了。
下面是通过相关标签及私有 IP 设置 Name 标签的代码：
```Python
import json
import boto3


def get_tag_value(instance, key):
    _ = [tag['Value'] for tag in instance['Tags'] if tag['Key'] == key]
    return _[0] if _ else None


def set_name_by_tags(client, instance_id):
    try:
        result = client.describe_instances(InstanceIds=[instance_id])
        instance = result['Reservations'][0]['Instances'][0]

        private_ip = instance['PrivateIpAddress']
        region = get_tag_value(instance, 'Region')
        app = get_tag_value(instance, 'App')
        layer = get_tag_value(instance, 'Layer')
        env = get_tag_value(instance, 'Env')
        group = get_tag_value(instance, 'Group')

        if not (private_ip and region and app and layer and env and group):
            # some tag is empty, can not generate name
            return False, '[Error] some tag is empty'

        ip_suffix = '-'.join(private_ip.split('.')[-2:])
        new_name = f'{region}-{app}-{layer}-{env}-{group}-{ip_suffix}'
        old_name = get_tag_value(instance, 'Name')
        if old_name:
            if old_name == new_name:
                # already have name, nothing need to do
                return True, 'Alreay have the right name'
            else:
                # remove old name
                client.delete_tags(Resources=[instance_id], Tags=[{'Key': 'Name', 'Value': old_name}])
                print(f'remove old Name: {old_name}')

        # set new name
        client.create_tags(Resources=[instance_id], Tags=[{'Key': 'Name', 'Value': new_name}])
        return True, f'Created Name: {new_name}'
    except Exception as e:
        return False, f'[Error] {str(e)}'


def lambda_handler(event, context):
    region = event['region']
    instance_id = event['detail']['EC2InstanceId']
    client = boto3.client('ec2', region_name=region)

    print(f'Ready to set Name for instance: {instance_id}')
    result, msg = set_name_by_tags(client, instance_id)
    print(msg)

    return {
        "statusCode": 200,
        "body": {'result': 'OK' if result else 'Error', 'msg': msg}
    }

```

在页面的右上方可以配置一个测试事件，然后点击测试按钮就可以看到脚本的运行结果了。测试通过后点击保存就完成了。

## 完成

现在 AWS Auto Scaling 扩展出的新实例后，都会触发 CloudWatch Events 向 AWS Lambda 发送一个事件调用 Lambda 函数，函数中的 Python 代码会为新实例添加一个 Name 标签。


## 总结

此方案相比我最初的方案，有以下几个优点：
1. 通过事件触发脚本，不需要定时扫描，及时有效。
2. 做到了无服务器部署，避免了一些维护工作。
3. 可灵活配置，AWS Lambda 可复用。


又一次感受到了 AWS 的强大，完整的生态环境提供一整套的服务支持，只有想不到，没有做不到。
这一次尝试也给我了很多启发，后续可以好好思考如何利用好这些功能，在工作生活中实现更多的自动化。
