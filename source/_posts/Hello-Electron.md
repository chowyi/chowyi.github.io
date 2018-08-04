---
title: Hello Electron
date: 2017-4-26 10:04:22
tags:
- JavaScript
- Electron
categories:
- JavaScript
---


近半年来接触到了大量的前端开发工作，ReactJs， NodeJs等技术让我对Web前端有了颠覆性的认识。常年游走在后端的我，也对Js产生了浓厚的兴趣。这两天发现了Electron，看起来很有意思，学习记录一下。
<!--more-->


## What is Electron？

[官网](https://electron.atom.io/)的介绍：
> Build cross platform desktop apps with JavaScript, HTML, and CSS

> If you can build a website, you can build a desktop app. Electron is a framework for creating native applications with web technologies like JavaScript, HTML, and CSS. It takes care of the hard parts so you can focus on the core of your application. 

Electron是一个基于 [Node.js](https://nodejs.org/) 和 [Chromium](http://www.chromium.org/) 的框架，你可以使用JavaScript, HTML 和 CSS来构建跨平台的桌面应用。

我以前用过C++写过Qt，用C#写过WPF， 用Java写过Swing，但用Js来构建桌面应用，我还是第一次听说。而且因为是基于Node.js，Electron天然的拥有跨平台的特性。看起来很可口，赶紧试一试！


## 第一个Electron项目

### 安装 node 和 npm

非常简单，网上教程很多，不再赘述。已安装跳过此步骤。
Windows 和 Linux方法相同：
下载编译好的程序包解压到任意目录，配置好环境变量即可。    
国内用户可以执行`npm install cnpm -g`安装cnpm来使用淘宝的npm镜像加速。

### 使用npm初始化项目

1. 新建项目目录`hello`
2. 在项目目录`hello`下执行`npm init`，根据提示填写项目信息。
    ```
    E:\test\electron\hello>npm init
    This utility will walk you through creating a package.json file.
    It only covers the most common items, and tries to guess sensible defaults.

    See `npm help json` for definitive documentation on these fields
    and exactly what they do.

    Use `npm install <pkg> --save` afterwards to install a package and
    save it as a dependency in the package.json file.

    Press ^C at any time to quit.
    name: (hello)
    version: (1.0.0)
    description: This is my fisrt Electron application.
    entry point: (index.js)
    test command:
    git repository:
    keywords:
    author: ChowYi
    license: (ISC)
    About to write to E:\test\electron\hello\package.json:

    {
        "name": "hello",
        "version": "1.0.0",
        "description": "This is my fisrt Electron application.",
        "main": "index.js",
        "scripts": {
            "test": "echo \"Error: no test specified\" && exit 1"
        },
        "author": "ChowYi",
        "license": "ISC"
    }

    Is this ok? (yes)
    ```
    完成后会在当前目录下生成一个`package.json`文件。
3. 执行`npm install --save-dev electron`安装Electron
    `--save`表示安装完成后自动在上一步生成的`package.json`文件中添加此软件为依赖项
    `-dev`表示把依赖项添加到开发环境而不是生产环境的配置中
    注：网上大部分教程这一步的命令是：`npm install --save-dev electron-prebuilt`，这已经过时了。这么做也许依然可以成功安装，但官方更推荐新的用法，关于其中的来龙去脉有兴趣的同学可以看这篇[文章](https://electron.atom.io/blog/2016/08/16/npm-install-electron)。
4. 安装完成后`package.json`应该如下（版本号也许会不一样）：
    ```
    {
        "name": "hello",
        "version": "1.0.0",
        "description": "This is my fisrt Electron application.",
        "main": "index.js",
        "scripts": {
            "test": "echo \"Error: no test specified\" && exit 1"
        },
        "author": "ChowYi",
        "license": "ISC",
        "devDependencies": {
            "electron": "^1.6.6"
        }
    }
    ```
### 编写代码

在项目目录`hello`下新建文件`index.js`，文件名取决于`package.json`中`main`配置的值。
编写代码:
(这段代码来自electron的[示例项目](https://github.com/electron/electron-quick-start/blob/master/main.js)，注释清晰完备，不需要再解释什么了)
```
const electron = require('electron')
// Module to control application life.
const app = electron.app
// Module to create native browser window.
const BrowserWindow = electron.BrowserWindow

const path = require('path')
const url = require('url')

// Keep a global reference of the window object, if you don't, the window will
// be closed automatically when the JavaScript object is garbage collected.
let mainWindow

function createWindow () {
    // Create the browser window.
    mainWindow = new BrowserWindow({width: 800, height: 600})

    // and load the index.html of the app.
    mainWindow.loadURL(url.format({
        pathname: path.join(__dirname, 'index.html'),
        protocol: 'file:',
        slashes: true
    }))

    // Open the DevTools.
    // mainWindow.webContents.openDevTools()

    // Emitted when the window is closed.
    mainWindow.on('closed', function () {
	// Dereference the window object, usually you would store windows
	// in an array if your app supports multi windows, this is the time
	// when you should delete the corresponding element.
	mainWindow = null
    })
}

// This method will be called when Electron has finished
// initialization and is ready to create browser windows.
// Some APIs can only be used after this event occurs.
app.on('ready', createWindow)

// Quit when all windows are closed.
app.on('window-all-closed', function () {
    // On OS X it is common for applications and their menu bar
    // to stay active until the user quits explicitly with Cmd + Q
    if (process.platform !== 'darwin') {
        app.quit()
    }
})

app.on('activate', function () {
    // On OS X it's common to re-create a window in the app when the
    // dock icon is clicked and there are no other windows open.
    if (mainWindow === null) {
        createWindow()
    }
})

// In this file you can include the rest of your app's specific main process
// code. You can also put them in separate files and require them here.
```

上面的代码中加载了`index.html`文件，所以我们现在新建文件`index.html`：
```
<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>Hello World!</title>
</head>
<body>
    <h1>Hello World!</h1>
    <!-- All of the Node.js APIs are available in this renderer process. -->
    We are using Node.js <script>document.write(process.versions.node)</script>,
    Chromium <script>document.write(process.versions.chrome)</script>,
    and Electron <script>document.write(process.versions.electron)</script>.
</body>

<script>
    // You can also require other files to run in this process
    require('./renderer.js')
</script>
</html>
```
### 启动项目

4.1 设置启动脚本：在`package.json`中的`scripts`部分再加一行：`"start": "electron ."`
4.2 执行`npm start`启动项目

### 完成

如果项目启动没有报错，而且弹出了一个窗口，写着大大的 Hello World! 就说明我们的第一个Electron项目成功了！
