---
title: 源码编译安装Python3及问题解决
date: 2017-5-4 11:07:05
tags:
- Python
categories:
- Python
---


今天在用源码编译安装Python3.6的时候遇到了ssl模块找不到的问题，查了些资料解决了，记录一下。
<!--more-->

## 第一次的安装步骤
```
wget https://www.python.org/ftp/python/3.6.1/Python-3.6.1.tgz
tar -zxf Python-3.6.1.tgz
cd Python-3.6.1
./configure
make
make install
```

在python命令行中执行`import ssl`会报错：`ImportError: No module named _ssl `。
检查`make`的输出结果，显示：
```
Failed to build these modules:
_hashlib     _ssl
```

## 解决方案
删除之前编译过的文件夹，重新解压一份。
修改`Python3.6.1/setup.py`文件：
```
# Detect SSL support for the socket module (via _ssl)
        search_for_ssl_incs_in = [
                              '/usr/local/ssl/include',
                              '/usr/local/ssl/include/openssl', # 添加此行，否则可能会报错找不到 rsa.h 文件
                              '/usr/contrib/ssl/include/'
```

修改`Python3.6.1/Module/Setup.dist`文件：
```
# line: 207
# Socket module helper for SSL support; you must comment out the other
# socket line above, and possibly edit the SSL variable:
SSL=/usr/local/ssl
_ssl _ssl.c \
        -DUSE_SSL -I$(SSL)/include -I$(SSL)/include/openssl \
        -L$(SSL)/lib -lssl -lcrypto
```
把上面的四行注释放开，然后再执行：
```
./configure
make
make install
```
即可。

方法参考自：http://stackoverflow.com/questions/5937337/building-python-with-ssl-support-in-non-standard-location

## 可能无效的方法
在网上搜索的时候还尝试了下面的方法，然而并没有什么用：
在执行`./configure`时带参数`--with-ssl`
此方法可能已过时，参考自：http://stackoverflow.com/questions/5128845/importerror-no-module-named-ssl


## 注意事项
最好还是带前缀编译安装，我这种方法可能会覆盖系统自带的python而造成不必要的麻烦。
