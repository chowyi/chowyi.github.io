---
title: AWS Lambda@Edge小试牛刀——代替Nginx处理请求
date: 2019-01-24 17:59:08
permalink: handle-request-like-nginx-by-lambda-at-edge/
tags:
- AWS
- AWS Lambda
categories:
- DevOps
---

前一段时间，我在工作中用 Lambda 实现了很多功能，比如，对云上资源进行定时巡检，CloudWatch警报转发等等。现在我对 AWS Lambda 类似的传统用法已经非常熟悉了，除此之外，AWS Lambda还有一个强大的功能 —— Lambda@Edge。之前通过学习文档了解了一些，今天正好有实际需求可以拿来练练手。
<!--more-->

## 背景

我们有一些保护性注册的域名需要跳转到网站的首页，比如需要将 `www.example.com.au` 跳转到官网首页 `www.example.com` ，之前我们使用 Nginx 做重定向，配置如下：
```
server {
    listen       80;
    server_name www.example.com.au;
    access_log /var/log/nginx/access_example_com_au.log;
    error_log  /var/log/nginx/error_example_com_au.log warn;

    rewrite ^ http://www.example.com$request_uri? permanent;
}
```

可以看到配置非常简单， Nginx 收到来自 `www.example.com.au` 请求后直接 permanent 301 重定向到 `www.example.com` 。那为这么简单的一个小功能开一台服务器部署 Nginx 显然有些浪费，并且还有服务器宕机的风险和维护成本。

有同学可能会问：为什么不用 DNS 解析直接把 `www.example.com.au` CNAME 到 `www.example.com` 呢？  
这是因为当使用 https 协议访问网站时，CNAME 跳转会有 SSL 证书的问题。

我们现在的解决方式就是使用 AWS Lambda@Edge。



## 什么是 Lambda@Edge

Edge 在这里指的是 CloudFront 的边缘节点，Lambda@Edge 即运行在 CloudFront 边缘节点上的 Lambda 函数。因此，Lambda@Edge 必须配合 CloudFront 一起使用。

在**请求**或**响应**进出 CloudFront 的前后，都可以使用 Lambda@Edge 来拦截**请求**或**响应**，并对**请求**或**响应**做出修改。

![Lambda@Edge示意图](https://docs.aws.amazon.com/zh_cn/lambda/latest/dg/images/cloudfront-events-that-trigger-lambda-functions.png)

如图所示，在 Client——CloudFront——Origin 的整个链路中，Lambda@Edge 一共有四处位置可以拦截并修改**请求**或**响应**，依次是：

1. Viewer request(查看器请求)，在 CloudFront 收到用户请求之后，查找 CloudFront cache 之前。
2. Origin request(源请求)，在 CloudFront cache 未命中之后，转发请求到源之前。
3. Origin response(源响应)，在 CloudFront 收到来自源的响应之后，缓存 CloudFront cache 之前。
4. Viewer response(查看器响应)，在 CloudFront 缓存 cache 之后，返回响应到用户之前。

在 Lambda@Edge 中，我们可以检查或修改 Headers，重写 URL 的路径，甚至直接在 Lambda 中生成新的 request 或 response。基于这些能力，我们可以做到：

- 检查 Cookie，从而重写站点不同版本的 URL 以进行 A/B 测试。
- 根据 User-Agent 标头将不同的对象发送给您的用户，该标头包含有关提交请求的设备的信息。例如，您可以根据用户的设备向用户发送分辨率不同的图像。
- 检查标头或授权令牌，在将请求转发到源之前插入一个相应的标头并允许访问控制。
- 添加、删除和修改标头，然后重写 URL 路径，将用户定向到缓存中的不同对象。
- 生成新的 HTTP 响应，将未经身份验证的用户重定向到登录页面，直接从边缘创建和交付静态网页，或执行其他操作。有关更多信息，请参阅 Amazon CloudFront 开发人员指南 中的使用 Lambda 函数生成对查看器和源请求的 HTTP 响应。

在 AWS 的官方文档中，给出了很多[Lambda@Edge函数示例](https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/lambda-examples.html)。


## 从 Nginx 迁移到 Lambda@Edge

### 配置 CloudFront

Lambda@Edge 函数是运行在 CloudFront 边上节点上的，因此我们必须先创建 CloudFront Distribution(分配)。

对于我的本次需求来说，源站的配置无所谓，所有的请求都通过 Lambda@Edge 直接重定向，不存在回源的情况。我就简单的配置 `*.example.com.au` 回源到 `www.example.com` 就好。唯一要注意的就是需要在 CloudFront 上配置好 `*.example.com.au` 的 SSL 证书。

![配置CLoudFront Distribution](https://blog-1252856176.file.myqcloud.com/post/handle-request-like-nginx-by-lambda-at-edge/create-cloudfront-distribution.png)

### 创建 Lambda 函数

我以 AWS Lambda 提供的蓝图（blueprint）为模板开始创建，在蓝图（blueprint）中搜索关键词**cloudfront**，选择**cloudfront-modify-response-header**。给函数起个名字，然后选择一个运行函数的角色。

![创建Lambda@Edge函数](https://blog-1252856176.file.myqcloud.com/post/handle-request-like-nginx-by-lambda-at-edge/create-lambda-at-edge-by-blueprint.png)

运行函数的角色需要有如下权限用来在 CloudWatch 中记录日志：
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:PutLogEvents"
            ],
            "Resource": [
                "arn:aws:logs:*:*:*"
            ]
        }
    ]
}
```

然后根据实际需求修改函数的代码。我这里只需要将收到的请求全部301重定向即可。代码如下：
```javascript
'use strict';

exports.handler = (event, context, callback) => {
    const request = event.Records[0].cf.request;
    const oldURI = request.uri;

    const response = {
            status: '301',
            statusDescription: 'Moved Permanently',
            headers: {
                location: [{
                    key: 'Location',
                    value: `https://www.example.com${oldURI}` + (request.querystring?('?'+request.querystring):''),
                }]
            },
            
        };
    
    callback(null, response);
};

```

保存函数，接下来就可以部署到 Lambda@Edge 了。

### 部署到 Lambda@Edge

完成 Lambda 函数的编写并保存后，选择【操作】-【部署到Lambda@Edge】。

在这里选择前面创建的CloudFront**分配**（Distribution）。CloudFront 事件选择**查看器请求**（Viewer Request），因为我的需求只要重定向，不需要回源，也用不到 CloudFront cache。点击部署，这样 Lambda 函数就会被部署在 CloudFront 边缘节点上。

![部署到Lambda@Edge](https://blog-1252856176.file.myqcloud.com/post/handle-request-like-nginx-by-lambda-at-edge/deploy-lambda-at-edge.png)

这样，当有来自 `*.example.com.au` 的请求，都会在 CloudFront 上触发 ViewerRequest 事件，按照 Lambda 函数中的逻辑被 301 重定向到 `www.example.com`。

## 小结

在这个场景下：使用 AWS Lambda@Edge 有如下优势：
1. 无服务器部署
    1.1 节省成本，不需要购买服务器。Lambda 函数只在被触发时按秒计费。
    1.2 无需维护，永不宕机。
    1.3 弹性扩容，可从容应对大流量。
2. 计算更靠近用户端，响应迅速。

这次遇到的需求比较简单，而且我之前有使用 AWS Lambda 的相关经验，整个过程还算顺利。接下来我会继续研究 Lambda@Edge 的其他用法，工作中也会有实际的场景。在更深入的学习之后，我会再详细记录 Lambda@Edge 函数的写法和调试方法。



