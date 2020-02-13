---
title: Vary response header导致的CDN预热无效
permalink: vary-response-header-causes-miss-cdn-cache
date: 2020-02-12 12:29:43
tags:
- CDN
- HttpHeader
categories:
- DevOps
---

今年春节被新型冠状病毒（2019-nCoV）困在家里了，哪也不敢去。不过远程工作已经开始了，前两天需要在阿里云CDN上预热一个静态资源，预热后浏览器访问却依旧很慢，响应头表明缓存未命中。咨询了阿里云技术支持，又查找了一些资料，终于搞明白了，详细记录如下。太长不看版请直接跳到文章尾部[总结](#总结)。
<!--more-->

## 背景

现有静态资源存在于 AWS S3 存储桶中，国内用户访问速度慢且不稳定，需要借助阿里云CDN做加速，回源 S3 并缓存文件。

## 预期现象

在阿里云控制台完成CDN预热后，国内用户访问直接命中CDN缓存，实现加速效果。

## 实际现象

在阿里云控制台完成CDN预热后，从浏览器访问该资源依然很慢（40s左右），响应头`x-cache: MISS TCP_MISS dirn:-2:-2`表明未命中CDN缓存。


## 问题原因

通过与阿里云技术支持沟通，初步定位到问题原因是响应头中包含了`Vary: Accept-Encoding`字段。  
那么响应头`Vary`是起到什么作用的呢？为什么会导致CDN预热后请求仍然无法命中缓存呢？  

我先查了[MDN文档中有关Vary的部分](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Vary):
> The Vary HTTP response header determines how to match future request headers to decide whether a cached response can be used rather than requesting a fresh one from the origin server. It is used by the server to indicate which headers it used when selecting a representation of a resource in a content negotiation algorithm.

简单说就是，Vary Header 的值应该是其他一个或多个 Header Name，用来告诉服务器(Web服务器、CDN等)，在决定是直接返回缓存还是从源站重新取时，要判断 Vary Header 指定的头部是否相同。  

在本次场景下，响应头`Vary: Accept-Encoding`表示，如果两个请求的 Accept-Encoding Header 不同，即使其他参数、路径相同，服务器(Web服务器、CDN等)也应该把它们看做是不同的请求，不能复用缓存。  

通过与阿里云的技术支持沟通我了解到，阿里云CDN在进行预热操作时，发出的请求类似`curl 'https://www.example.com/test.js'`，但浏览器发出请求头中包含了 Accept-Encoding，类似于`curl 'https://www.example.com/test.js' -H 'accept-encoding: gzip, deflate, br'`，因此对于CDN来说，浏览器发出请求不同于预热时发出请求，也就不能复用缓存了。

## 解决办法

如上所述，由于响应头中包含了`Vary: Accept-Encoding`字段，导致用户请求不能复用CDN预热时生成的缓存。解决办法就是去掉响应头中的`Vary`字段。

我的源站是 AWS S3，我检查之后发现源站并没有在响应中添加`Vary`这个头部，再次联系阿里云技术支持，原来是阿里云CDN开启[智能压缩](https://help.aliyun.com/document_detail/27127.html)功能后会向请求中添加这一字段。关闭智能压缩功能，刷新CDN并重新预热就可以了。

## 总结

1. 响应头中的`Vary`字段表示：如果两个请求`Vary`指定的头部不同，则服务器(Web服务器、CDN等)不应该复用缓存。
2. 解决办法是去掉响应头中的`Vary`字段。
3. 响应头中的`Vary`字段是阿里云CDN智能压缩功能添加的。

