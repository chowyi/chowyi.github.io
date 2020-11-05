---
title: CloudFront域名劫持的原理与保护
date: 2019-03-01 22:12:19
permalink: the-way-of-cloudfront-domain-hijacking/
tags:
- AWS
- CloudFront
categories:
- DevOps
---


在配置 CloudFront Distribution 时，需要填写的有一项是**Alternate Domain Names (CNAMEs)**，中文是**备用域名 (CNAMEs)**。一直不太清楚为什么要填这一项。

- 在 DNS 上直接把域名解析到 CloudFront Distribution 生成的域名 dxxxxxxxxxxxx.cloudfront.net 上不就可以了吗？  
- 为什么新建 CloudFront Distribution 时填写的 Alternate Domain Names (CNAMEs) 不能与现有的重复？  
- 是填写 subdomain 好，还是填写 wildcard domain 好？  


昨天，在我创建 CloudFront Distribution 时，AWS 提示我填写的 subdomain 已经配置在了一个 Distribution 上，而我的账号上并没有这样的配置。由此我了解到了 CloudFront 域名劫持这一现象。带着上面的疑问我做了些实验，查询了文档并向 AWS Support 了解到了相关原理，在这里记录一下。
<!--more-->

## 我的实验流程

1. 新建一个 CloudFront Distribution，源站设置为`www.baidu.com`，没有配置 Alternate Domain Names (CNAMEs)。
2. 创建一条 DNS 记录，将 `test.example.com` 指向了上面创建的 Distribution。
3. 访问 `test.example.com`，可以看到 CloudFront 的报错信息：The request could not be satisfied.
4. 访问 Distribution 自带的域名，可以成功访问到源站。
    ![cloudfront-request-could-not-be-satisfied](https://blog-1252856176.file.myqcloud.com/post/the-way-of-cloudfront-domain-hijacking/cloudfront-request-could-not-be-satisfied.png)

    **结论**：创建 CloudFront Distribution 时一定要添加 Alternate Domain Names (CNAMEs)，否则 CloudFront 会报错。

5. 现在我再创建一个 Distribution，源站设置为`www.google.com`，配置 Alternate Domain Names (CNAMEs) 为 `test.example.com`。
6. 现在再访问 `test.example.com`，竟然回源到了 `www.google.com`！实现了 CloudFront 域名劫持！
    ![cloudfront-domain-hijacking](https://blog-1252856176.file.myqcloud.com/post/the-way-of-cloudfront-domain-hijacking/cloudfront-domain-hijacking.png)


## 什么是 CloudFront 域名劫持

在上面我的实验中可以看到，`test.example.com`解析到了源站配置为`www.baidu.com`的 Distribution，但这个 Distribution 没有配置 Alternate Domain Names (CNAMEs)。之后我又添加了一个源站为`www.google.com`的 Distribution， Alternate Domain Names (CNAMEs) 配置为`test.example.com`。现在访问`www.example.com`看到的是`www.google.com`，而不是原本期待的`www.baidu.com`。创建后面这一 Distribution 的可以是任何人（不需要是`www.example.com`的Owner），这一过程就叫做 CloudFront 域名劫持。


## CloudFront 域名劫持的原理

从 AWS Support 那里，我了解到 CloudFront 是用 Alternate Domain Names （有CNAME的情况下）或是 Distribution 自带的域名如 `dxxxxxxxxx.cloudfront.net`（没有CNAME的情况下） 来匹配请求的 Host，所以并不总是 DNS 中指定的地址。

以下说明引用自 AWS Support 给我的回答：
> 因为 CloudFront 服务的边缘站点使用的是一个资源共享的模式，所以在默认的情况下 Distribution 本身并没有独立的专属 IP 地址，而是共享同一群IP地址，因此需要通过 Host 标头去匹配请求和 Distribution 之间的关联。
>
> 在上面的实验中，从访问 `test.example.com` 到第二个 Distribution 的源站内容的过程中其实产生了两个连接：   
> 第一个连接是从 `test.example.com` 到整个 CloudFront 服务，这个连接的解析将会通过 `test.example.com` 所在的 DNS 管理服务器来完成；  
> 第二个连接则是从 CloudFront 到达具体的 CloudFront Distribution，这一部分是由 CloudFront 来完成，所以这个时候 CloudFront 会去寻找将 `test.example.com` 作为 CNAME 的 Distribution，然后进行连接。


## 可以不设置 Alternate Domain Names (CNAMEs) 吗？

不可以。

从上面的实验中也可以看到，不配置 CNAMEs 会收到 CloudFront 的报错 The request could not be satisfied 。为什么呢？

由上面 CloudFront 域名劫持的原理可以知道，如果不指定 CNAME，通过自己的域名去访问时，CloudFront 将不知道该把请求匹配到哪个 Distribution 上。因此，即使不考虑有人恶意进行 CloudFront 域名劫持，也应该设置 CNAME。


## 如何防止避免 CloudFront 域名劫持

### 用户方面

1. 创建 CloudFront Distribution 时一定要配置 Alternate Domain Names (CNAMEs)。
2. 删除不再使用的 Distribution 后，一定要把指向该 Distribution 的域名解析一起删除掉。
3. 如果子域名都在同一账号上，建议使用泛域名如 `*.example.com` 作为 CNAME 以避免任何别的账号配置到 `example.com` 的域名。

### AWS方面

AWS 也做了一些保护措施：

1. 当你从 Distribution 删除 Alternate Domain Names (CNAMEs) 时，AWS 会提醒你删除指向 CloudFront 的 DNS 记录。
2. AWS 会持续性检查，如你尝试添加泛域名 `*.example.com` 到 CNAMEs 失败了，就说明已经有别的账号添加了 `example.com` 的子域名。这时可以联系 AWS Support 来验证域名归属，处理这个问题。
3. 当你从 Distribution 删除 Alternate Domain Names (CNAMEs) 时，AWS 还会检查该解析是否仍指向你的 Distribution。在删除这条解析之前，将不能再添加这个域名到 Alternate Domain Names (CNAMEs)。
4. 域名劫持违反了 AWS 的服务条例，如果有人这么做，AWS 将进行封号处理。


## 总结

1. 实践出真知。对这个问题我一直有疑惑，直到动手实验才真正理解。
2. 透过表象看本质。不配置 CNAMEs 就会报错 The request could not be satisfied，从这里就应该可以推测出 CloudFront 是通过匹配请求中的 Host 标头来关联 Distribution的。
3. 不知道其他厂商的 CDN 是什么原理，会不会有类似的问题，以后遇到要注意。


## 参考资料

1. [AWS Security Blog - Enhanced Domain Protections for Amazon CloudFront Requests](https://amazonaws-china.com/tw/blogs/security/enhanced-domain-protections-for-amazon-cloudfront-requests/)
2. [How do I resolve the error CNAMEAlreadyExists when setting up a CNAME alias for my Amazon CloudFront distribution?](https://amazonaws-china.com/premiumsupport/knowledge-center/resolve-cnamealreadyexists-error/?nc1=h_ls)
3. AWS Support