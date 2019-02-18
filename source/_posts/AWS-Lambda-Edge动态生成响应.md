---
title: AWS Lambda@Edge动态生成响应
date: 2019-02-18 23:08:12
permalink: aws-lambda-generate-response
tags:
- AWS
- AWS Lambda
categories:
- DevOps
---

最近用了不少 AWS Lambda@Edge，前两篇文章记录了我用 Lambda@Edge 实现请求重定向的过程，今天我又完成了一项新的需求，使用 Lambda@Edge 动态生成响应。
<!--more-->

## 背景

国内的网站都需要备案，备案需要在网站的页脚上注明公司名称和备案号。我司有很多站点都需要备案，这些站点都需要一个类似包含如下信息的备案页：站点名称、公司名称、备案号、copyright、logo图片。现在每个站点都有一个这样的 html 静态页面放在服务器上，由 nginx 对外提供服务。这样做有如下几个问题：
1. 成本较高，需要提供一台 nginx 服务器。
2. 维护麻烦，新增站点要创建新的 html 页面。
3. 需要运维保障服务器平稳运行。

要解决以上问题，还是需要无服务器化。有同学可能会想到使用对象存储来托管静态资源，这可以解决上面的第1点和第3点。要解决上面全部3个问题，这次我的做法是使用 Lambda@Edge 动态生成响应。


## 方案概览

1. 将各站点的备案页域名都指向 CloudFront。
2. CloudFront 上可以配置适配多个域名的 SSL证书（通过 AWS Certificate Manager申请）。
3. 编写 Lambda 函数部署到 CloudFront 上拦截 ViewerRequest（查看器请求）。
    3.1 获取请求站点：`request.headers.host[0].value`。
    3.2 根据请求的站点从配置获取公司名称，备案号和 Logo URI。
    3.3 按当前日期生成 copyright 信息，例如：`2019 © XXX公司`。
    3.4 将以上信息填充到 HTML 模板中，生成响应。
    3.5 返回响应到用户。

在 Lambda@Edge 生成响应并返回非常简单，同重定向类似：
```JavaScript
const pageHTML = '<html>...</html>';
const response = {
        status: '200',
        statusDescription: 'OK',
        headers: {},
        body: pageHTML,
    };

callback(null, response);
```

我的代码如下：
```JavaScript
'use strict';

const companyName = 'XXX有限公司';
const imageUri = 'https://example.com/logo.png';
const icpInfos = {
    'www.aaa.com': {
        'companyName': '' || companyName ,
        'icpNumber': '粤ICP备12345678号-6',
        'imageUri': '' || imageUri
    },
    'www.bbb.net': {
        'companyName': '' || companyName ,
        'icpNumber': '粤ICP备12345678号-4',
        'imageUri': '' || imageUri
    },
};

exports.handler = (event, context, callback) => {
    const request = event.Records[0].cf.request;
    const host = request.headers.host[0].value;
    const icpInfo = icpInfos[host];
    if(!icpInfo){
        callback(null, request);
        return;
    }
    
    const year = new Date().getFullYear();
    const pageHTML = `<!DOCTYPE html><html lang="zh-cmn-Hans"><head><meta charset="UTF-8">
        <title>${host}</title>
        <style>
            *{
              margin: 0;
              padding: 0;
              background-color: #f7f7f7;
            }
            h1{
              text-align: center;
              font-size: 48px;
              padding: 10%;
            }
            p{
              position: absolute;
              bottom: 0;
              left: 0;
              width: 100%;
              text-align: center;
              color: #999;
              padding: 10px 0;
            }
            p a {
              color: #999;
              padding-left: 20px;
            }
        </style>
        </head>
        <body>
          <p style="text-align:center"><img src="${icpInfo.imageUri}"></p>  
          <h1>${host}</h1>
          <p>${year} &copy; ${icpInfo.companyName} <a href="http://www.miitbeian.gov.cn/">${icpInfo.icpNumber}</a></p>
        </body>
        </html>`;
        
    const response = {
        status: '200',
        statusDescription: 'OK',
        headers: {},
        body: pageHTML,
    };

    callback(null, response);
    
};
```

## 总结

1. 用户从浏览器访问域名，域名解析到 CloudFront，触发 Lambda@Edge 的 ViewerRequest。在 ViewerRequest 中根据站点、当前时间动态生成正常包含相应备案号、copyright 时间等信息的 HTTP 响应。
2. Lambda@Edge 运行在靠近用户的 CloudFront 边缘节点上，响应更快。
3. 新增站点只需要在 Lambda@Edge 代码中添加配置，无需创建 HTML 文件。
4. copyright 信息可以动态更新（当然，静态资源也可以用 JavaScript 实现）。
5. 成本低，无需维护，稳定性高。

相比之前使用 Lambda@Edge 做重定向，其实这次的需求也没有太多新东西，理解了原理就非常简单，但实现的效果非常棒！
