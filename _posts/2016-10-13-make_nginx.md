---
layout: post
title:  "nginx源码编译"
date:   2016-10-13 15:43:42 +0800
categories: web开发
tags: nginx
---

由于官网目前能下载到的`nginx-1.11.4`不支持`http2`：

```bash
D:\http2\nginx-1.11.4>nginx -s reload
nginx: [emerg] the "http2" parameter requires ngx_http_v2_module in D:\http2\nginx-1.11.4/conf/nginx.conf:98
```

原因是官方编译的时候没有带上`--with-http_v2_module`，因此需要重新编译`nginx`

要在`windows`下编译，太痛苦了吧，还是移师`mac`吧

重新编译`nginx`需要依赖以下模块：

* `gzip`模块需要`zlib`库    ----  已经有了

* `rewrite`模块需要`pcre`库   ---- 不需要编译，只需要编译`nginx`的时候指定源码路径

* `ssl`功能需要`openssl`库    ----  已经有了，但是怕内建的版本太旧不支持`ALPN`，还是重新下载编译吧

#### 1. 编译OpenSSL

在[http://www.openssl.org/source](http://www.openssl.org/source)下载当前最新的版本源码

解压缩`openssl-xx.tar.gz`包， 进入解压缩目录，执行：

```bash
./config
make
sudo make install
```

#### 2. 编译Nginx

在[http://www.pcre.org](http://www.pcre.org)下载当前最新的版本的`pcre`源码

在[http://nginx.org/en/download.html](http://nginx.org/en/download.html)下载当前最新的版本的`nginx`源码

解压缩`pcre-xx.tar.gz`、`nginx-xx.tar.gz`包，进入解压缩目录，执行：

```bash
./configure --with-pcre=/Users/enzo/http2/pcre-8.39 --with-http_v2_module --with-http_ssl_module
make
sudo make install
```
 

Ok，完成！
