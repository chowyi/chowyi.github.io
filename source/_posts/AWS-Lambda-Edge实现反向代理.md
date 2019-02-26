---
title: AWS Lambda@Edge实现反向代理
date: 2019-02-21 21:37:41
permalink: reverse-proxy-by-lambda-at-edge
tags:
- AWS
- AWS Lambda
categories:
- DevOps
---


前面几篇文章记录了我学习使用 AWS Lambda@Edge 由浅入深的过程，通过 Lambda@Edge 实现了请求重定向和动态生成响应：

- [AWS Lambda@Edge小试牛刀——代替Nginx处理请求](/handle-request-like-nginx-by-lambda-at-edge/)
- [使用Route53配置顶级域名跳转的两种方式——Lambda@Edge再显身手](/two-ways-to-redirect-domain-zone-apex/)
- [AWS Lambda@Edge动态生成响应](/aws-lambda-generate-response/)

今天再进一步，使用 Lambda@Edge 实现反向代理。

<!--more-->

## 背景

使用 Lambda@Edge 替代了之前 Nginx 重定向、静态资源托管等功能后，我在想能不能彻底把我们的 Nginx 服务器干掉，既节省成本，又省心。那就要试验一下 Nginx 另一大常用功能——反向代理能不能在 Lambda@Edge 上实现。


## 动手前的一些思考

1. 反向代理和请求重定向有什么区别？  
    请求重定向的过程：  

        1. Client 发送 request 到 Server(HTTPServer/Nginx/Lambda@Edge)。
        2. Server 收到 request 后，返回 redirectResponse（状态码为 301 或 302，Headers包含 Location 属性）。
        3. Client 收到 response 后，取得 Headers 中的 Location 属性值。
        4. Client 再次发送 request 到 Location 指向的 Server。

    请求重定向的整个过程中，Client 一共发送了两次 request，两次 request 相对独立。Client 明确的知道响应是谁返回的。

    反向代理的过程：

        1. Client 发送 request 到 Server(HTTPServer/Nginx/Lambda@Edge)。
        2. Server 收到 request 后保持连接，将请求转发至配置中预设的 backend，等待 backend 响应。
        3. Server 收到 backend 的 response 后，再转回给 Client。

    反向代理的过程中：对 Client 来说，只有一次请求一次响应，反向代理服务器（Server）代理了到 backend 间请求和响应转发的过程。Client 只能看到反向代理服务，并不关心响应到底是谁生成的。

2. 如何实现请求和响应的转发？  
    如果从字面上理解”转发“这个词就把问题想复杂了，我们并不需要真正的把从 Client 接到的 request ”转发“ 到 backend，实际上我们只需要重新发送一个与原请求内容一致的 request 即可，”转发“响应也是同理。只要符合 HTTP 协议所制定的规则，我们可以根据需求生成并发送各种各样的请求或响应。

3. 在转发请求和响应的过程中，具体要转发什么内容？  
	- Path
	- Headers
	- Content


## 实践

对 Lambda@Edge 的编程模型（解析请求，生成响应，callback机制）我已经非常熟悉了，我现在需要去了解一下 nodejs 处理 http 请求的相关模块，因为我是 Python 技能栈，对 nodejs 还不太熟悉，而我的 Lambda 函数是 nodejs 写的（AWS 提供的 blueprint 就是 nodejs 写的，应该也支持其他语言，但此场景下 nodejs 的效率似乎略高于 python）。


### 在 nodejs 中发送 http 请求

在 Lambda 中使用原生的库更加方便一些，虽然现在 AWS 也提供了 Layer 功能来管理第三方组件。NodeJs 自带 http 和 https 两个模块用来处理请求相关功能。  
查[官方文档](https://nodejs.org/api/http.html#http_http_request_options_callback)看一下，很简单。下面是文档中的例子：
```JavaScript
const postData = querystring.stringify({
  'msg': 'Hello World!'
});

const options = {
  hostname: 'www.google.com',
  port: 80,
  path: '/upload',
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'Content-Length': Buffer.byteLength(postData)
  }
};

const req = http.request(options, (res) => {
  console.log(`STATUS: ${res.statusCode}`);
  console.log(`HEADERS: ${JSON.stringify(res.headers)}`);
  res.setEncoding('utf8');
  res.on('data', (chunk) => {
    console.log(`BODY: ${chunk}`);
  });
  res.on('end', () => {
    console.log('No more data in response.');
  });
});

req.on('error', (e) => {
  console.error(`problem with request: ${e.message}`);
});

// write data to request body
req.write(postData);
req.end();
```

唯一要注意的是文档中下面这句话：
> Note that in the example req.end() was called. With http.request() one must always call req.end() to signify the end of the request - even if there is no data being written to the request body.

一开始没注意看，以为调用了 `http.request()` 就发送请求了呢，浪费了很多时间调试。看文档还是要仔细！


### 生成响应

在 `http.request()` 方法的回调中，就可以拿到 backend 响应并生成新的 response，然后调用 Lambda@Edge 的 callback 返回给 Client 了。这里要注意的是：

1. 不能直接将 `http.request()` 回调中的响应通过 `callback` 函数返回给 Client，也好理解，毕竟格式不一样嘛。
2. Lambda@Edge 中请求和响应的 Headers 中的属性值都是 Array，需要这样写：
```JavaScript
const headers = {'host': [{'key': 'Host', 'value': 'example.com'}]}
```
(Headers 中的属性名大小写不敏感）
3. 在 Lambda@Edge 中有一些标头（Headers）被列在黑名单中不能添加或修改，还有一些标头在特定的生命周期中是只读的。如果尝试添加或修改这些 Headers，Lambda@Edge 会返回 502。所以在将 backend 返回的响应转回 Client 前，要把 Headers 中的这些标头过滤掉。详见文档：[Lambda 函数的要求和限制](https://docs.aws.amazon.com/zh_cn/AmazonCloudFront/latest/DeveloperGuide/lambda-requirements-limits.html#lambda-header-restrictions)

### 我的代码

下面是我编写的 Lambda 函数的完整代码，实现了 Nginx 的 rewrite 和 proxy_pass 功能，可以通过配置灵活添加多个 server 或 location。我对 NodeJs 的了解有限，如有不妥之处，恳请指出。

```JavaScript
'use strict';

const url = require('url');
const pathUtil = require('path');
const http = require('http');
const https = require('https');

const servers = [
    {
        'name': 'test.example.net',
        'locations': {
            '/': {
                'rewrite': {
                    'status': '302',
                    'statusDescription': 'Moved Temporarily',
                    'location': 'https://m.example.com',
                }
            },
            '/aaa': {
                'proxy_pass': 'http://download.example.net',
            },
            '/bbb': {
                'proxy_pass': 'http://download.example.net',
            },
        },
    },
];

const LambdaViewerRequestReadOnlyHeaders = ['Content-Length','Host','Transfer-Encoding','Via'];
const lambdaHeaderBlacklist = ['Connection', 'Expect', 'Keep-alive','Proxy-Authenticate','Proxy-Authorization','Proxy-Connection','Trailer','Upgrade','X-Accel-Buffering','X-Accel-Charset','X-Accel-Limit-Rate','X-Accel-Redirect','X-Amz-Cf-*','X-Amzn-*','X-Cache','X-Ede-*','X-Forwarded-Proto','X-Real-IP'];

function isForbidHeader(headerName, forbidHeaders){
    for(let i=0;i<forbidHeaders.length;i++){
        if(headerName.toLowerCase() === forbidHeaders[i].toLowerCase()){
            return true;
        }
    }
    return false;
}


exports.handler = (event, context, callback) => {
    const request = event.Records[0].cf.request;
    const oldURI = request.uri;
    const method = request.method;
    let host = request.headers.host[0].value;
    
    for(let i=0;i<servers.length;i++){
        let server = servers[i];
        if(server.name !== host){
            continue;
        }
        else{
            if(server.locations){
                for(let path in server.locations){
                    if(path !== oldURI){
                        continue;
                    }
                    let location = server.locations[path];
                    if(location.rewrite){
                        const response = {
                            status: location.rewrite.status,
                            statusDescription: location.rewrite.statusDescription,
                            headers: {
                                location: [{
                                    key: 'Location',
                                    value: location.rewrite.location,
                                }]
                            },
                        };
                        callback(null, response);
                        return;
                    }
                    if(location.proxy_pass){
                        let parsedUrl = url.parse(location.proxy_pass);
                        let reqClient = parsedUrl.protocol === 'https'? https: http;
                        let headers = {};
                        for(let headName in request.headers){
                            if(headName.toLowerCase() === 'host'){
                                continue;
                            }
                            headers[headName] = request.headers[headName][0].value;
                        }
                        let options = {
                            hostname: parsedUrl.host,
                            port: parsedUrl.protocol === 'https'? 443: 80,
                            path: pathUtil.join(parsedUrl.pathname, oldURI) + (request.querystring?('?'+request.querystring):''),
                            method: method,
                            headers: headers,
                        };
                        reqClient.request(
                            options,
                            (resp) => {
                                let content = '';
                                resp.on('data', (chunk) => { content += chunk; });
                                resp.on('end', () => {
                                    let headers = {};
                                    for(let name in resp.headers){
                                        if(!isForbidHeader(name, lambdaHeaderBlacklist.concat(LambdaViewerRequestReadOnlyHeaders))){
                                            headers[name] = [{'key': name, 'value': resp.headers[name]}];
                                        }
                                    }
                                    const response = {
                                        status: resp.statusCode,
                                        statusDescription: 'OK',
                                        body: content,
                                        headers: headers,
                                    };
                                    callback(null, response);
                                    return;
                                });
                            }
                        ).end();
                    }
                }
            }
        }
    }
};

```

## 总结

1. 要仔细看文档。在 nodejs 的 `http.request()`后要调用 `end()` 方法触发请求和 Lambda 黑名单标头两处我都浪费了很多时间去调试，这是应该可以避免的。
2. 看清本质，不论是 Nginx 还是 Lambda@Edge 或是其他什么 Http 服务器，本质都是处理 HTTP 请求和响应，都遵循 HTTP 协议。
3. AWS Lambda 太好用了！

## 更正（2019-02-26）

重定向和反向代理的逻辑不适合放在同一个 Lambda 函数中。  

实现重定向的 Lambda@Edge 函数部署在 ViewerRequest 可以在更贴近用户的边缘节点计算，达到更快的响应。但部署在 ViewerRequest 的 Lambda 函数有最多 5 秒的超时限制，因此如果反向代理的源站响应时间过长或请求过大，Lambda 就会产生 Timeout 的错误并向客户端返回503。而 OriginRequest 则可以设置最多 30 秒的超时时间。因此把实现反向代理的 Lambda 函数部署在 OriginRequest 也许是个更好的选择，并且可以利用 CloudFront 的缓存功能。  

但是，将同一个站点的不同逻辑配置在两个 Lambda 函数中不利于后续的管理和维护。Lambda@Edge 的使用场景还需要谨慎考虑。