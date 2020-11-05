---
title: 再聊 HTTP Header Vary 
date: 2020-11-04 21:17:11
permalink: talk-about-http-header-vary-again
tags:
- CDN
- HttpHeader
categories:
- DevOps
---

几个月前写过一篇关于 HTTP Header `Vary` 的[文章](https://chowyi.com/vary-response-header-causes-miss-cdn-cache/)。上次它给我带来了一些麻烦，今天再来聊聊它，这次看看它是怎么帮到我的。

<!--more-->

## 背景

1. 我有一些静态资源文件存储在阿里云 OSS Bucket 中，前置 CDN 做缓存，并在 CDN 上绑定了自定义域名（假设域名为 xxx.com）。
2. 有多个站点 aaa.com, bbb.com, ccc.com 等需要发起 XHR 请求经过 CDN 取得存储桶中的静态资源 `xxx.com/test.js`。
3. 上述调用显然会遇到跨域问题。（[什么是跨域资源共享CORS？](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)）
4. 解决跨域问题也很简单，在响应头中指定 `Access-Control-Allow-Origin` 即可。阿里云 OSS 提供了配置 CORS 规则的功能。

![aliyun_console_cors_conf](https://blog-1252856176.file.myqcloud.com/post/talk-about-http-header-vary-again/cors_conf.png)

## 问题

通过查看 `Access-Control-Allow-Origin` 的[文档]()可以知道，它只能有一个值，要么设置为`*`允许所有站点CORS，要么指定允许单个站点CORS `Access-Control-Allow-Origin: https://developer.mozilla.org`。如果要允许多个站点CORS怎么做呢？那就需要在后端添加逻辑，对请求返回响应的时候，将相应的 `Origin` 设置为 `Access-Control-Allow-Origin` 的值。显然阿里云考虑到了这一点，可以为 OSS Bucket 配置多条 CORS 规则，针对不同来源的请求可以给出不同的响应头。

配置完成，接下来开始测试。
从 aaa.com 发起 XHR 请求资源，响应头为 `access-control-allow-origin: http://www.aaa.com`，没有问题。

![cors_allowed](https://blog-1252856176.file.myqcloud.com/post/talk-about-http-header-vary-again/cors_allowed.png)

从 bbb.com 发起 XHR 请求资源，响应头还是为 `access-control-allow-origin: http://www.aaa.com`，CORS头不匹配，跨域请求被拦截。

原来从 bbb.com 发起的请求复用了 CDN 上之前对 aaa.com 响应的缓存！

![cors_not_allowed](https://blog-1252856176.file.myqcloud.com/post/talk-about-http-header-vary-again/cors_not_allowed.png)

## 解决

看到报错,我马上就想起来在阿里云控制台上配置 CORS 规则的时候，就看到了有一个 Response Vary 选项。当时没太理解这个选项有什么用，现在恍然大悟，勾选上就可以了。  

在阿里云 OSS 控制台配置 CORS 规则的时候，勾选上 Response Vary 选项，在返回的响应头中除了 `Access-Control-Allow-Origin` 还会添加 `Vary: Orign`。这个头部告诉 CDN，如果两个请求的`Origin`头部的值不同，就不能共用同一份缓存。  

现在去刷新CDN缓存，然后重新从 aaa.com 和 bbb.com 分别发起请求，可以看到它们都得到了正确的响应。

## 总结

1. [上一次](https://chowyi.com/vary-response-header-causes-miss-cdn-cache/)因为阿里云CDN智能压缩功能自动添加的 `Vary: Accept-Encoding` 头部导致用预热的缓存不能被用户请求复用。需要去掉这个头部。
2. 这一次因为缺少了 `Vary: Origin` 导致多个源共用了同一份缓存，CORS 策略失效。
3. 我在最初没有配置 `Vary: Origin` 暴露出两个问题：
    3.1 我对 `Access-Control-Allow-Origin` 不够了解，以为它可以有逗号分隔的多个值。
    3.2 我对 `Vary` 了解不够深刻，经过之前的学习后，没能想到更多实际的应用场景（比如本次遇到的问题）。
4. 表扬阿里云，OSS 控制台上可以轻松的配置这些选项。
5. 吐槽阿里云，文档不清晰，看了没看懂。SDK 功能不全，还不支持配置 Response Vary。
