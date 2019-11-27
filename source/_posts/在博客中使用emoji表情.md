title: 在博客中使用emoji表情
date: 2016-02-04 19:20:13
permalink: add-emoji-selector-in-blog
tags:
- Django
- emoji
- Python
categories:
- Django
---
近期在尝试使用 Django 开发一个个人博客，在评论和留言功能中，我希望能加入些表情。emoji 作为一种流行的字符表情当然是首选。在这个过程中我遇到了一些问题，现在解决了，记录一下。
<!--more-->

要在 web 页面的评论框中添加输入 emoji 表情的功能，我想应该需要 js 插件来实现。我在 Github 上找到了[Emoji Picker](https://github.com/one-signal/emoji-picker)。这是一个emoji表情选择器，它可以在你的输入框上添加一个表情选择按钮。像这样：
![](https://camo.githubusercontent.com/98fa676463e7d839be12101225dac3e37daf0bf2/687474703a2f2f6f6e652d7369676e616c2e6769746875622e696f2f656d6f6a692d7069636b65722f73637265656e73686f742e706e67)

那么如何使用呢？它的文档是这么写的：
*注意：按它的文档做是不够的！这是个坑！*
> # Installation & Usage:

>*On Robustness*: This library isn't super robust, so if you find any issues, please report it so it can be fixed (or feel free to fix it yourself). Code quality improvements are also welcome, always looking to make it better!

>*On CDN & Multiple Files*: Currently, the number of JavaScript files you have to include is not ideal (6 files). The files will eventually be concatenated and minified, but it might be a bit until this happens. When that's complete, it'll be added to CDNJs as well.

>1. In your `<head>` section, add the following *stylesheet* links. Adjust the `lib/css` path to match yours.

>  ```
    <link href="lib/css/nanoscroller.css" rel="stylesheet">
    <link href="lib/css/emoji.css" rel="stylesheet">
  ```

>2. Before the end of your `<body>` section, add the following *JavaScript* links. This library depends on jQuery, so jQuery must also be included, before these scripts are run. Once again, adjust the `lib/css` path to match yours.

>  ```
    <!-- ** Don't forget to Add jQuery here ** -->
    <script src="lib/js/nanoscroller.min.js"></script>
    <script src="lib/js/tether.min.js"></script>
    <script src="lib/js/config.js"></script>
    <script src="lib/js/util.js"></script>
    <script src="lib/js/jquery.emojiarea.js"></script>
    <script src="lib/js/emoji-picker.js"></script>
  ```

>3. On any input field, add the data attribute `data-emojiable="true"`. 

>4. That's all you need for the default options. Play around with the demo to see what the default options give you.

它的文档说的很简单，总结起来就是：
1. 在页面头部`<head>`中引入它的 css 文件
2. 在`<body>`最后引入它的 javascript 文件，别忘了先引入jQuery
3. 在任何 `<input>`标签中加入属性`data-emojiable="true"`
4. 完成， 如果使用默认设置，这样就可以了

**然而这样是不够的！**
我的页面看起来什么也没有发生。查看一下它提供的demo源码，原来最下面还有这么一段js：
```
<script>
    $(function() {
        // Initializes and creates emoji set from sprite sheet
        window.emojiPicker = new EmojiPicker({
            emojiable_selector: '[data-emojiable=true]',
            assetsPath: 'lib/img/',
            popupButtonClasses: 'fa fa-smile-o'
     });
      // Finds all elements with `emojiable_selector` and converts them to rich emoji input fields
      // You may want to delay this step if you have dynamically created input fields that appear later in the loading process
      // It can be called as many times as necessary; previously converted input fields will not be converted again
      window.emojiPicker.discover();
    });
  </script>
```
所以我们加上这一段，别忘了修改`assetsPath`为你的路径。

试了一下，为什么还是没有demo中看到的那个笑脸按钮？
再看源码，原来demo还有引用了这么一个css：
```
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/font-awesome/4.4.0/css/font-awesome.min.css">
```
这是一套专门为bootstrap设计的图标字体库,名为font-awesome。
我们可以直接引用这个css或者去下载一份 font-awesome 添加到项目。
至此，终于可以在页面中使用它了！
