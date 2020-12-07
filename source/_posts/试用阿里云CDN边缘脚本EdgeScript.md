---
title: 试用阿里云CDN边缘脚本EdgeScript
date: 2020-12-06 11:20:29
permalink: try-aliyun-cdn-edgescript/
tags:
- CDN
- EdgeScript
- AWS Lambda
categories:
- DevOps
---

最初了解函数计算这个概念是从 AWS Lambda 开始的，后来又知道了 Lambda@Edge，我当时学习之后直呼 Amazing，简直强大又灵活，后来我们的很多业务都用 Lambda@Edge 做了架构演进。遗憾的是 Lambda@Edge 这个好东西只有 AWS Global 才能用，国内特供版的 AWS 是没有这个功能的，那就只能寄希望于阿里云国货当自强了。当时的大概是2018年底，跟阿里云也有过沟通，他们有计划要上线一个类似的功能。到了2019年底，阿里云 EdgeScript 上线了，鉴于刚上线的产品通常都不太稳定，我也没有怎关注。转眼时间已经来到了2020年底，我来试用一下阿里云的 EdgeScript，看看功能如何。
<!--more-->

阿里云作为国内一流的云服务应该是毋庸置疑的，我不知道把它跟世界一流的 AWS 放在一起比是否公平，但我用的最多的也就是这两家了。无意捧一踩一，仅从技术角度做功能对比。

## 语法

### Lambda@Edge
AWS Lambda@Edge 支持 Python, Ruby, Node.js, Go, Java, .Net 几种常用编程语言的多个版本，我可以根据我的技能栈，第三方库，或是性能要求选择我喜欢的语言来实现需求。学习成本几乎为零，更可以借助这些编程语言本身的强大特性。  

比如我擅长 Python 和 JavaScript，在编写 Lambda@Edge 函数时，我喜欢选择 Node.js 的环境，因为 Node.js 自带的 `url`, `path`, `https` 等package可以很方便的处理 url 和 request 相关功能。

### EdgeScript
EdgeScript 是一套自定义语法规则，详见文档[EdgeScript语法](https://help.aliyun.com/document_detail/126566.html)。我并不介意去学习一套新的语法规则，但 EdgeScript 的语法似乎过于简陋了。比如：
1. 只有 `if...else...` 语句，不支持 `else if`, 要通过嵌套方式实现。
2. 没有 `for`, `while` 等循环语句，通过用于处理字典类型的内置函数`foreach(d, f, user_data)`实现，灵活性较差。
3. 运算符仅有`=`赋值运算符和`-`负号运算符，其他运算均需使用内置函数。
4. 我最不能接受的一点：**EdgeScript全文不允许出现任何双引号。**

通过与阿里云技术支持沟通，我了解到在需要用到双引号的地方要用`tochar()`来代替，`tochar(39)`是单引号 ，双引号是`tochar(34)`。再加上连接字符串没有`+`加号运算符，要用`concat`函数，处理起字符串就特别复杂甚至无法手工编写，我需要另写一个 Python 脚本来生成在 EdgeScript 中拼接字符串的代码。比如：之前我们使用 Lambda@Edge 对很多简单的静态页面做了 Serverless 改造，直接通过 Lambda@Edge 生成响应体（一个场景是生成一个简单的包含公司主体信息的备案页）。在 AWS Lambda@Edge 中我可以直接把整段 HTML 作为字符串放在响应体中返回，甚至还可以利用 Javascript 的模板字符串功能在其中替换变量。而这个简单的需求在 EdgeScript 中就成了灾难：以一个 HTML 中的 a 标签为例，`<a href="http://example.com/">EXAMPLE</a>`，这个标签在 EdgeScript 中需要写成这样：`concat('<a href=', tochar(34), 'http://example.com/', tochar(34), '>EXAMPLE</a>')`。如果要生成一个完成的 HTML 页面，手写几乎无法实现。

## 依赖库

在 EdgeScript 中只能使用自定义函数或内置的函数，并且由于它简陋的语法，很难实现复杂的功能。而在 AWS Lambda@Edge 中，你可以引入第三方库，比如 Python 的 requests 或是其他任何语言的任何 SDK。

## 脚本管理

### EdgeScript
- EdgeScript 为单个脚本文件，仅能通过 Web IDE 编写保存。
- 无版本控制。  

### Lambda@Edge
- AWS Lambda@Edge 可以通过 Web IDE 编写或上传 Zip 文件。
- 支持以目录形式组织的多个文件，可以分模块编写代码或引入第三方库。
- 每次部署代码都会保存为一个版本。
- 有 Layer 的概念，可以多个函数共享一些通用的组件。

## 调试

### EdgeScript
EdgeScript 可以发布至模拟环境，本地绑定 Hosts 后可以较好的模拟生产环境进行测试。

### Lambda@Edge
Lambda@Edge 可以配置测试事件，在线运行函数，可以在控制台上看到运行结果，程序输出和报错信息。

## 日志

### EdgeScript
暂时没看到如何记录和查看日志。

### Lambda@Edge
函数的调用及函数中的输出会记录在 CloudWatch 日志组中。

## Web IDE

EdgeScript WebIDE 具有简单的代码高亮和代码提示功能。  

Lambda@Edge WebIDE 支持多种编程语言，功能较为完备。

## 部署

Lambda@Edge 基于版本管理函数，同一个函数可以部署在多个 CloudFront Distribution 上，函数易复用。  

EdgeScript 每个 CDN 对应单独的 EdgeScript。

## 执行位置

Lambda@Edge 可以部署在一次请求的四个阶段：`Viewer request`, `Origin request`, `Origin response`, `Viewer response`。

![Lambda@Edge的四个阶段](https://docs.aws.amazon.com/zh_cn/lambda/latest/dg/images/cloudfront-events-that-trigger-lambda-functions.png)

EdgeScript 执行位置为请求处理开始和请求处理结尾。（我理解对应Lambda@Edge为 Viewer Request 和 Origin request。）

![EdgeScript的执行位置](http://static-aliyun-doc.oss-cn-hangzhou.aliyuncs.com/assets/img/7064139651/p62071.png)

## 其他

目前阿里云推出了很多服务，但这些服务之间整合的还不够好。比如除了 EdgeScript，还有 EdgeRoutine, 边缘节点服务ENS，函数计算。其他的几项服务我还不太了解，去使用这些服务还是有比较高的学习成本。AWS 整合的比较好，Lambda 是基本款的函数计算服务，部署在 CloudFront 上就是 Lambda@Edge，在理解和使用上有一个比较统一的体验。类似的问题还有阿里云的 CDN、全站加速、WAF 和 SCDN（其实都是一回事，应该整合为一个产品）。

## 总结

Aliyun CDN EdgeScript 目前来看还不够成熟，与其说是 Script，不如说是一个功能更强的 config。在某些简单的场景下用起来应该还不错，比如修改 Http Headers 或是处理简单的重定向。想要完全用 EdgeScript 实现 Serverless 还是比较困难的。不知道阿里云处于什么考虑，EdgeScript 使用了一套自定义的语法规则。未来如果要实现更强大的功能，还是需要支持至少一门主流编程语言。
