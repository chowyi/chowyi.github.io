---
title: AWS Lambda常见问题
date: 2018-09-17 18:25:45
permalink: aws-lambda-faq
tags:
- AWS
- AWS Lambda
categories:
- DevOps
---

前两天对 AWS Lambda 做了一些学习，发现正好有实际需求用的上。于是业务需求又反推学习，现在我对 AWS Lambda 有了更多的了解。下面以问答形式记录一些我遇到的问题和解决方案。

<!--more-->

## 如何测试 AWS Lambda

代码编写完成之后，我们总是需要进行测试，在 AWS Lambda 中也不例外。AWS Lambda 中的函数由事件驱动，在没有真实事件发生时，我们如何简单快速地进行测试呢？

### 第一步：获取事件格式

AWS Lambda 函数被触发时接收两个参数：`event` 和 `context`。前者是描述事件详细内容的 json 串，后者包含运行时环境的上下文信息。
要对 AWS Lambda 函数进行测试，我们需要构造一个与真实环境 `event` 结构相同的 json 串。CloudWatch Events 中事件的 json 格式可以在其规则的配置页中看到。

![cloudwatch-events-json-example.png](https://blog-1252856176.file.myqcloud.com/post/aws-lambda-faq/cloudwatch-events-json-example.png)

其实[文档](https://docs.aws.amazon.com/zh_cn/lambda/latest/dg/eventsources.html)这里也有写。

### 第二步：配置测试事件

在 AWS Lambda 函数配置的右上方选择**配置测试事件**打开配置窗口。填入上一步中获取的事件示例 json 串。

![aws-lambda-set-test-event.png](https://blog-1252856176.file.myqcloud.com/post/aws-lambda-faq/aws-lambda-set-test-event.png)

### 第三步：发起测试事件

点击**测试**按钮，模拟事件发送，查看程序输出。


## 如何配置第三方依赖库

AWS Lambda 的运行时环境内置各语言相应的 AWS SDK，例如 Python 运行时环境自带 boto3。但如何使用第三方依赖库呢？

遗憾的是 AWS Lambda 没有提供好的方式来解决这个问题。只能用户自己下载好程序的各依赖库源码，然后打包一起上传。上传 zip 包，AWS 会自己解压。

![aws-lambda-upload-package.png](https://blog-1252856176.file.myqcloud.com/post/aws-lambda-faq/aws-lambda-upload-package.png)


## 运行时环境是否有固定的出口IP
AWS Lambda 运行时环境本身没有固定IP，但可以将其放在 VPC 中运行。

我的一个需求是在 AWS Lambda 中调用另一个系统的 API，而此系统需要添加调用方的 IP 到白名单才能被访问。因此就要求 AWS Lambda 的运行时环境需要有固定 IP。
通过查询文档并与 AWS 的技术支持沟通，AWS Lambda 运行时环境本身没有固定IP，但可以将其放在 VPC 中运行。配置好 VPC 后，此函数就可以通过 VPC 的公网 NAT 出口访问外部了，也就有了固定的出口IP。

需要注意：为 AWS Lambda 函数配置 VPC，需要该函数的执行角色拥有 **AWSLambdaVPCAccessExecutionRole** 策略。

![aws-lambda-vpc-config.png](https://blog-1252856176.file.myqcloud.com/post/aws-lambda-faq/aws-lambda-vpc-config.png)

![aws-lambda-vpc-role.png](https://blog-1252856176.file.myqcloud.com/post/aws-lambda-faq/aws-lambda-vpc-role.png)

## 环境变量的使用

在 AWS Lambda 函数代码编辑器的下方可以看到环境变量表单，在这里可以添加一些环境变量以在不同的情形下复用代码。
这里添加的环境变量同本地机器的环境变量一样，正常的引用就可以了，例如在 Python 中可以这样取值：
```Python
import os

print(os.environ['API_SERVER'])
```

注意：目前为止，中国区的 AWS Lambda 还不支持设置环境变量的功能。

![aws-lambda-env-vars.png](https://blog-1252856176.file.myqcloud.com/post/aws-lambda-faq/aws-lambda-env-vars.png)
