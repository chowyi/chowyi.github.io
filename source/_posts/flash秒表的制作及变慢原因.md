---
title: flash秒表的制作及变慢原因
date: 2010-06-07 16:33:27
tags:
- Flash
categories:
- Flash
---


我使用 Adobe flash CS3 professional(ActionScript 2.0) 制作过一个电子秒表，后来发现秒表出现明显的变慢现象，经过一番尝试后，终于解决了这个小问题。在这里把经验与大家分享一下。
<!--more-->

在说明问题原因之前，很有必要叙述一下我的秒表机制。

首先，定义了4个变量，分别用作：时（myHour）、分（myMinute）、秒（mySecond）、毫秒（myMillisecond）。然后使用`setInterval()`函数，每隔10毫秒将变量myMillisecond递加1,到99时，myMillisecond再恢复到0，并把mySecond，如此依次递增myMinute、myHour。之后用动态文本显示即可。

利用上述机制我制作了一个flash电子表，测试成功。可不久就发现了秒表变慢的问题。

解决办法：我提高了flash的帧频，问题迎刃而解了。flash的默认帧频是12fps，最高帧频为120fps。当我提高帧频至80fps（或更高）时，问题就很好的解决了！

希望对大家有帮助~~