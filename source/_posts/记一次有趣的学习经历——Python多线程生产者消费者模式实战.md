---
title: 记一次有趣的学习经历——Python多线程生产者消费者模式实战
date: 2017-04-19 11:45:02
tags:
- Python
- 多线程
categories:
- Python
---


从接触Python到现在也有将近两年的时间了，工作中对异步任务的处理一直依靠Celery框架来完成,对Python多线程这部分知识一直没有一个感性直接的认识。这两天正好遇到一个小任务，在遇到问题--寻求解决方案--完成任务的过程中，学习了Python多线程的知识并切身体会到了它的优势与应用场景，觉得非常有趣，记录分享在这里。
<!--more-->

## 任务需求
现在有一个excel文件包含上千条url,它们在工作表的第一列（格式如下）。任务要求检查这些url的可访问性，把对应url请求的Http响应码存入第二列，把页面内容保存在第三列。

|&nbsp;|A|B|C|
|--|--|--|--|
|1|https://www.baidu.com|&nbsp;|&nbsp;|
|2|https://github.com|&nbsp;|&nbsp;|
|3|https://www.python.org|&nbsp;|&nbsp;|

## 尝试
任务需求看起来简单且清晰，我想了一下，Python有xlrd和xlwt用来读写excel, 用requests来做Http请求，没有技术难点，那么开始写代码吧。
```
# -*- coding:utf8 -*-
import xlrd
import xlwt
import requests

def main():
    # 新建输出excel文件
    output_file = xlwt.Workbook(encoding = 'utf-8')     
    output_table = output_file.add_sheet('sheet1')

    # 打开保存了url信息的excel文件
    url_file = xlrd.open_workbook('url.xlsx')
    url_table = url_file.sheets()[0]
    nrows = url_table.nrows

    for row_num in xrange(nrows):
        row = url_table.row_values(row_num)
        url = row[0]
        output_table.write(row_num, 0, url)
        print 'get url: %s' % url
        try:
            resp = requests.get(url)
            status_code = resp.status_code
            output_table.write(row_num, 1, status_code)

            if status_code == requests.codes.ok:
                try:
                    content = resp.content[:32000]
                    output_table.write(row_num, 2, content)
                except Exception as e:
                    print url, e.message

        except Exception as e:
            output_table.write(row_num, 1, 'Error')

    output_file.save('./output.xls')


if __name__ == '__main__':
    main()
```

测试运行，代码工作正常，只是速度太慢了，检查千余项url至少要花费1小时以上，不给力啊！

## 修改
上面的代码已经跑通了业务逻辑，但阻塞式的等待网络请求响应花费了太多时间，如果能并发的向多个url发出请求互不等待就好了。这样多线程的方案就呼之欲出了。
```
# -*- coding:utf8 -*-
import xlrd
import xlwt
import requests
import threading

# 并发线程数
THREAD_NUM = 30

def testUrl(url, index, table):
    print 'get url: %s' % url
    table.write(index, 0, url)

    try:
        resp = requests.get(url)
        status_code = resp.status_code
        table.write(index, 1, status_code)

        if status_code == requests.codes.ok:
            try:
                content = resp.content[:32000]
                table.write(index, 2, content)
            except Exception as e:
                print url, e.message
    except:
        table.write(index, 1, 'Error')


def batchTestUrls(rows, table):
    for row in rows:
        testUrl(row['url'], row['row_num'], table)


def main():
    output_file = xlwt.Workbook(encoding = 'utf-8')     
    output_table = output_file.add_sheet('sheet1')

    rows = []

    url_file = xlrd.open_workbook('url.xlsx')
    url_table = url_file.sheets()[0]
    nrows = url_table.nrows

    for row_num in xrange(nrows):
        row = url_table.row_values(row_num)
        url = row[0]
        rows.append({'row_num': row_num, 'url': url})

    size = len(rows) / THREAD_NUM
    for i in range(THREAD_NUM):
        if i == THREAD_NUM - 1:
            rows_slice = rows[i*size:]
        else:
            rows_slice = rows[i*size: i*size+size]
        t = threading.Thread(target=batchTestUrls, args=(rows_slice, output_table))
        t.start()
        t.join()


    output_file.save('./output.xls')


if __name__ == '__main__':
    main()
```
在上面的代码中我设置了30个线程，把所有的url切片分给这些线程同时执行网络请求。
测试运行，看起来好很多了，比起第一版代码效率有了明显提升。
但很快我就发现了新的问题：因为一开始就把url切片分给了多个线程，在程序运行一段时间后，有些线程已经完成任务停止工作了，有些线程还在孤军奋战，这显然是浪费了资源嘛。
还能再给力一点吗？

## 完善
第二版代码最大的问题就是多线程资源没有充分的利用起来，一开始就给这些线程分配定量的工作，干的快线程的完成任务就闲下来了。如果能让这些线程在完成一项工作之后主动获取新的工作就好了。
想到这一点，方案就清晰了：
现在我只要把待处理的工作都放在一个任务队列里，然后让线程们自己去任务队列认领工作，完成一项就再认领一项，直到队列中没有任务为止。
```
# -*- coding:utf8 -*-
import xlwt
import xlrd
import requests
import threading
from time import time
from Queue import Queue


# 并发线程数
THREADS_NUM = 30


def testUrl(url, index, table):
    print 'get url: %s' % url
    table.write(index, 0, url)

    try:
        resp = requests.get(url)
        status_code = resp.status_code
        table.write(index, 1, status_code)

        if status_code == requests.codes.ok:
            try:
                content = resp.content[:32000]
                table.write(index, 2, content)
            except Exception as e:
                print url, e.message
    except:
        table.write(index, 1, 'Error')
    
    
class Producer(threading.Thread):
    def __init__(self, name, queue):
        threading.Thread.__init__(self, name=name)
        self.queue = queue

    def run(self):
        url_file = xlrd.open_workbook('url.xlsx')
        url_table = url_file.sheets()[0]
        nrows = url_table.nrows

        for row_num in xrange(nrows):
            row = url_table.row_values(row_num)
            url = row[0]
            self.queue.put((row_num, url))


class Consumer(threading.Thread):
    def __init__(self, name, queue, table):
        threading.Thread.__init__(self, name=name)
        self.queue = queue
        self.table = table

    def run(self):
        while not self.queue.empty():
            index, url = self.queue.get()
            testUrl(url, index, self.table)


def main():
    file = xlwt.Workbook(encoding = 'utf-8')     
    table = file.add_sheet('sheet1') 


    queue = Queue()
    producer = Producer('Producer', queue)

    consumers = []
    for i in range(THREADS_NUM):
        consumer = Consumer('Consumer_%s' % str(i), queue, table)
        consumers.append(consumer)

    producer.start()
    producer.join()
    for consumer in consumers:
        consumer.start()

    
    for consumer in consumers:
        consumer.join()

    file.save('./output.xls')
    print 'All threads finished!'
    


if __name__ == '__main__':
    start = time()
    main()
    end = time()
    print 'time used %s seconds' % str(end - start)
```
上面的代码使用Python的Queue做任务队列，编写了一个Producer类向队列里插入任务，编写了一个Consumer类从队列里获取并完成任务。现在只要实例化多个Consumer，它们就会不知疲倦的工作起来啦。


## 总结
- Python多线程使用场景：
    在IO密集型的任务中使用多线程可以显著提升程序效率。本次任务中有大量的时间耗费在等待网络请求响应而CPU处于空闲状态，是一个应用多线程的典型场景。
- 在Python中如何创建多线程：
    Thread是Python中的线程类，有两种使用方法：
    方法一：
    ```
    import threading

    def action(arg):
        # do something
        print 'action %s' % arg
    
    for i in range(5):
        t = threading.Thread(target=action, args=(i,))
        t.start()
    ```

    方法二：
    ```
    class TaskThread(threading.Thread):
        def __init__(self, arg)
            super(TaskThread, self).__init__()
            self.arg = arg

        def run(self):
            # do something
            print 'arg is :%s' % self.arg

    for i in range(5):
        t = TaskThread(i)
        t.start()
    ```
    join()方法
    > join([timeout])

    > Wait until the thread terminates. This blocks the calling thread until the thread whose join() method is called terminates – either normally or through an unhandled exception – or until the optional timeout occurs.

    > When the timeout argument is present and not None, it should be a floating point number specifying a timeout for the operation in seconds (or fractions thereof). As join() always returns None, you must call isAlive() after join() to decide whether a timeout happened – if the thread is still alive, the join() call timed out.

    > When the timeout argument is not present or None, the operation will block until the thread terminates.

    > A thread can be join()ed many times.

    > join() raises a RuntimeError if an attempt is made to join the current thread as that would cause a deadlock. It is also an error to join() a thread before it has been started and attempts to do so raises the same exception.
    简单的说就是：阻塞当前上下文环境的线程，直到调用此方法的线程终止或到达指定的timeout（可选参数）。
- Queue的使用：
    Python的Queue模块中的Queue类实现了一个线程安全的队列，在多线程编程中十分有用，避免了自己去操作锁的麻烦。


