title: Bottle Tutorial 官方教程中文翻译
permalink: bottle-tutorial-chinese-translation
date: 2016-02-26 11:40:46
updated: 2016-2-28 11:57:07
tags:
- Bottle
- Python
categories:
- Python
---
> 在学习Bottle的同时，我翻译了这篇 [Bottle官网](http://bottlepy.org/docs/dev/tutorial.html)上的教程，希望能为后来者提供一些便利。我尽自己所能翻译的准确易懂，但因个人水平有限而不可避免的会出现一些错误，希望大家指正。对有能力阅读英文原文的同学我还是推荐去阅读官方的教程。看英文吃力的同学，我也希望你可以把本文与原文对比阅读，以便更好地体会一些专业术语的准确含义。

# Bottle: Python Web Framework
Bottle是一个快速，简单，轻量的 [WSGI](http://www.wsgi.org/) 微型Python web 框架。它只有一个文件模块，不依赖除了 [Python标准库](http://docs.python.org/library/) 之外的任何库。
<!--more-->

* **Routing:** 请求到函数调用的映射，支持整洁和动态的URL。
* **Templates:** 快速且python风格的[内置模板引擎](http://bottlepy.org/docs/dev/tutorial.html#tutorial-templates)，并支持 [mako](http://www.makotemplates.org/), [jinja2](http://jinja.pocoo.org/) 和[cheetah](http://www.cheetahtemplate.org/)。
* **Utilities:** 便捷的获取表单数据，文件上传，cookies，headers和其他HTTP相关的元数据(metadata)。
* **Server:** 内置HTTP开发服务器并支持[paste](http://pythonpaste.org/), [fapws3](https://github.com/william-os4y/fapws3), [bjoern](https://github.com/jonashaag/bjoern), [gae](https://developers.google.com/appengine/), [cherrypy](http://www.cherrypy.org/)或其他任何的 [WSGI](http://www.wsgi.org/) HTTP 服务器。

# 教程
这个教程将向你介绍 Bottle web框架的概念和特性，包含基础和进阶部分。你可以从头到尾的通读一遍，或者留着它当做参考使用。自动生成的 [API文档](http://bottlepy.org/docs/dev/api.html) 也许更能吸引你,它包含更多细节，但不如这篇教程解释的多。大部分常见的问题可以在我们的 [方法收集](http://bottlepy.org/docs/dev/recipes.html) 或者 [常见问题](http://bottlepy.org/docs/dev/faq.html) 中找到答案。如果你需要任何帮助，发 [邮件](mailto:bottlepy@googlegroups.com) 给我们或者访问我们的 [交流频道](http://webchat.freenode.net/?channels=bottlepy)。

## 安装
Bottle 不需要依赖任何其他的库。你只需要下载 [bottle.py](http://bottlepy.org/bottle.py) 到你的工程目录就可以了：
```
$ wget http://bottlepy.org/bottle.py
```
这样你就得到了包含新特性的最新版本。如果你更喜欢一个稳定的开发环境，你应该坚持使用稳定的版本。你可以在 [PyPI](http://pypi.python.org/pypi/bottle) 找到它们，也可以通过 pip(推荐)，easy_install或者你的包管理器来安装：
```
$ sudo pip install bottle              # recommended
$ sudo easy_install bottle             # alternative without pip
$ sudo apt-get install python-bottle   # works for debian, ubuntu, ...
```
无论哪种方式，你都需要使用 Python 2.6 或更新版本(包括 3.2+)来运行 bottle 应用。如果你没有系统范围的权限来安装或者只是简单的不想这么做，可以先创建一个 [virtualenv](http://pypi.python.org/pypi/virtualenv) 虚拟环境：
```
$ virtualenv develop              # Create virtual environment
$ source develop/bin/activate     # Change default python to virtual one
(develop)$ pip install -U bottle  # Install bottle to virtual environment
```
如果你还没有安装 virtualenv 的话，可以这么做：
```
$ wget https://raw.github.com/pypa/virtualenv/master/virtualenv.py
$ python virtualenv.py develop    # Create virtual environment
$ source develop/bin/activate     # Change default python to virtual one
(develop)$ pip install -U bottle  # Install bottle to virtual environment
```

## 快速开始："HELLO WORLD"
这里假设你已经安装好了 Bottle 或者已经把 bottle.py 拷贝到了你的工程目录下。让我们来看一个非常基础的"Hello World"的例子吧：
```
from bottle import route, run

@route('/hello')
def hello():
    return "Hello World!"

run(host='localhost', port=8080, debug=True)
```
就是这样。运行脚本，访问 [http://localhost:8080/hello](http://localhost:8080/hello) 你将会在浏览器中看到 "Hello World!"。它是这么工作的：[`
route()`](http://bottlepy.org/docs/dev/api.html#bottle.route)装饰器会绑定一段代码到一个URL路径上。在这个例子中，我们把`/hello`指向了`hello()`函数。这被称作一个*route*(即装饰器的名字)，这是 bottle 框架中最为重要的一个概念。如果你愿意你可以定义任意数量的路由。每当浏览器请求一个URL，预之关联的函数将会被调用并将返回值发回浏览器。就是这么简单。
最后一行的[`run()`](http://bottlepy.org/docs/dev/api.html#bottle.run) 会启动一个内置的开发服务器。它将运行在`localhost`的`8080`端口上对请求提供服务，知道你输入`Control-c`来停止它。你以后可以使用其他的服务后端，目前来说，这就足够了。它不需要安装，不可思议的简单就可以把你的程序运行起来做本地调试。
在早期的开发过程中，[Debug 模式](http://bottlepy.org/docs/dev/tutorial.html#tutorial-debugging) 是非常有用的，但在公开发布的应用中应该将它关闭。请记住这一点。

### 默认的应用(The Default Application)
为了简单，本教程中大部分的示例都使用模块级(module-level) [`route()`](http://bottlepy.org/docs/dev/api.html#bottle.route) 装饰器来定义路由。这样会将路由(routes)添加到一个全局的“默认应用(default application)”中，它是你第一次调用 [`route()`](http://bottlepy.org/docs/dev/api.html#bottle.route) 时自动创建的一个 [Bottle](http://bottlepy.org/docs/dev/api.html#bottle.Bottle) 的实例。其他的几个模块级(module-level)装饰器和方法也关联到的这个默认应用上。如果你想用一种更加面向对象的方式且不介意多写一点代码，你也可以创建独立的应用(application)而不是使用全局的一个：
```
from bottle import Bottle, run

app = Bottle()

@app.route('/hello')
def hello():
    return "Hello World!"

run(app, host='localhost', port=8080)
```
面向对象的方式在 [默认应用(Default Application)](http://bottlepy.org/docs/dev/tutorial.html#default-app) 部分有更加深入的讲解。你只需要知道你有这么一种选择。

## 请求路由(Request Routing)
上一小节中我们只用了一个路由创建了一个非常简单的 web 应用。这里再一次列出 "Hello World"例子中的路由部分：
```
@route('/hello')
def hello():
    return "Hello World!"
```
[`route()`](http://bottlepy.org/docs/dev/api.html#bottle.route) 装饰器连接一个URL来回调函数并添加一个新的路由(route)到 [默认应用(Default Application)](http://bottlepy.org/docs/dev/tutorial.html#default-app)。然而，只有一个路由(route)的程序看起来有些乏味。我们再添加一些吧(别忘了`from bottle import template`)：
```
@route('/')
@route('/hello/<name>')
def greet(name='Stranger'):
    return template('Hello {{name}}, how are you?', name=name)
```
这个例子说明了两件事情：你可以对一个函数绑定不止一个路由(route)，你可以在URL中使用通配符并通过关键字参数来使用它们。

### 动态路由(Dynamic Routes)
包含通配符的路由(route)称作动态路由(相对于静态路由)，它可以匹配不止一个URL。一个简单的通配符(e.g. `<name>`)由包含在尖括号中直到下一个斜杠`/`前的一个或多个字符构成。例如，`/hello/<name>`既可以匹配请求`/hello/alice`，也可以匹配请求`/hello/bob`，但是不能匹配`/hello`,`/hello/`或`hello/mr/smith`。

URL中被通配符匹配的部分可以作为一个关键字参数传递到请求调用(request callback)中。这样你可以轻松的实现使用优雅且有意义的URL的RESTful。这儿还有一些其他的例子：
```
@route('/wiki/<pagename>')            # matches /wiki/Learning_Python
def show_wiki_page(pagename):
    ...

@route('/<action>/<user>')            # matches /follow/defnull
def user_api(action, user):
```
过滤器(filters)可以用来定义更加特殊的通配符，或者用来在匹配部分传递到请求调用之前转换它们。使用`<name:filter>`或`<name:filter:config>`来定义过滤器通配符。config部分是可选的，这取决于所使用的过滤器。
下面的是几个过滤器的默认实现，也可以添加更多：
* **:int** 仅匹配数字(含符号)并将值转换为整型integer。
* **:float** 与 :int 相似但匹配小数。
* **:path** 非贪婪的方式匹配包含斜杠的所有字符，可以用来匹配不止一个路径的分割部分。
* **:re** 允许你在config域中指定自定义的正则表达式。匹配到的值不会被修改。

我们来看一些实际的例子：
```
@route('/object/<id:int>')
def callback(id):
    assert isinstance(id, int)

@route('/show/<name:re:[a-z]+>')
def callback(name):
    assert name.isalpha()

@route('/static/<path:path>')
def callback(path):
    return static_file(path, ...)
```
你也可以添加自己的过滤器。详情查看 [请求路由(Request Routing)](http://bottlepy.org/docs/dev/routing.html)

### HTTP请求方法(HTTP Request Methods)
HTTP协议定义了几种[请求方法](http://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html)来完成不同的任务。如果没有指定其他的方法，所有路由(route)默认为GET方法，这样只会匹配 Get 请求。如果要处理其他请求方法如 POST,PUT,DELETE 或 PATCH，需要给 [`route()`](http://bottlepy.org/docs/dev/api.html#bottle.route) 装饰器添加一个`method`参数或者使用下面五种装饰器中的一个： [`get()`](http://bottlepy.org/docs/dev/api.html#bottle.get), [`post()`](http://bottlepy.org/docs/dev/api.html#bottle.post), [`put()`](http://bottlepy.org/docs/dev/api.html#bottle.put), [`delete()`](http://bottlepy.org/docs/dev/api.html#bottle.delete), [`patch()`](http://bottlepy.org/docs/dev/api.html#bottle.patch)。

POST方法通常用于HTML表单提交。这个例子展示了如何处理一个使用POST方法的登陆表单：
```
from bottle import get, post, request # or route

@get('/login') # or @route('/login')
def login():
    return '''
        <form action="/login" method="post">
            Username: <input name="username" type="text" />
            Password: <input name="password" type="password" />
            <input value="Login" type="submit" />
        </form>
    '''

@post('/login') # or @route('/login', method='POST')
def do_login():
    username = request.forms.get('username')
    password = request.forms.get('password')
    if check_login(username, password):
        return "<p>Your login information was correct.</p>"
    else:
        return "<p>Login failed.</p>"
```
这个例子中`/login`URL连接到两个不同的回调函数，一个处理GET请求，一个处理POST请求。前者向用户展示一个HTML表单，后者由用户提交表单是调用并校验用户在表单中输入的登陆凭证。关于 **Request.forms** 的用法在 [请求数据(Request Data )](http://bottlepy.org/docs/dev/tutorial.html#tutorial-request) 部分有更详细的介绍。

#### 特殊的方法：HEAD 和 ANY
HEAD方法用来请求与GET方法一致的响应，但是没有响应中的body部分。这对于索引资源的元信息(meta-information)非常有用，因为你不必去下载整个文档。Bottle 将会自动的处理这些请求，它会调用同GET方法一致的路由(route)并删掉了请求的body部分(如果有的话)。你不需要自己去指定如何的HEAD routes。

此外，非标准的 ANY 方法作为低优先级的后备方法使用：仅仅在没有定义其他路由(route)时，监听 ANY 方法的routes会忽略HTTP的方法而匹配请求。这对代理路由(proxy-routes)重定向请求到其他指定的子应用非常有用。

总的来说；HEAD请求没有对应route时将落回到(fall back) GET routes，所有的请求落回到(fall back) ANY routes, 但仅仅是没有处理原始请求指定的方法的route时会这样。

### 路由静态文件(Routing Static Files)
类似图片或是CSS等静态文件不会自动的被提供服务。你必须创建一个route和一个回调函数来控制对哪些文件提供服务并且在哪里找到它们：
```
from bottle import static_file
@route('/static/<filename>')
def server_static(filename):
    return static_file(filename, root='/path/to/your/static/files')
```
**static_file()** 函数可以以安全且方便的方式来提供文件服务(查看：[静态文件](http://bottlepy.org/docs/dev/tutorial.html#tutorial-static-files))。这个例子限制了仅在`/path/to/your/static/files`目录中的文件，因为`<filename>`不会匹配包含斜杠的路径。要提供子文件夹中的文件服务，将通配符改用 path 过滤器：
```
@route('/static/<filepath:path>')
def server_static(filepath):
    return static_file(filepath, root='/path/to/your/static/files')
```
指定类似`root='./static/files'`的相对根路径时要小心。工作目录(working directory (`./`))和项目目录并不总是相同。

### 错误页(Error Pages)
如果有什么错误发生，Bottle会显示一个包含大量信息的错误页面。你可以使用 [`error()`](http://bottlepy.org/docs/dev/api.html#bottle.error) 装饰器来重载指定 HTTP 状态码的默认页面：
```
from bottle import error
@error(404)
def error404(error):
    return 'Nothing here, sorry'
```
这样，404文件找不到错误发生时将会向用户显示一个自定义的错误页面。传递到错误处理中唯一的参数是 [HTTPError](http://bottlepy.org/docs/dev/api.html#bottle.HTTPError) 的一个实例。除此之外，错误处理器与一个常规的请求调用十分相似。你可以从`[request](http://bottlepy.org/docs/dev/api.html#bottle.request)`中读取数据，向`[response](http://bottlepy.org/docs/dev/api.html#bottle.response)`中写入数据并返回除了 [HTTPError](http://bottlepy.org/docs/dev/api.html#bottle.HTTPError) 以外任何支持类型的数据。

错误处理器仅仅使用在你的应用返回或抛出了 [HTTPError](http://bottlepy.org/docs/dev/api.html#bottle.HTTPError) 异常(**abort()**就会那么做)的情况下。改变 **Request.status** 或返回 [HTTPResponse](http://bottlepy.org/docs/dev/api.html#bottle.HTTPResponse) 不会触发错误处理。

## 生成内容(Generating content)
在纯粹的WSGI中，你可以从应用中返回的数据类型是非常有限的。应用程序必须返回可迭代(iterable)的字节串。你可以返回一个字符串（因为字符串是可迭代的），但这样会导致大部分服务器一个字符一个字符的传输你的内容。Unicode字符串将不能使用。这是非常不实用的。

Bottle 就更加灵活，支持更多种类的数据格式。它甚至添加了一个 `Content-Length` header,如果可能，它会自动编码 Unicode 字符。下面是你可以从应用程序中返回的数据类型列表及简介：
**Dictionaries**
    如上面提到的，Python 字典(或它的子类)可以被自动的转换成 JSON 字符串并带着`Content-Type`设置为`application/json`的头部返回到浏览器。这使得实现基于 JSON 的 API 将会非常简单。除了 JSON 以外的其他格式也是支持的。查看 *tutorial-output-filter* 来了解更多。
**空字符串，`False`,`None`或其他非真值(non-true values)**
    这些将产生空的输出，`Content-Length`头部被设置为 0。
**Unicode字符串**
    对Unicode字符串会自动的使用`Content-Type`header中指定的编码类型(默认为utf8)来编码,然后把它当做不同的自己字节串(byte strings)(见下面)。
**Byte strings**
    Bottle返回字符串是把它作为一个整体(而不是迭代每一个字符)并根据字符串的长度添加`Content-Length`header。字符串列表会先把连接(joined)起来。其他可迭代产生字符串对象(iterables yielding byte strings)不会被连接，因为它们也许会很大而不适合放入内存中。在这种情况下，`Content-Length`header将不会被设置。
**[HTTPError](http://bottlepy.org/docs/dev/api.html#bottle.HTTPError)或[HTTPResponse](http://bottlepy.org/docs/dev/api.html#bottle.HTTPResponse)的实例**
    返回它们和把它们作为异常抛出具有同样的效果。返回 [HTTPError](http://bottlepy.org/docs/dev/api.html#bottle.HTTPError) 时，错误处理器将会被应用。查看 [Error Pages](http://bottlepy.org/docs/dev/tutorial.html#tutorial-errorhandling) 获取更详细的信息。
**文件对象(File objects)**
    所有拥有`.read()`方法的对象将被看做是一个文件或类文件对象，并被传递到 WSGI 服务器框架定义的 `wsgi.file_wrapper`。一些 WSGI 服务器会使用最优的系统调用来高效的传输文件。其他情况下，这将仅仅是不断的把数据库放入内存。可选的头部例如`Content-Length`或`Content-Type`不会被自动的设置。如果可能，使用**send_file()**。查看[Static Files](http://bottlepy.org/docs/dev/tutorial.html#tutorial-static-files) 获取更多信息。
**可迭代对象和生成器(Iterables and generators)**
    你可以在请求的调用中使用`yield`或者返回一个可迭代对象(只要它返回的是 byte strings, unicode strings, [HTTPError](http://bottlepy.org/docs/dev/api.html#bottle.HTTPError)或[HTTPResponse](http://bottlepy.org/docs/dev/api.html#bottle.HTTPResponse))。很遗憾，嵌套的迭代对象不被支持。请注意，HTTP状态码在可迭代对象产生第一个非空白值(non-empty)时将被返回。之后的变化没有反应。
上面这个列表的顺序是有意义的。例如你返回有个拥有`read()`方法的 [str](http://docs.python.org/library/functions.html#str) 子类，它将仍然被看做是strings而不是file，因为strings首选被处理。

#### 改变默认编码(Changing the Default Encoding)
Bottle使用`Content-Type`header中的 *charset* 参数来决定如何编码Unicode字符串。这个header默认为`text/html; charset=UTF8`，可以使用 **Response.content_type** 属性来改变它，或者直接使用 **Response.charset** 属性。(关于 [Response](http://bottlepy.org/docs/dev/api.html#bottle.Response))对象的详细介绍在 [The Response Object](http://bottlepy.org/docs/dev/tutorial.html#tutorial-response)部分。
```
from bottle import response
@route('/iso')
def get_iso():
    response.charset = 'ISO-8859-15'
    return u'This will be sent with ISO-8859-15 encoding.'

@route('/latin9')
def get_latin():
    response.content_type = 'text/html; charset=latin9'
    return u'ISO-8859-15 is also known as latin9.'
```
在某些少见的情况下，Python编码的名字不同于HTTP规范中提供的名字。如果是这种情况，那你需要做两件事：首先设置 **Response.content_type** header(保持不变的发送的客户端)，然后设置 **Response.charset** 属性(用来编码Unicode)。

### 静态文件(Static Files)
你可以直接返回文件对象(file objects)，但使用 **static_file()** 是用来提供文件服务的推荐方式。它会自动的猜测一个mime类型(mime-type)，添加一个`Last-Modified` header，为了安全因素限制路径到`root`目录并发出适当的错误响应(权限错误时发出403，找不到文件时发出404)。它甚至会提供`If-Modified-Since`header并且会生成一个`304 Not Modified`响应。如果你不希望它自动猜测MIME类型，可以手动的设置。
```
from bottle import static_file
@route('/images/<filename:re:.*\.png>')
def send_image(filename):
    return static_file(filename, root='/path/to/image/files', mimetype='image/png')

@route('/static/<filename:path>')
def send_static(filename):
    return static_file(filename, root='/path/to/static/files')
```
如果真的需要，你还可以把 **static_file()** 的返回值作为异常抛出。

#### 强制下载(Forced Download)
当MIME类型已知且有了关联的应用程序时，大部分浏览器会尝试打开下载的文件(例如PDF文件)。如果你不想这样，你可以强制弹出一个下载对话框并指定一个建议的文件名：
```
@route('/download/<filename:path>')
def download(filename):
    return static_file(filename, root='/path/to/static/files', download=filename)
```
如果`download`参数是`True`，会使用文件原来的名称。

### HTTP错误和重定向(HTTP Errors and Redirects)
`abort()` 函数是产生HTTP错误页的便捷方法。
```
from bottle import route, abort
@route('/restricted')
def restricted():
    abort(401, "Sorry, access denied.")
```
要将客户端重定向到另一个URL，你可以发送一个带有设置到新URL的`Location`header的`303 See Other` 响应。**redirect()**方法会替你那样做：
```
from bottle import redirect
@route('/wrong/url')
def wrong():
    redirect("/right/url")
```
你可以提供另一个HTTP状态码作为第二个参数。
> 注意：
> 两个函数都会通过抛出 [HTTPError](http://bottlepy.org/docs/dev/api.html#bottle.HTTPError) 异常来终止你的回调代码。

#### 其他异常(Other Exceptions)
除了 [HTTPResponse](http://bottlepy.org/docs/dev/api.html#bottle.HTTPResponse)和[HTTPError](http://bottlepy.org/docs/dev/api.html#bottle.HTTPError) ，其他异常均会返回一个 `` 响应，因此它们不会导致你的WSGI服务器崩溃。你可以在你的中间件(middleware)中将`bottle.app().catchall`设置为`False`来关闭这种处理异常的行为。

### [Response](http://bottlepy.org/docs/dev/api.html#bottle.Response)对象(The Response Object)
HTTP状态码，响应头部(response headers)和cookies等响应元数据(metadata)会恰到好处的存放在一个叫做 [response](http://bottlepy.org/docs/dev/api.html#bottle.response) 的对象中并传输的浏览器。你可以直接操作这些元数据(metadata)或使用一些预先定义好的函数。完整的API和特性列表在 API 部分中有介绍(see [Response](http://bottlepy.org/docs/dev/api.html#bottle.Response))，但大部分常见的情况和特点也会在这介绍。

#### Status Code
[HTTP状态码](http://bottlepy.org/docs/dev/http_code)用于控制浏览器的行为，它默认为`200 OK`。在大部分情况下，你不需要手动的去设置 **Response.status** 属性，但使用 **abort()** 方法或返回 [HTTPResponse](http://bottlepy.org/docs/dev/api.html#bottle.HTTPResponse) 实例时需要一个适当的状态码。任意的整数都可以做状态码，但 [HTTP规范](http://bottlepy.org/docs/dev/http_code) 中没有的状态码将会使浏览器迷惑，而且这样也不够标准。

#### Response Header
`Cache-Control`或`Location`等响应头部通过 **Response.set_header()** 来定义。这个方法需要两个参数，头部的名称和值。名称是大小写敏感的：
```
@route('/wiki/<page>')
def wiki(page):
    response.set_header('Content-Language', 'en')
    ...
```
大部分头部都是独一无二的，就是说每个名称作为一个头部发送到客户端。然而也有些头部允许在响应中出现多次。添加一个附加的头部，使用 **Response.add_header()** 而不是 **Response.set_header()** ：
```
response.set_header('Set-Cookie', 'name=value')
response.add_header('Set-Cookie', 'name2=value2')
```
请注意，这只是个例子。如果你需要使用cookies，[往下看](http://bottlepy.org/docs/dev/tutorial.html#tutorial-cookies)。

### Cookies
cookie是保存在用户浏览器中的一些文本。你可以通过 **Request.get_cookie()** 来访问之前定义过的cookies，使用 **Response.set_cookie()** 来设置新的cookies：
```
@route('/hello')
def hello_again():
    if request.get_cookie("visited"):
        return "Welcome back! Nice to see you again"
    else:
        response.set_cookie("visited", "yes")
        return "Hello there! Nice to meet you"
```
**Response.set_cookie()**方法接受一些参数用于控制cookies的生命周期和行为。一些常用的设置如下：
* **max_age:** 最大存活时间，单位是秒。默认为`None`。
* **expires:** 一个datetime对象或UNIX时间戳。默认为`None`。
* **domain:** 允许读取此cookie的域名。默认为当前域名。
* **path:** 限制cookie到指定的路径。默认为`/`。
* **secure:** 限制cookie使用HTTPS连接。默认为off。
* **httponly:** 阻止客户端使用JavaScript来读取此cookie。默认为off，Python 2.6及更新要求必填。

如果 *expires* 和 *max_age* 都没有设置，那么cookie将会在浏览器会话(session)结束或浏览器关闭时过期。使用cookies时你需要考虑一下几点：
* 大部分浏览器中限制cookie不能超过4KB。
* 一些用户把他们的浏览器设置为不接受cookies。大部分搜索引擎也会忽略cookies。确保你的程序在没有cookies也可以正常工作。
* cookies被存储在客户端并且没有任何的加密。不管你在cookie中存了什么内容，用户都可以读取它。更糟的是，攻击者可以通过你的 [XSS](http://en.wikipedia.org/wiki/HTTP_cookie#Cookie_theft_and_session_hijacking) 漏洞来窃取用户的cookies。
* cookies可以很容易的被恶意用户伪造，不要相信cookies。

#### 签名的cookies(Signed Cookies)
正如上面提到的，cookies很容易被恶意的用户篡改。Bottle可以对你的cookies签名来阻止篡改。你需要做的只是在读取或设置cookies时通过 *secret* 参数传入签名密钥并保证它不被泄露。这样的话，如果cookie没有签名或者签名密钥不匹配， **Request.get_cookie()** 将会返回`None`：
```
@route('/login')
def do_login():
    username = request.forms.get('username')
    password = request.forms.get('password')
    if check_login(username, password):
        response.set_cookie("account", username, secret='some-secret-key')
        return template("<p>Welcome {{name}}! You are now logged in.</p>", name=username)
    else:
        return "<p>Login failed.</p>"

@route('/restricted')
def restricted_area():
    username = request.get_cookie("account", secret='some-secret-key')
    if username:
        return template("Hello {{name}}. Welcome back.", name=username)
    else:
        return "You are not logged in. Access denied."
```
另外，Bottle会自动的序列化(pickles)和反序列化(unpickles)保存在签名cookies中数据。这样你就可以在cookies中存储可以序列化(pickle-able)的任意类型数据，只要序列化后的数据不会超过4KB的限制。
> 警告：
> 签名的cookies并没有被加密(客户端仍然可以看到其内容)，也没有防止拷贝(客户端仍然可以恢复一个旧的cookie)。签名的主要目的保证序列化及反序列化的安全并防止篡改，而不是在客户端保存秘密的信息。

## 请求数据(Request Data)
cookies, HTTP header, HTML `<form>`域和其他的请求数据都可以从全局的 [request](http://bottlepy.org/docs/dev/api.html#bottle.request) 对象中获取。这个特殊的对象总是来自于当前的请求，甚至是在多个客户端连接的多线程环境中：
```
from bottle import request, route, template

@route('/hello')
def hello():
    name = request.cookies.username or 'Guest'
    return template('Hello {{name}}', name=name)
```
[request](http://bottlepy.org/docs/dev/api.html#bottle.request) 对象是 [BaseRequest](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest) 的一个子类 ，它拥有丰富的API来访问数据。我们在这里仅介绍常用的特性，对于初学者这足够了。

### [FormsDict](http://bottlepy.org/docs/dev/api.html#bottle.FormsDict)介绍(Introducing FormsDict)
Bottle使用一种特殊类型的字典来保存表单数据和cookies。[FormsDict](http://bottlepy.org/docs/dev/api.html#bottle.FormsDict) 和普通的dictionary一样，不同的时有一些新的特性帮助你更简单的完成任务。
**Attribute access:** 字典中所有的值都可以作为属性来访问。这些虚拟属性返回Unicode字符串，即使找不到值或者Unicode解码失败。如果是那样，返回的字符串是空的，但是仍然存在：
```
name = request.cookies.name

# is a shortcut for:

name = request.cookies.getunicode('name') # encoding='utf-8' (default)

# which basically does this:

try:
    name = request.cookies.get('name', '').decode('utf-8')
except UnicodeError:
    name = u''
```
**Multiple values per key:** [Formsdict](http://bottlepy.org/docs/dev/api.html#bottle.FormsDict) 是 [MultiDict](http://bottlepy.org/docs/dev/api.html#bottle.MultiDict) 的子类，每个key可以对应存储多个值。标准的字段取值方法只会返回一个值，但是 [getall()](http://bottlepy.org/docs/dev/api.html#bottle.MultiDict.getall) 方法会返回指定key的所有值的列表(可能是空的)：
```
for choice in request.forms.getall('multiple_choice'):
    do_something(choice)
```
**WTForms support:** 有些库(例如：[WTForms](http://wtforms.simplecodes.com/))希望输入是全Unicode编码(all-unicode)的字典。[FormsDict.decode()](http://bottlepy.org/docs/dev/api.html#bottle.FormsDict.decode)会替你那么做的。它解码所有的值并返回自身的一份拷贝，同时保留一键多值和所有其他的特性。
> 注意：
> 在Python 2 中所有的keys和values都是字节字符串(byte-strings)。如果需要解码，你可以调用 [FormsDict.getunicode()](http://bottlepy.org/docs/dev/api.html#bottle.FormsDict.getunicode) 或者通过属性来获取值。两种方法都会尝试解码字符串(默认使用utf8)，如果失败会返回空的字符串。不需要捕获 *UnicodeError*：
```
>>> request.query['city']
'G\xc3\xb6ttingen'  # A utf8 byte string
>>> request.query.city
u'Göttingen'        # The same string as unicode
```
> 在Python 3 中所有的字符串都是Unicode编码的，但HTTP协议是基于字节的(byte-based)。服务器必须在把字节字符串(byte strings)传递到应用之前已某种方式解码。从安全的角度考虑，WSGI建议使用 ISO-8859-1 (aka latin1)，一种可逆的单字节编码，它可以在后面用其他编码再次编码。在Bottle中， [FormsDict.getunicode()](http://bottlepy.org/docs/dev/api.html#bottle.FormsDict.getunicode) 和 属性取值的方法可以做到这一点，但字典的取值方法不行。它们返回由服务器提供的未经改变的值，也许不是你想要的：
```
>>> request.query['city']
'GÃ¶ttingen' # An utf8 string provisionally decoded as ISO-8859-1 by the server
>>> request.query.city
'Göttingen'  # The same string correctly re-encoded as utf8 by bottle
```
> 如果你需要正确的编码整个字典的值(例如：[WTForms](http://wtforms.simplecodes.com/))，你可以使用 [FormsDict.decode()](http://bottlepy.org/docs/dev/api.html#bottle.FormsDict.decode) 来获取一份二次编码的拷贝。

### Cookies
cookies 是一些保存在客户端浏览器的小段文本，并会通过每次请求返回到服务器。cookies非常有用，它们可以保存比一次请求更多的状态(HTTP本身是没有状态的)，但不应该用来解决安全相关的问题。它们可以在客户端被轻易的伪造。

所有的cookies都从客户端发送并且可以通过 [BaseRequest.cookies](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.cookies) (一个[FormsDict](http://bottlepy.org/docs/dev/api.html#bottle.FormsDict))来获取。下面的例子演示了一个简单的基于cookie的访问计数：
```
from bottle import route, request, response
@route('/counter')
def counter():
    count = int( request.cookies.get('counter', '0') )
    count += 1
    response.set_cookie('counter', str(count))
    return 'You visited this page %d times' % count
```
[BaseRequest.get_cookie()](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.get_cookie)不同于直接访问cookies。它支持把签名的cookies作为独立的一部分来解码。

### HTTP头部(HTTP Headers)
所有来自客户端发出的HTTP headers(例如：`Referer`,`Agent`或是`Accept-Language`)都被存储在一个 [WSGIHeaderDict](http://bottlepy.org/docs/dev/api.html#bottle.WSGIHeaderDict) 中 并且可以通过 [BaseRequest.headers](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.headers) 属性来访问。 [WSGIHeaderDict](http://bottlepy.org/docs/dev/api.html#bottle.WSGIHeaderDict) 是一个键值大小写敏感的基本字典：
```
from bottle import route, request
@route('/is_ajax')
def is_ajax():
    if request.headers.get('X-Requested-With') == 'XMLHttpRequest':
        return 'This is an AJAX request'
    else:
        return 'This is a normal request'
```

### 查询变量(Query Variables)
查询字符串(例如`/forum?id=1&page=5`)常用于传输少量的键值对到服务器。你可以使用 [BaseRequest.query](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.query) 属性(一个[FormsDict](http://bottlepy.org/docs/dev/api.html#bottle.FormsDict))来访问这些值并通过 [BaseRequest.query_string](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.query_string) 属性来获取整个字符串。
```
from bottle import route, request, response, template
@route('/forum')
def display_forum():
    forum_id = request.query.id
    page = request.query.page or '1'
    return template('Forum ID: {{id}} (page {{page}})', id=forum_id, page=page)
```

### HTML表单处理(HTML `<form>` Handling)
我们从最基本的开始。在HTML中，一个典型的`<form>`是这样的：
```
<form action="/login" method="post">
    Username: <input name="username" type="text" />
    Password: <input name="password" type="password" />
    <input value="Login" type="submit" />
</form>
```
`action`属性指定的URL会接收到表单的数据。`method`指定使用HTTP的哪种方法(GET或POST)。使用`method="get"`时，表单中的值将会附加在URL后面并可以通过上面提到的方法 [BaseRequest.query](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.query) 来访问。考虑到这样是不安全的并且有其他的一些限制，我们这里使用`method="POST"`。如果不确定，请使用`POST`形式。

通过`POST`方式传输的表单数据作为一个 [FormsDict](http://bottlepy.org/docs/dev/api.html#bottle.FormsDict) 被储存在 [BaseRequest.forms](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.forms) 中。服务器端的代码应该是这样的：
```
from bottle import route, request

@route('/login')
def login():
    return '''
        <form action="/login" method="post">
            Username: <input name="username" type="text" />
            Password: <input name="password" type="password" />
            <input value="Login" type="submit" />
        </form>
    '''

@route('/login', method='POST')
def do_login():
    username = request.forms.get('username')
    password = request.forms.get('password')
    if check_login(username, password):
        return "<p>Your login information was correct.</p>"
    else:
        return "<p>Login failed.</p>"
```
还一些用来获取表单数据的其他属性。其中一些为了访问方便从不同的地方获取值并组合起来。下面的表格会给你一个清晰的概览。

| **Attribute** | **GET Form fields** | **POST Form fields** | **File Uploads** |
|--------|--------|--------|--------|
| [BaseRequest.query](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.query) | yes | no | no |
| [BaseRequest.forms](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.forms) | no | yes | no |
| [BaseRequest.files](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.files) | no | no | yes |
| [BaseRequest.params](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.params) | yes | yes | no |
| [BaseRequest.GET](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.GET) | yes | no | no |
| [BaseRequest.POST](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.POST) | no | yes | yes |

### 文件上传(File Uploads)
为了支持文件上传，我们必须对`<form>`标记做一点小小的改变。首先，我们要在`<form>`表单中添加属性`enctype="multipart/form-data"`，告诉浏览器使用一种不同的方法来编码表单数据。然后我们添加`<input type="file" />`标记让用户来选择一个文件。这是一个例子：
```
<form action="/upload" method="post" enctype="multipart/form-data">
  Category:      <input type="text" name="category" />
  Select a file: <input type="file" name="upload" />
  <input type="submit" value="Start upload" />
</form>
```
Bottle 把上传文件作为 [FileUpload](http://bottlepy.org/docs/dev/api.html#bottle.FileUpload) 的一个实例保存在 [BaseRequest.files](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.files) 中，同时还包含一些关于上传文件的其他元信息(metadata)。假设你想把文件保存到磁盘：
```
@route('/upload', method='POST')
def do_upload():
    category   = request.forms.get('category')
    upload     = request.files.get('upload')
    name, ext = os.path.splitext(upload.filename)
    if ext not in ('.png','.jpg','.jpeg'):
        return 'File extension not allowed.'

    save_path = get_save_path_for_category(category)
    upload.save(save_path) # appends upload.filename automatically
    return 'OK'
```
[FileUpload.filename](http://bottlepy.org/docs/dev/api.html#bottle.FileUpload.filename)包含客户端文件系统上文件的名字，但是已经被格式化到规范的格式以避免文件名中包含特殊字符或是路径分隔符而引起错误。如果你需要未被修改过的文件名，可以查看 [FileUpload.raw_filename](http://bottlepy.org/docs/dev/api.html#bottle.FileUpload.raw_filename)。

如果你要保存文件到磁盘，强烈建议使用 [FileUpload.save](http://bottlepy.org/docs/dev/api.html#bottle.FileUpload.save) 方法。它可以避免一些常见的错误(例如：除非你明确的指定，否则它不会重写已经存在的文件)并且可以高效地把文件保存的内存。你也可以通过 [FileUpload.file](http://bottlepy.org/docs/dev/api.html#bottle.FileUpload.file) 直接访问文件对象。小心点就行了。

### JSON内容(JSON Content)
一些JavaScript或是REST客户端会发送`application/json`内容到服务器。[BaseRequest.json](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.json) 属性包含转换过格式的数据，如果的它是有效的话。

### 原始的请求体(The raw request body)
你可以通过 [BaseRequest.body](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.body) 把原始的body数据作为类文件对象(file-like object)访问。取决于内容的长度和 [BaseRequest.MEMFILE_MAX](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.MEMFILE_MAX) 的设置，它可能是 **BytesIO**缓冲区或是一个临时文件。无论是哪一种情况，body都会在你访问属性之前完整缓存下来。如果你预期有大量的数据并且想要直接访问未缓存的流(stream)，看看`request['wsgi.input']`吧。

### WSGI环境(WSGI Environment)
每个 [BaseRequest](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest) 实例都包含一个 WSGI 环境字典。原始的内容保存在 [BaseRequest.environ](http://bottlepy.org/docs/dev/api.html#bottle.BaseRequest.environ) ，但是请求对象本身也表现为一个字典。大部分有用的数据会通过特定的方法或属性暴露出来，但是如果你想要直接访问 [WSGI 环境变量](http://bottlepy.org/docs/dev/WSGIspecification) ， 你可以这么做：
```
@route('/my_ip')
def show_ip():
    ip = request.environ.get('REMOTE_ADDR')
    # or ip = request.get('REMOTE_ADDR')
    # or ip = request['REMOTE_ADDR']
    return template("Your IP is: {{ip}}", ip=ip)
```

## 模板(Templates)
Bottle 带有一个快速且强大的内置模板引擎叫做 [SimpleTemplate Engine](http://bottlepy.org/docs/dev/stpl.html)。你可以使用 [template(http://bottlepy.org/docs/dev/api.html#bottle.template)]() 函数或 [view(http://bottlepy.org/docs/dev/api.html#bottle.view)]() 装饰器来渲染模板。你需要做的只是提供模板的名字并通过关键字参数传递你需要传递到模板上的数据。这有一个简单的例子演示如何渲染一个模板：
```
@route('/hello')
@route('/hello/<name>')
def hello(name='World'):
    return template('hello_template', name=name)
```
这会加载模板文件`hello_template.tpl`并用设置的`name`变量来渲染它。Bottle会在`./views/`文件夹或在`bottle.TEMPLATE_PATH`中指定的文件夹列表中查找模板。

[view()](http://bottlepy.org/docs/dev/api.html#bottle.view)装饰器允许你返回一个字典到模板变量而不是调用 [template()](http://bottlepy.org/docs/dev/api.html#bottle.template)：
```
@route('/hello')
@route('/hello/<name>')
@view('hello_template')
def hello(name='World'):
    return dict(name=name)
```
### 语法(Syntax)
模板的语法知识Python语音上薄薄的一层(a very thin layer around the Python language)。它的主要目的是确保正确的排版，因此你可以格式化的你模板而不用担心排版问题。下面的连接是完整的原发介绍：[SimpleTemplate Engine](http://bottlepy.org/docs/dev/stpl.html)

这有一个简单的模板示例：
```
%if name == 'World':
    <h1>Hello {{name}}!</h1>
    <p>This is a test.</p>
%else:
    <h1>Hello {{name.title()}}!</h1>
    <p>How are you?</p>
%end
```

### 缓存(Caching)
编译后模板会被缓存在内存中。模板的改变不会显示出来直到你清空模板缓存。调用`bottle.TEMPLATES.clear()`来清空模板缓存。缓存在debug模式下是禁用的。

## 插件(Plugins)
*0.9版的新特性*
Bottle的核心功能包含了大部分的使用情况，但是作为一个微型框架，它也有局限性。这就是插件要来做的了。插件添加框架中没有的功能，集成一切第三方库或是仅仅用来自动化一些重复的工作。
我们有一个不断增长的[可用插件列表](http://bottlepy.org/docs/dev/plugins/index.html)，大部分的插件都被设计为轻便且可复用的。你的问题很有很大的可能已经被解决了并且有可用的插件。如果没有，[插件开发指引](http://bottlepy.org/docs/dev/plugindev.html)可以帮助你。

插件的作用和API多种多样，这取决于特定的插件。以`SQLitePlugin`为例，它检查一个包含`db`参数的调用并在每次调用发生时创建一个新的数据库连接。这使得使用数据库变得非常方便：
```
from bottle import route, install, template
from bottle_sqlite import SQLitePlugin

install(SQLitePlugin(dbfile='/tmp/test.db'))

@route('/show/<post_id:int>')
def show(db, post_id):
    c = db.execute('SELECT title, content FROM posts WHERE id = ?', (post_id,))
    row = c.fetchone()
    return template('show_post', title=row['title'], text=row['content'])

@route('/contact')
def contact_page():
    ''' This callback does not need a db connection. Because the 'db'
        keyword argument is missing, the sqlite plugin ignores this callback
        completely. '''
    return template('contact')
```
其他插件可能位于线程安全的**本地(local)**对象中，用于改变 [request](http://bottlepy.org/docs/dev/api.html#bottle.request) 对象，过滤调用返回的数据或是完全的跳过调用。以一个“认证”插件为例，它会检查session并返回一个登陆页面而不是调用原始的回调方法。具体会做什么，这取决于插件。

### 应用范围的安装(Application-wide Installation)
插件可以安装到整个应用范围或是仅仅添加到某些指定的路由(routes)以添加更多功能。大部分插件可以被安全的安装在所有routes上并且他们足够的智能，不会过度的调用不需要的方法。

我们以`SQLitePlugin`举例。它仅仅影响需要数据库连接的路由调用(route callbacks)。正因如此，我们可以安装应用范围的插件而不会受到过度的影响。

安装插件，只需要调用 **install()**，把插件作为它的第一个参数：
```
from bottle_sqlite import SQLitePlugin
install(SQLitePlugin(dbfile='/tmp/test.db'))
```
现在插件还不会应用到请求调用(route callbacks)。它会延迟以确保没有遗漏的routes。如果需要，你可以先安装插件然后添加routes。但是安装插件是有顺序要求的。如果一个插件需要数据库连接，那你必须先安装数据库插件。

#### 卸载插件(Uninstall Plugins)
你可以使用一个名字(name)，类(class)或者一个实例(instance)来调用 **uninstall()** 卸载一个之前安装的插件：
```
sqlite_plugin = SQLitePlugin(dbfile='/tmp/test.db')
install(sqlite_plugin)

uninstall(sqlite_plugin) # uninstall a specific plugin
uninstall(SQLitePlugin)  # uninstall all plugins of that type
uninstall('sqlite')      # uninstall all plugins with that name
uninstall(True)          # uninstall all plugins at once
```
插件可以在任何时候安装或是卸载，甚至在服务器运行时也是如此。这种特性可以用来做一些巧妙的技巧(比如仅在需要的时候安装缓慢的调试插件)，但不要过度使用。插件列表的每一次改变，route缓存都会刷新并且所有的插件都会被重新应用。
> 注意：
> 模块级(module-level)的 **install()** 和 **uninstall()**对 [默认应用(Default Application)](http://bottlepy.org/docs/dev/tutorial.html#default-app) 产生作用。要针对指定的应用管理插件，使用 [Bottle](http://bottlepy.org/docs/dev/api.html#bottle.Bottle) 应用对象的相应方法。

### 指定路由的安装(Route-specific Installation)
如果你指向对一小部分routes安装插件，便捷的做法是使用 [route()](http://bottlepy.org/docs/dev/api.html#bottle.route) 装饰器的`apply`参数：
```
sqlite_plugin = SQLitePlugin(dbfile='/tmp/test.db')

@route('/create', apply=[sqlite_plugin])
def create(db):
    db.execute('INSERT INTO ...')
```

### 黑名单插件(Blacklisting Plugins)
你可能想要明确的指定一些routes禁用某个插件。[route()](http://bottlepy.org/docs/dev/api.html#bottle.route) 装饰器有一个`skip`参数可以达到次目的：
```
sqlite_plugin = SQLitePlugin(dbfile='/tmp/test1.db')
install(sqlite_plugin)

dbfile1 = '/tmp/test1.db'
dbfile2 = '/tmp/test2.db'

@route('/open/<db>', skip=[sqlite_plugin])
def open_db(db):
    # The 'db' keyword argument is not touched by the plugin this time.

    # The plugin handle can be used for runtime configuration, too.
    if db == 'test1':
        sqlite_plugin.dbfile = dbfile1
    elif db == 'test2':
        sqlite_plugin.dbfile = dbfile2
    else:
        abort(404, "No such database.")

    return "Database File switched to: " + sqlite_plugin.dbfile
```
`skip`参数接受一个值或者一个列表的值。你可以使用名字(name)，类(class)或是实例(instance)来指定需要被跳过的插件。设置`skip=True`跳过所有插件。

### 插件和子应用(Plugins and Sub-Applications)
大部分插件被指定到它们安装到的应用上。因此，它们不应该对使用 [Bottle.mount()](http://bottlepy.org/docs/dev/api.html#bottle.Bottle.mount) 挂载的子应用产生影响。这有一个例子：
```
root = Bottle()
root.mount('/blog', apps.blog)

@root.route('/contact', template='contact')
def contact():
    return {'email': 'contact@example.com'}

root.install(plugins.WTForms())
```
每当你挂载一个应用，Bottle会在主应用上创建一个代理路由(proxy-route)来转发所有的请求到子应用。默认情况下这些代理路由(proxy-route)是禁用插件的。这样，我们例子中的 *WTForms* 插件会对`/contact`route产生作用，而对子应用的`/blog`route不起作用。

作为默认这是一种稳妥的行为，但这并不是绝对的。下面的例子为指定的代理路由(proxy-route)重新激活了所有的插件：
```
root.mount('/blog', apps.blog, skip=None)
```
但这仍然有一个问题：插件会把整个子应用看做是单独一个路由(也就是上面提到的proxy-route)。为了达到只对子应用中独立的route起作用的目的，你需要明确的指定安装插件到挂载的应用上。

## 开发(Development)
现在你学会了基本的用法想要来做一个你自己的应用吗？这里有些提示或许能对你有所帮助。

### 默认应用(Default Application)
Bottle会管理一个存放 [Bottle](http://bottlepy.org/docs/dev/api.html#bottle.Bottle) 实例的全局栈，并把栈顶元素作为模块级(module-level)方法和装饰器的默认作用对象。例如 [route()](http://bottlepy.org/docs/dev/api.html#bottle.route) 就是在默认应用上调用 [Bottle.route()](http://bottlepy.org/docs/dev/api.html#bottle.Bottle.route) 的一种便捷写法：
```
@route('/')
def hello():
    return 'Hello World'

run()
```
对小型应用来说这非常方便，可以帮你少些一些代码。但这也意味着，只要你导入模块，所有的routes都会被装载到全局的默认应用上。为了避免这种副作用，Bottle提供了第二种选择，以更加清晰的方式来创建应用：
```
app = Bottle()

@app.route('/')
def hello():
    return 'Hello World'

app.run()
```
分离应用对象对提高代码复用性很有帮助。其他的开发者可以安全的从你的模块中导入某个`app`并使用 [Bottle.mount()](http://bottlepy.org/docs/dev/api.html#bottle.Bottle.mount) 方法来合并到其他应用上。

*0.13版新特性*
从 Bottle-0.13 起你可以使用 [Bottle](http://bottlepy.org/docs/dev/api.html#bottle.Bottle) 实例作为上下文管理器了：
```
app = Bottle()

with app:

    # Our application object is now the default
    # for all shortcut functions and decorators

    assert my_app is default_app()

    @route('/')
    def hello():
        return 'Hello World'

    # Also useful to capture routes defined in other modules
    import some_package.more_routes
```

### 调试模式(Debug Mode)
在前期开发中，调试模式非常有用。
```
bottle.debug(True)
```
在调试模式下，当有错误发生时，Bottle 提供更多有用的调试信息。它还会禁用一些可能会妨碍你的优化配置，并提供一些检查来提示你可能存在的错误配置。

这是一些debug模式下不同之处的不完全列表：
* 默认错误页会显示调用堆栈。
* 模板不会被缓存。
* 插架会被立刻的应用。

请确保不要在生产环境中使用debug模式。

### 自动重启(Auto Reloading)
在开发过程中，为了测试改动的代码你不得不一遍遍的重启服务器。自动重启可以帮助你。每次你编辑了一个模块文件，服务器进程会重新启动并加载你最新的代码。
```
from bottle import run
run(reloader=True)
```
工作原理：主进程不会启动服务器，但它会使用与主进程启动时相同的命令行参数创建一个新的子进程。所有的模块级(module-level)代码会被至少执行两遍。请注意这一点。

子进程会把`os.environ['BOTTLE_CHILD']`设置为`True`并作为一个普通的应用服务器启动。当有装载的模块发生变化时，主进程会终止子进程然后重新创建一个新的子进程。模板的改变不会触发重启。请使用debug模式来关闭模板缓存。

重启功能的实现依赖于停止子进程的能力。如果你运行在Windows或是其他不支持用来终止子进程的`signal.SIGINT`(Python中抛出的`KeyboardInterrupt`)的操作系统上,注意退出处理器和最后的条目(clauses)等等，它们不会在`SIGTERM`后执行。

### 命令行界面(Command Line Interface)
从版本 0.10 开始你可以把 Bottle 作用一个命令行工具来使用了：
```
$ python -m bottle

Usage: bottle.py [options] package.module:app

Options:
  -h, --help            show this help message and exit
  --version             show version number.
  -b ADDRESS, --bind=ADDRESS
                        bind socket to ADDRESS.
  -s SERVER, --server=SERVER
                        use SERVER as backend.
  -p PLUGIN, --plugin=PLUGIN
                        install additional plugin/s.
  -c FILE, --conf=FILE  load config values from FILE.
  -C NAME=VALUE, --param=NAME=VALUE
                        override config values.
  --debug               start server in debug mode.
  --reload              auto-reload on file changes.
```
*ADDRESS*参数需要一个IP地址或是IP:PORT。默认为`localhost:8080`。其他的参数应该可以见名知意了。

所有的插件和应用都通过导入(import)表达式来指定。它们由表达式和一个使用点号分隔的导入路径(例：package.module)组成，用于解析模块中的命名空间。具体的介绍可以看 [load()](http://bottlepy.org/docs/dev/api.html#bottle.load)。这有一些例子：
```
# Grab the 'app' object from the 'myapp.controller' module and
# start a paste server on port 80 on all interfaces.
python -m bottle -server paste -bind 0.0.0.0:80 myapp.controller:app

# Start a self-reloading development server and serve the global
# default application. The routes are defined in 'test.py'
python -m bottle --debug --reload test

# Install a custom debug plugin with some parameters
python -m bottle --debug --reload --plugin 'utils:DebugPlugin(exc=True)'' test

# Serve an application that is created with 'myapp.controller.make_app()'
# on demand.
python -m bottle 'myapp.controller:make_app()''
```

## 部署(Deployment)
默认的Bottle运行在内置的 [wsgiref WSGIServer](http://docs.python.org/library/wsgiref.html#module-wsgiref.simple_server) 服务器上。这个非线程(non-threading)的HTTP服务器用在在开发或是运行初期时很不错，但随着服务数量的增长，它可能存在性能上的瓶颈。

提升性能最简单的方式就是安装一个对现场的服务器例如 [paste](http://pythonpaste.org/) 或是 [cherrypy](http://www.cherrypy.org/),然后告诉 Bottle去使用它：
```
bottle.run(server='paste')
```
更多相关介绍在另一篇单独的文章中：[Deployment](http://bottlepy.org/docs/dev/deployment.html)


## 术语(Glossary)
**callback**
    程序员编写callback用于外部动作触发时调用。在web框架的上下文中，URL常常通过一个指定的callback和程序代码映射在一起。
**decorator**
    返回值是另一个函数的函数。通常使用 `@decorator` 语法作为功能转换使用。查看 [Python函数定义文档](http://docs.python.org/reference/compound_stmts.html#function) 查看更多有关decorator的介绍。
**environ**
    一个保存了所有文档信息的结构，用于交叉引用。在分析阶段后环境信息会被序列化(pickle)，因此后续的运行只需要读取解析新的和变化的文档。
**handler function**
    处理指定事件(event)或情况(situation)的函数。在web框架开发中，handler function作为callback依附于应用中指定的URL上。
**source directory**
    项目中包含所有源文件的文件夹及子文件夹。
