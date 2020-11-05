---
title: Python连续赋值的两个要点
date: 2019-05-13 17:00:59
permalink: key-points-of-chained-assignment-in-python/
tags:
- Python
categories:
- Python
---

## 背景

写 Python 四年有余了，常见的坑和奇淫巧技也都知道一些。今天在 Python 连续赋值上遇到了一个新的知识点，学习记录一下。

Python 连续赋值容易导致错误的情况有两个知识点：
1. 相同的引用
2. 赋值的顺序（本次探讨的重点）

<!--more-->

## 相同的引用

这个有点 Python 经验的同学都知道或者很好理解，对可变对象的赋值，其实赋的是可变对象的引用。  
在下面的代码中，因为`['hello']`是一个列表，即可变对象，变量`a`和`b`都是这个同一个列表的引用。因此，对`b`修改时，`a`的值也变了，因为它们本来就指向同一个对象。

```python
s1 = s2 = 'hello'
s2 += 'world'
print(s1)    # Output: 'hello'   
  
a = b = ['hello']
b.append('world')
print(a)    # Output: ['hello', 'world']
print(id(a) == id(b))   # Output: True
```

## 赋值的顺序

通常我们并不是太关心在使用连续赋值过程中赋值的顺序，就像下面这两行代码，赋值的先后次序并不影响结果。

```python
s1 = s2 = 'hello'
a = b = ['hello']
```

但到底是：
- 是先把`'hello'`赋值给`s2`，然后把`s2`赋值给`s1`吗？
- 还是把`'hello'`赋值给`s2`，然后`'hello'`赋值给`s1`呢？
- 又或者是把`'hello'`赋值给`s1`，然后`'hello'`赋值给`s2`呢？

在下面这个例子中让我犯了难。
```python
class ListNode:
    def __init__(self, x):
        self.val = x
        self.next = None

    def __repr__(self):
        return '->'.join(map(str, ([self.val, self.next.val] if self.next else [self.val])))


def main():
    a = ListNode(0)
    a = a.next = ListNode(1)
    print(a)

    b = ListNode(0)
    b.next = ListNode(1)
    b = b.next
    print(b)


if __name__ == '__main__':
    main()
```

> Output:
> 1->1
> 1

我本期望赋值操作可以从右向左进行赋值，让上面代码中的变量`a`和`b`有相同的结果。即：

```python
a = a.next = ListNode(1)
```

我期望上面的语句等同于

```python
a.next = ListNode(1)
a = a.next
```

事与愿违，解释器总是很忠实的按照原有的设计去执行代码。那么它到底是怎么做的呢？
我们可以`import dis`，把 python 代码反汇编来看一下。
```python
import dis

class ListNode:
    def __init__(self, x):
        self.val = x
        self.next = None

    def __repr__(self):
        return '->'.join(map(str, ([self.val, self.next.val] if self.next else [self.val])))


def main():
    a = ListNode(0)
    a = a.next = ListNode(1)
    print(a)

    b = ListNode(0)
    b.next = ListNode(1)
    b = b.next
    print(b)


if __name__ == '__main__':
    dis.dis(main)
```

下面看看 Line 14, 18, 19 翻译过来的汇编码：

汇编语言我并不了解，下面的注释是我根据命令名称的猜测，如有错误恳请指出。

```
...

 14           8 LOAD_GLOBAL              0 (ListNode)    // 加载ListNode
             10 LOAD_CONST               2 (1)            // 加载常量 1
             12 CALL_FUNCTION            1                // 调用函数（可能是ListNode的构造函数）
             14 DUP_TOP
             16 STORE_FAST               0 (a)            // 赋值给变量 a
             18 LOAD_FAST                0 (a)            // 加载变量 a
             20 STORE_ATTR               1 (next)        // 赋值给属性 next

...

 18          38 LOAD_GLOBAL              0 (ListNode)    // 加载ListNode
             40 LOAD_CONST               2 (1)            // 加载常量 1
             42 CALL_FUNCTION            1                // 调用函数（可能是ListNode的构造函数）
             44 LOAD_FAST                1 (b)            // 加载变量 b
             46 STORE_ATTR               1 (next)        // 赋值给属性 next

 19          48 LOAD_FAST                1 (b)            // 加载变量 b
             50 LOAD_ATTR                1 (next)        // 加载变量 b 的属性 next
             52 STORE_FAST               1 (b)            // 赋值给变量 b

...
```

由汇编码就很容易看出，在 Python 中类似`a = b = 'hello'`的连续赋值过程，实际上是将最右侧的常量或变量，对其左侧`=`前的各变量从左至右依次赋值。
即：
```python
a = b = 'hello'
```
等同于
```python
a = 'hello'
b = 'hello'
```

回到上面让我犯难的例子中：

```python
a = a.next = ListNode(1)
```

等同于

```python
_ = ListNode(1)  
a = _
a.next = _
# 注意：这里我用_暂存了对象，整个过程中只创建了一个对象
```

而不是

```python
a.next = ListNode(1)
a = a.next
```

了解了原理，要想实现我期望的效果，其实只要调换一下顺序就行，像下面这样：
```python
a.next = a = ListNode(1)
```

不过这样降低了可读性，不利于理解，最好还是不要这样写！

## 总结

1. 这种错误隐秘难调，最好不要使用连续赋值（Chained Assignment）。
2. 对基础知识点要加强学习。
3. 连续赋值在其他编程语言中似乎不是这样的，Python 是个特例，这点有待考证！！！

