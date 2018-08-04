---
title: Nginx安装笔记
date: 2016-07-30 17:14:27
tags:
- nginx
categories:
- DevOps
---
在工作中接触到了Nginx，非常感兴趣。自己尝试着安装了一下，在这里记录一下安装过程吧。
<!--more-->
## 系统环境
我是在虚拟机中安装的，操作系统是 Ubuntu 12.04.5 LTS

## 安装依赖模块
Nginx依赖以下模块：
- zlib (gzip模块需要 zlib 库)
- pcre (rewrite模块需要 pcre 库)
- openssl (ssl 功能需要openssl库)

### 安装pcre

1. 下载pcre编译安装包。下载地址：[http://www.pcre.org/](http://www.pcre.org/)
    我下载的是 [pcre-8.38.tar.gz](ftp://ftp.csx.cam.ac.uk/pub/software/programming/pcre/pcre-8.38.tar.gz)。注意不要装成pcre2了。
2. 解压缩pcre-xx.tar.gz包。
3. 进入解压缩目录，执行./configure。 （可能需要sudo权限）
4. make （可能需要sudo权限）
5. make install （可能需要sudo权限）

> 如果报错：You need a C++ compiler for C++ support，则需要安装gcc和g++。
网上查到的安装方法大部分为：
`yum install -y gcc gcc-c++`
在Ubuntu上则是：
`sudo apt-get install gcc g++`


### 安装zlib

1. 下载zlib编译安装包。下载地址：[http://www.zlib.net/](http://www.zlib.net/)
    我下载的是 [zlib-1.2.8.tar.gz](http://zlib.net/zlib-1.2.8.tar.gz)
2. 解压缩zlib-xx.tar.gz包。
3. 进入解压缩目录，执行./configure。 （可能需要sudo权限）
4. make （可能需要sudo权限）
5. make install （可能需要sudo权限）


### 安装openssl

1. 下载openssl编译安装包。下载地址：[https://www.openssl.org/source/](https://www.openssl.org/source/)
    我下载的是 [openssl-1.0.2h.tar.gz](https://www.openssl.org/source/openssl-1.0.2h.tar.gz)
2. 解压缩zlib-xx.tar.gz包。
3. 进入解压缩目录，执行./config。 （可能需要sudo权限）
4. make （可能需要sudo权限）
5. make install （可能需要sudo权限）

## 安装Nginx

1. 下载Nginx编译安装包。下载地址：[http://nginx.org/en/download.html](http://nginx.org/en/download.html)
    我下载的是 [nginx-1.10.1.tar.gz](http://nginx.org/download/nginx-1.10.1.tar.gz)
2. 解压缩nginx-xx.tar.gz包。
3. 进入解压缩目录，执行./configure。 （可能需要sudo权限）
4. make （可能需要sudo权限）
5. make install （可能需要sudo权限）

> 若安装时找不到上述依赖模块，使用`--with-openssl=<openssl_dir>`、`--with-pcre=<pcre_dir>`、`--with-zlib=<zlib_dir>`指定依赖的模块目录。如已安装过，此处的路径为安装目录；若未安装，则此路径为编译安装包路径，nginx将执行模块的默认编译安装。

## 参考资料
[Nginx安装与使用 - 吴秦 - 博客园 http://www.cnblogs.com/skynet/p/4146083.html](http://www.cnblogs.com/skynet/p/4146083.html)
