---
title: 使用Route53配置顶级域名跳转的两种方式——Lambda@Edge再显身手
date: 2019-02-14 20:43:40
permalink: two-ways-to-redirect-domain-zone-apex
tags:
- AWS
- AWS Lambda
categories:
- DevOps
---


As we know, 在 RFC 中有明确说明 CNAME 记录不能与其他记录共存，特别是容易与 MX 记录产生冲突。因此，很多域名解析服务提供商都不允许为顶级域名设置 CNAME 解析。那如果我希望访问域名 example.net 跳转到 example.com 该如何做呢？

使用Nginx等后端服务器做跳转这里不考虑，现在讲究 Serverless 嘛。通过查询 AWS 相关文档，我找到了两种方法。  

1. 使用 Route53 提供的 Alias 别名记录配合 S3 静态网站托管功能实现。
2. 使用 Route53 提供的 Alias 别名记录配合 CloudFront 和 Lambda@Edge 实现。

需要注意：

- 两种方法都依赖于 Route53 的别名记录功能，这不是 RFC 标准中提供方法，详细原理下文中说明。其他解析服务商应该也有类似的功能。  
- 方法1使用 S3 静态网站托管功能实现跳转，设置简单，但无法配置 SSL 证书，需要 https 协议的网站不适用于此方法。  
- 为了解决方法1不适用 https 协议的问题，可以使用方法2，在 CloudFront 上配置 SSL 证书，利用 Lambda@Edge 实现跳转。  

下面详细记录我的配置过程。
<!--more-->

## 什么是 Alias 别名记录

引用一段来自[AWS Route53 文档](https://docs.aws.amazon.com/zh_cn/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)中的介绍：

> Amazon Route 53 别名记录 为 DNS 功能提供特定于 Route 53 的扩展。别名记录允许您将流量路由到所选 AWS 资源，如 CloudFront 分配和 Amazon S3 存储桶。它们还允许您将流量从托管区域中的一个记录路由到另一个记录。
> 与 CNAME 记录不同，您可以在 DNS 命名空间的顶端节点（又称为顶级域名）上创建别名记录。例如，如果您注册了 DNS 名称 example.com，则顶级域名就是 example.com。您不能为 example.com 创建 CNAME 记录，但可以为将流量路由到 www.example.com 的 example.com 创建别名记录。 

> 当您使用别名记录将流量路由到 AWS 资源时，Route 53 会自动识别资源中的更改。例如，假设 example.com 的一个别名记录指向位于 lb1-1234.us-east-2.elb.amazonaws.com 上的一个 ELB 负载均衡器。如果负载均衡器的 IP 地址更改，Route 53 将使用新 IP 地址自动开始响应 DNS 查询。

![顶级域名不允许配置CNAME记录](https://blog-1252856176.file.myqcloud.com/post/two-ways-to-redirect-domain-zone-apex/cname-is-not-permitted-at-apex.png)

### 个人理解（如果有误还望指正）

别名记录是 Route53 对 A记录的一层包装，将域名与 AWS 中的资源ID 关联起来。例如，将一个域名通过别名记录解析到 CloudFront Distribution，从外部来看实际上是用 A记录把域名解析到了一个 CloudFront 边缘节点的 IP，在内部 Route53 会自动识别资源中的更改，返回资源实际对应的 IP。通过命令`dig example.com`就可以看到。


## 通过 S3 实现跳转

这个方法我是在 AWS 的文档库中搜索到的：[有没有办法使用 Amazon S3 和 Amazon Route 53 将一个域重定向到其他域？](https://amazonaws-china.com/cn/premiumsupport/knowledge-center/redirect-domain-route-53/)

详情看原文，这里简述步骤：

1. 创建一个与当前域名完全同名的 S3 存储桶。
2. 设置存储桶的Properties(属性)，设置 Static Website Hosting(静态网站托管)，选择重定向请求，目标设置为要跳转到的域名。
3. 为当前域名添加一条 A记录，选择 Alias 别名，设置为上一步中创建的 S3 存储桶。

跟着文档设置完成后，确实实现了顶级域名的跳转。但整个过程中都没有地方让我配置 SSL 证书，也就是说这种方法不支持 https 协议。

虽然这种方法不够完美，但还是给了我启发！我可以利用 Alias别名记录将域名指向 CloudFront 配置 SSL证书，然后利用 Lambda@Edge 拦截 ViewerRequest 实现跳转。


## 通过 Lambda@Edge 实现跳转（CloudFront配置SSL证书）

整个流程大致如下：
![使用Lambda@Edge实现顶级域名跳转](https://blog-1252856176.file.myqcloud.com/post/two-ways-to-redirect-domain-zone-apex/apex-domain-redirect-by-cloudfront-and-lambda.png)

之前的一遍文章[AWS Lambda@Edge小试牛刀——代替Nginx处理请求](/handle-request-like-nginx-by-lambda-at-edge/)详细的说明如何创建 CloudFront Distribution 和 Lambda 函数。同样的方法，现在来实现本次的需求。

1. 先创建一个新的 CloudFront Distribution(分配)，注意配置 Alternate Domane Names(备用域名) 为 example.net 并配置 SSL 证书，源站配置无所谓，因为我们要用 Lambda@Edge 拦截 ViewerRequest 做跳转。
2. 创建一个 Lambda@Edge 函数，将来自 example.net 的请求都 301 重定向到 example.com，下面贴上我简化后的函数代码。
3. 将 Lambda@Edge 函数部署到第一步创建的 CloudFront Distribution 的 ViewerRequest 上。
4. 为 example.net 添加一条 A记录，Alias别名设置为第一步创建的 CloudFront Distribution 的域名。

这样就大功告成了，看起来有点复杂，动手配置一次就会明白啦！

```JavaScript
'use strict';

const siteConfigs = {
    'example.net': {
        'response': {
            'status': '301',
            'statusDescription': 'Moved Permanently',
            'location': 'https://example.com'
        }
    }
};

exports.handler = (event, context, callback) => {
    const request = event.Records[0].cf.request;
    const oldURI = request.uri;
    let host = request.headers.host[0].value;
    
    for(let key in siteConfigs){
        if(key === host){
            const siteConfig = siteConfigs[key];
            if(siteConfig.response){
                const response = {
                    status: siteConfig.response.status,
                    statusDescription: siteConfig.response.statusDescription,
                    headers: {
                        location: [{
                            key: 'Location',
                            value: siteConfig.response.location + oldURI + (request.querystring?('?'+request.querystring):''),
                        }]
                    },
                };
                callback(null, response);
            }
            break;
        }
    }
    callback(null, request);
};

```

## 总结

在这次的工作任务中学到了很多 DNS 的相关知识。AWS 提供了跟多强大的功能，灵活运用会有意想不到的效果。
