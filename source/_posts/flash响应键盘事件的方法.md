---
title: flash响应键盘事件的方法
date: 2010-02-04 21:55:00
tags:
- Flash
categories:
- Flash
---


这篇日志摘自 [闪客帝国](http://www.flashempire.com/) 卡其色 的文章。与大家分享一下。
<!--more-->

## 方法一

第一种响应键盘的方法：利用按钮进行检测来实现响应键盘在按钮的on事件处理函数中不但可以对鼠标事件作出响应，而且可以对键盘事件作出响应。 如在按钮的动作面板中加入如下所示的代码，在敲击键盘上的X键时输出窗口中将提示： X is pressed
在按钮上加上：
```
on(keyPress "x"){
    trace("X is pressed");
} 
```

但是要注意的是：检测键盘上的字母键时，字母都应为小写。
如果要检测键盘中的特殊键，Flash中有一些专门的代码来表示它们，下面列出了一些常用的功能键的表示代码：

`<Left>`
`<Right>`
`<Up>`
`<Down>`
`<Space>`
`<Home>`
`<End>`
`<Insert>`
`<PageUp>`
`<PageDown>`
`<Enter>`
`<Delete>`
`<Backspace>`
`<Tab>`
`<Escape>`

如要检测键盘上的`<Left>`键，可以使用下面的ActionScript：
```
on(keyPress "&lt;Left&gt;"){
    trace("Left is pressed");
}
```

另外，你可以在一个按钮中加入若干个on函数，也可以在一个on函数中结合多种事件，这使 您可以为按钮定义自己熟悉常用的快捷键，如下所示：
```
on(release, keyPress "&lt;Left&gt;"){
    _root.myMC.prevFrame();
}
on(release, keyPress "&lt;Right&gt;"){
    _root.myMC.nextFrame();
}
```

上面的第一个语句实现单击按钮或按键盘上的左方向键，控制影片剪辑myMC回退1帧，
而上面的第二个语句实现单击按钮或按键盘上的右方向键，控制影片剪辑myMC前进1帧。 


## 方法二

第二种响应键盘的方法：利用Key对象来实现响应键盘的操作

利用按钮检测按键动作很有效，但是并不利于检测持续按下的键，所以不适合于制作某些通过键盘控制的游戏。
这时，您就需要用到Key对象。Key对象包含在动作面板的“对象”/“影片”目录下面，它由Flash内置的一系列方法、常量和函数构成。使用Key对象可以检测某个键是否被按下，如要检测左方向键是否被按下，可以使用如下ActionScript：
```
if (Key.isDown(Key.LEFT)){  
    trace("The left arrow is down");
}
```

函数`Key.isDown`返回一个布尔值，当该数中的参数对应的键被按下时返回`true`，否则返回`false`。常量`Key.LEFT`代表键盘上的左方向键。当左方向键被按下时，该函数返回`true`。
Key对象中的常量代表了键盘上相应的键，下面列出了一些基本的常量：
一些功能键的表示：
`Key.BACKSPACE`
`Key.ENTER`
`Key.PGUP`
`Key.PGDN`
`Key.CAPSLOCK`
`Key.ESCAPE`
`Key.LEFT`
`Key.RIGHT`
`Key.UP`
`Key.DOWN`
`Key.HOME`
`Key.END`
`Key.CONTROL`
`Key.SHIFT`
`Key.DELETEKEY`
`Key.INSERT`
`Key.SPACE`
`Key.TAB`

以上是键盘上的功能键，那么如何表示键盘上的字母键呢？
Key对象提供了一个函数`Key.getCode`来实现这一功能，如下所示：
```
if (Key.isDown(Key.getCode("x"))){  
    trace("X is pressed");  
}
```

上面脚本的意思就是，利用`Key.getCode`函数来告诉系统你是否按下了x键，如果按下了x键以后，函数`Key.isDown`则会返回`true`,在输出窗口就会输出X is pressed。


## 方法三

第三种响应键盘的方法：利用键盘侦听的方法来实现响应键盘（个人习惯用这种方法）

假设在影片剪辑的onClipEvent(enterFrame)事件处理函数中检测按键动作，而影片剪辑所在的时间轴较长，或计算机运算速度较慢，就有可能出现这种情况：即当在键盘上按下某个键时还未来得及处理onClipEvent(enterFrame)函数，那么按键动作将被忽略，这样的话，很多你想要的效果就会无法实现了。
另外，还有一个需要解决的问题就是，在某些游戏（如射击）中，我们需要按一次键就执行一次动作（发射一发子弹），即使长时间按住某个键不放也只能算作一次按键，而Key对象并不能区别是长时间按住同一个键还是快速地多次按键。
所以如果要解决这个问题，就需要用到键盘侦听的方法。你可以使用 “侦听器（listener）”来侦听键盘上的按键动作。
要使用侦听器之前，首先需要创建它，你可以使用如下所示的命令来告诉计算机你需要侦听某个事件： 
```
Key.addListener(_root);
```

`Key.addListener`命令将主时间轴或某个影片剪辑 作为它的参数，当侦听的事件发生时，可以用这个参数指定的对象来响应该事件。
上面的代码指定主时间轴来响应该事件。要让主时间轴对该事件作出响应，还需要设置一个相应的事件处理函数，否则设置侦听器就没有什么意义了。
键盘侦听的事件处理函数有两个：onKeyUp和onKeyDown，如下所示： 
```
Key.addListener(_root);  
_root.onKeyUp = function(){  
    trace(Key.getAscii()); 
}
```

代码的意思是，当按下一个键并释放后，输出窗口将输出你按下的那个键的Ascii码
当然，你也可以使用影片剪辑作为侦听键盘的对象，只需要使用影片剪辑的路径代替`_root`作为`Key.addListener`命令的参数就可以了。

比如下面代码： 
```
Key.addListener(_root.mc);  
_root.mc.onKeyUp = function(){  
    trace(Key.getAscii());  
}
```

代码的意思是，当按下一个键并释放后，输出窗口将输出你按下的那个键的Ascii码,意思差不多，但是键盘侦听对象不同，一个是影片mc,一个是主时间轴。


## 方法四

第四种响应键盘的方法：利用影片剪辑的`keyUp`和`keyDown`事件来实现响应键盘。

最后一种方法很容易被忽视，但是也有一定的应用价值，最重要的是把概念弄清楚。
影片剪辑包含两个与键盘相关的事件keyUp和keyDown，使用它们也可以实现对按键事件的响应
例如下面的代码： 
```
onClipEvent (keyDown){
    trace(Key.getAscii());
}
//当按下键盘上的一个键的时候，输出窗口将输出按下的这个键的Ascii码值
```

函数`Key.getAscii`表示返回与按键相对应的ASCII码，其中ASCII码是一个整数，键盘上的每个字符对应一个ASCII码，如字母A对应的ASCII码为65，B对应的ASCII码为66，a对应的ASCII码为97, b对应的ASCII码为98，+ 对应的ASCII码为43等。需要注意的是：只有字符键才有ASCII码，键盘上的功能键是没有ASCII码的。
如果我想在输出窗口中输出与按键相对应的字符，那怎么办？
这时候，你可以使用String对象的fromCharCode函数将ASCII码转换成字符，如将上例的代码改成如下所示：
```
onClipEvent (keyDown){
    trace(String.fromCharCode(Key.getAscii()));
};
//意思就是说，当按下键盘的一个键，输出按下的这个键相对应的字符，当然除了功能键。
关于String对象的详细解释，大家可以查看动作面板的“对象”/“核心”目录下面。
```