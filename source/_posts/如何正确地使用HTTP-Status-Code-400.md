---
title: 如何正确地使用 HTTP Status Code 400
date: 2021-11-23 19:35:27
permalink: how-to-use-http-status-code-400-correctly/
tags: HTTP
categories: DevOps
---

大家对 HTTP 状态码一定都不陌生，也能随口说出它们的含义，但真的有仔细考虑过具体的使用场景吗？前两天与一位同学在何时使用状态码 400 这个问题上有了不同意见。  
原问题可以细分为如下几个场景：
1. 业务接口缺少必要的参数，是否可以返回 400 ？
2. 业务接口收到的 string 类型的参数，转 int 类型失败时，是否可以返回 400 ？
3. 业务接口的请求体内容在 json 反序列化到 Golang 结构体的过程中失败，是否可以返回 400 ？

读到这里，你的心中是否有了自己的答案？我与朋友一番讨论并搜集资料后，整理总结在此。  

<!--more-->

想要了解如何正确的使用 HTTP Status Code 400，最简单直接的方式就是查它的定义。

### 定义
随着 HTTP 协议不断地发展完善，RFC 中对状态码的定义也发生了一些变化。

#### [RFC 2068](https://www.rfc-editor.org/rfc/rfc2068#page-60) (January 1997)
> 10.4.1 400 Bad Request
>
>   The request could not be understood by the server due to malformed syntax. The client SHOULD NOT repeat the request without modifications.

由于错误的语法格式导致请求不能被服务器理解。客户端不应该不经修改重发请求。


#### [RFC 2616](https://www.rfc-editor.org/rfc/rfc2616#page-65) (June 1999)  
在 1999 年 6 月发布的 RFC 2616 淘汰了 RFC 2068，但其中 400 Bad Request 的定义没有发生任何变化。

#### [RFC 4918](https://www.rfc-editor.org/rfc/rfc4918#page-78) (June 2007)  
RFC 4918 是对 RFC 2616 的扩展补充，新增了状态码 422。
> 11.2.  422 Unprocessable Entity
> 
>    The 422 (Unprocessable Entity) status code means the server understands the content type of the request entity (hence a 415(Unsupported Media Type) status code is inappropriate), and the syntax of the request entity is correct (thus a 400 (Bad Request) status code is inappropriate) but was unable to process the contained instructions.  For example, this error condition may occur if an XML request body contains well-formed (i.e., syntactically correct), but semantically erroneous, XML instructions.

状态码 422 表示服务器理解了请求实体的内容类型（因此 415(不支持的媒体类型) 是不合适的）并且请求的语法也是正确的（因此 400 是不合适的），但是不能处理其中包含的实体。例如，一个 XML 的请求体格式是正确的（语法正确），但是语义是错误的。

#### [RFC 7231](https://www.rfc-editor.org/rfc/rfc7231#page-58) (June 2014)  
在 2014 年 6 月发布的 RFC 7231 淘汰了 RFC 2616，对状态码 400 给出了新的定义。

> 6.5.1.  400 Bad Request
> 
>    The 400 (Bad Request) status code indicates that the server cannot or will not process the request due to something that is perceived to be a client error (e.g., malformed request syntax, invalid request message framing, or deceptive request routing).

状态码 400 表示由于某种客户端的错误（例如错误的请求语法，无效的消息帧或欺骗性请求路由），服务器不能或不处理此请求。

### 小结（第一部分）
从上面的定义中可以看到，无论是 1997 年发布的 RFC 2068 还是在 2014 年重新修订的 RFC 7231 ，对状态码 400 的描述中都有提到 **malformed syntax（错误的语法）**。在 RFC 4918 对状态码 422 的定义中还特别区分了 **syntactically（语法）** 和 **semantically（语义）**。现在我们就可以回答最开始提出的三个问题。

> 1. 业务接口缺少必要的参数，是否可以返回 400 ？
> 2. 业务接口收到的 string 类型的参数，转 int 类型失败时，是否可以返回 400 ？
> 3. 业务接口的请求体内容在 json 反序列化到 Golang 结构体的过程中失败，是否可以返回 400 ？

问题 1 和问题 2 可以一起回答，不能。因为缺少参数和业务逻辑内部的类型转换都不属于语法错误，而是语义错误。对于 HTTP 协议和服务器来说，已经正确的承载、传输、接收了请求内容。  
问题 3 需要再细分讨论。在 json 反序列化的过程中，如果是因为 json 本身格式错误（比如乱码，缺少括号）导致的失败，应该返回状态码 400。如果请求体本身的 json 格式正确符合标准，但由于不能与业务中定义的结构体匹配导致的失败（比如字段类型不一致），则不应该返回状态码 400。

对于以上三种情况，我认为返回状态码 422 都是更合适的选择。

### 补充

但这是一定的吗？  
在 RFC 7231 中对[响应状态码](https://www.rfc-editor.org/rfc/rfc7231#page-58)的介绍如下：
>  The status-code element is a three-digit integer code giving the
   result of the attempt to understand and satisfy the request.  
>  
>   HTTP status codes are extensible.  HTTP clients are not required to understand the meaning of all registered status codes, though such digit, and treat an unrecognized status code as being equivalent to the x00 status code of that class, with the exception that a recipient MUST NOT cache a response with an unrecognized status code.

状态码是3位的整数，HTTP 状态码是可扩充的。HTTP 客户端不被要求理解所有状态码的含义，但一定要能理解每一种类别状态码，即状态码的第一位数字所代表的含义。不能被识别理解的状态码可以把它看做 x00，除非接受者不得缓存不能识别的状态码。

### 小结（第二部分）
由此可知，x00 状态码可以看作是该类别下各种状态的概括。我们已经知道状态码 422 是在 RFC 4918 中进行的扩展补充。当 HTTP 客户端不能识别状态码 422 时也可以把它看作 400。因此，在并不严格的情况下使用 400 代替 422 也不能算是错误。

### 我的思考

前面的讨论是从 HTTP 状态码的定义展开的。从分层的角度来看，HTTP 状态码是协议层的定义，是用来表示 HTTP 请求的结果，与业务层无关。业务层可以在响应体内定义自己的错误码。如果一个 HTTP 请求被正确的接收，但因为业务逻辑缺少参数，那么可以返回 HTTP Status Code 200，并在响应体内带上业务自定义的状态码和错误消息，比如
```
HTTP/1.1 200 OK
{"code": 10001, “msg": "invalid params"}
```

### 概念澄清

在前面状态码 400 的定义中有这么一句话：
> 由于错误的语法格式导致请求不能被服务器理解。

有的同学对此疑惑，我要一个 int 类型的参数，传给我一个 string 类型的参数，我就是不理解呀，因此说参数类型错误的场景符合 400 的定义。这里有两个概念需要明确。

1. 语法（syntactically），区别于语义（semantically）。缺少参数或参数错误都属于语义错误，即表达的内容不对，但格式是正确的。
2. 不能被服务器理解中的”服务器“。这里的服务器应该理解为 web server，比如 Nginx、Apache 或是 Golang 中的 http.Handler。要注意区别于应用程序 application，即我们开发的业务代码。在实际工作中我们常把 server + application 统称为”后端“，因此有同学容易混淆。

### 总结

1. 状态码 400 用于表示客户端的语法错误，服务器读不懂请求。
2. 状态码 422 表示服务器可以正确读取客户端的请求，但因为其内容中的语义错误而不能处理请求。
3. 接口缺少业务参数或参数类型错误时应优先考虑使用 422。
4. 在某些情况下，x00 是对该类状态码的概括，使用 400 代替 422 也并非错误。

### 彩蛋 Hyper Text Coffee Pot Control Protocol (HTCPCP/1.0)

在查阅资料的过程中，发现了 [RFC 2324](https://www.rfc-editor.org/rfc/rfc2324)，一个来自 1998 年愚人节玩笑，超文本咖啡壶控制协议，其中还定义了 `418 I'm a teapot`，哈哈。

### 参考资料：
1. [Should I return an HTTP 400 (Bad Request) status if a parameter is syntactically correct, but violates a business rule?](https://softwareengineering.stackexchange.com/questions/329229/should-i-return-an-http-400-bad-request-status-if-a-parameter-is-syntactically)
2. [http-status-codes-for-invalid-data-400-vs-422](https://www.bennadel.com/blog/2434-http-status-codes-for-invalid-data-400-vs-422.htm)
3. [Should I use HTTP status codes to describe application level events](https://softwareengineering.stackexchange.com/questions/305250/should-i-use-http-status-codes-to-describe-application-level-events)
