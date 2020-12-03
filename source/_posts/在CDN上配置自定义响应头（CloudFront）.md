---
title: 在CDN上配置自定义响应头（CloudFront）
date: 2020-03-11 13:37:34
permalink: custom-http-headers-on-cdn/
tags:
- AWS
- AWS Lambda
- CloudFront
categories:
- DevOps
---

今天接到一个小需求，要在业务系统的 HTTP 响应中添加几个安全相关的响应头。我的第一反应是在 Kong/Nginx 上配置，但我们的业务现在已经是 ServerLess 架构了，那么就要在 CDN 上做配置了。

我这里业务上用的 AWS CloudFront 和阿里云 CDN，同时我又去看了一下腾讯云 CDN 的配置方式，这里简单对比一下，我想重点说明的是 CloudFront。
<!--more-->

## 背景

需要在网站的响应中添加下面几个安全相关的响应头：


> [X-Content-Type-Options](https://developer.mozilla.org/docs/Web/HTTP/Headers/X-Content-Type-Options): nosniff

> [X-XSS-Protection](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/X-XSS-Protection): 10;mode=block

> [X-Frame-Options](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/X-Frame-Options): deny

> [Strict-Transport-Security](https://developer.mozilla.org/docs/Web/HTTP/Headers/Strict-Transport-Security): max-age=31536000; includeSubDomains

## 腾讯云

腾讯云的配置就很简单了，直接在控制台添加就可以了。

![qcloud-cdn-edit-http-headers](https://blog-1252856176.file.myqcloud.com/post/custom-http-headers-on-cdn/qcloud-cdn-edit-http-headers.png)

## 阿里云

阿里云的控制台也有配置 HTTP Headers 的地方，但有两点不足之处：

![aliyun-cdn-edit-http-headers](https://blog-1252856176.file.myqcloud.com/post/custom-http-headers-on-cdn/aliyun-cdn-edit-http-headers.png)

1. 配置的入口不太合理，不是常用阿里云的同学可能不太好找，它位于：`CDN控制台 -> 域名管理 -> 缓存配置 -> HTTP头`。
2. ~~目前（2020-03-11），在控制台上只支持配置以下 10 个 HTTP Headers，如果要配置其他 HTTP Header，需要提交工单处理。~~  
2020-12-03 Update：现在看阿里云已经支持在控制台上配置自定义Header了。

    - Content-Type
    - Cache-Control
    - Content-Disposition
    - Content-Language
    - Expires
    - Access-Control-Allow-Origin
    - Access-Control-Allow-Headers
    - Access-Control-Allow-Methods
    - Access-Control-Max-Age
    - Access-Control-Expose-Headers

我这次需求就需要通过提交工单来完成，好在工单处理的速度还算快，从我提交工单 -> 灰度节点测试 -> 全量部署，一个小时完成（工作日工作时间）。 不太理解阿里云为什么不能像腾讯云那样，让用户在控制台上就可以添加自定义的 HTTP Header。

## AWS CloudFront

CloudFront 配置 HTTP 响应头相对来说复杂一些，它不支持在控制台上直接配置添加，需要配合 Lambda@Edge 来实现，对用户来说也要求具备基本的开发能力（NodeJS/Python）。但同时它也更加强大，可以灵活应对多种多样的用户需求。如果用照相机来打比方，它更像一台专业的单反相机：一开始你也许会觉得它难于使用，甚至还不如手机拍照来的方便好看，然而一旦你掌握它之后，就能将它的强大功能为己所用，灵活创造出多种玩法。

前面说到，CloudFront 配置 HTTP 响应头需要配合 Lambda@Edge 来实现。触发 Lambda 函数的 CloudFront 事件一共有四种：

- Viewer Request
- Origin Request
- Origin Response
- Viewer Response

![cloudfront-events-that-trigger-lambda-functions](https://docs.aws.amazon.com/zh_cn/lambda/latest/dg/images/cloudfront-events-that-trigger-lambda-functions.png)

那么如何确定要在哪里配置我们的 Lambda 函数呢？

以下摘自官方文档-[如何确定用于触发 Lambda 函数的 CloudFront 事件](https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/lambda-how-to-choose-event.html):
> **您是否希望 CloudFront 缓存 Lambda 函数更改的对象？**
>
>    如果您希望 CloudFront 缓存 Lambda 函数修改的对象，以便在下次请求该对象时 CloudFront 可以从边缘站点中提供该对象，请使用源请求或源响应事件。这样可减少源上的负载、减少后续请求的延迟，并降低对后续请求调用 Lambda@Edge 的成本。
>
>    例如，如果要添加、删除或更改由源返回的对象的标头，并且希望 CloudFront 缓存结果，请使用源响应事件。

官方文档上还提供了一段示例代码：[示例：覆盖响应标头](https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/lambda-examples.html#lambda-examples-overriding-response-header)


我这里需要 CloudFront 来缓存我的响应，因此我要对`Origin Response` 编写 Lambda 函数。下面附上我的代码吧。

在`config`中可以配置按四种方式修改响应头。
1. add: 当且仅当`add`中指定的头部**不**存在时，添加头部。若已经存在，则忽略此配置。
2. replace: 当且仅当`replace`中指定的头部存在时，替换头部。若不存在，则忽略次配置。
3. remove: 当且仅当`remove`中指定的头部存在时，移除头部。若不存在，则忽略次配置。
4. append: 当`append`中指定的头部存在时，保持原头部不变，新增同名头部。若不存在，则直接添加头部。

```javascript
const config = {
    "add": [
        { "name": "X-Content-Type-Options", "value": "nosniff" },
        { "name": "X-XSS-Protection", "value": "10;mode=block" },
        { "name": "X-Frame-Options", "value": "deny" },
        { "name": "Strict-Transport-Security", "value": "max-age=31536000;includeSubDomains" }
    ],
    "replace": [],
    "remove": [],
    "append": [],
};


exports.handler = async (event, context, callback) => {

    const response = event.Records[0].cf.response;
    const respHeaders = response.headers;

    let addHeaders = config['add'];
    let replaceHeaders = config['replace'];
    let removeHeaders = config['remove'];
    let appendHeaders = config['append'];

    // If and only if the header is already set, replace its old value with the new one. Ignored if the header is not already set.
    if (replaceHeaders) {
        for (let i = 0; i < replaceHeaders.length; i++) {
            let h = replaceHeaders[i];
            if (respHeaders[h['name'].toLowerCase()]) {
                respHeaders[h['name'].toLowerCase()] = h['value'].split(',').map((v) => { return { 'key': h['name'], 'value': v.trim() } });
                console.log(`Replaced Header(${h['name']}).`);
            }
            else {
                console.log(`Header(${h['name']}) does not exists, ignore replacing.`);
            }
        }
    }

    // Unset the header(s) with the given name.
    if (removeHeaders) {
        for (let i = 0; i < removeHeaders.length; i++) {
            let h = removeHeaders[i];
            if (respHeaders[h['name'].toLowerCase()]) {
                delete respHeaders[h['name'].toLowerCase()];
                console.log(`Removed Header(${h['name']}).`);
            }
            else {
                console.log(`Header(${h['name']}) does not exists, ignore removing.`);
            }
        }
    }

    // If the header is not set, set it with the given value. If it is already set, a new header with the same name and the new value will be set.
    if (appendHeaders) {
        for (let i = 0; i < appendHeaders.length; i++) {
            let h = appendHeaders[i];
            if (respHeaders[h['name'].toLowerCase()]) {
                respHeaders[h['name'].toLowerCase()] = respHeaders[h['name'].toLowerCase()].concat(h['value'].split(',').map((v) => { return { 'key': h['name'], 'value': v.trim() } }));
                console.log(`Appended Header(${h['name']}).`);
            }
            else {
                respHeaders[h['name'].toLowerCase()] = h['value'].split(',').map((v) => { return { 'key': h['name'], 'value': v.trim() } });
                console.log(`Added(by append) Header(${h['name']}).`);
            }
        }
    }

    // Add function must be after Replace/Remove/Append. Otherwise added headers may be overwritten.
    // If and only if the header is not already set, set a new header with the given value. Ignored if the header is already set.
    if (addHeaders) {
        for (let i = 0; i < addHeaders.length; i++) {
            let h = addHeaders[i];
            if (respHeaders[h['name'].toLowerCase()]) {
                console.log(`Header(${h['name']}) already exists, ignore adding.`);
            }
            else {
                respHeaders[h['name'].toLowerCase()] = h['value'].split(',').map((v) => { return { 'key': h['name'], 'value': v.trim() } });
                console.log(`Added Header(${h['name']}).`);
            }
        }
    }


    callback(null, response);
};
```

## 总结

1. 腾讯云可以直接在 CDN 控制台配置自定义的响应头部。
2. ~~阿里云可以在 CDN 控制台配置部分响应头部，特殊的头部需要提工单后台处理。~~ 2020-12-03 Update: 已支持控制台配置自定义的响应头部。
3. AWS CloudFront 不支持在控制台配置响应头部，需要配合 Lambda@Edge 自行编写函数处理。

仅对 CDN 配置 HTTP Headers 而言，腾讯云最简单易用，AWS 最灵活强大。
