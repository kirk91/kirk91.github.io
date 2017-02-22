title: 在centos7上编译安装nghttp2
date: 2017-02-22 17:34:48
tags:
    - nghttp2
    - centos
    - ops
categories: [nghttp2]

---

> 最近有一个项目使用grpc对外提供服务, 在上线部署过程中遇到些问题，主要是http2流量的负载均衡和加解密。
本想着借助nginx来解决反向代理和加解密的问题，但无奈nginx暂还不支持http2的反向代理而且在相当长的一段时间
内没有计划, 具体信息参见[这里](https://trac.nginx.org/nginx/ticket/923), 于是只能使用其他的解决方案如
[nghttp2](https://nghttp2.org/)或[envoy](https://github.com/lyft/envoy)

[nghttp2](https://nghttp2.org/)是一个由日本工程师开发的项目，提供`C Library`和一些实用的`tools`
(nghttpd nghttp nghttpx等). 作者没有提供通用的发行包rpm或deb, 需要使用的话只能从源码开始编译;
文档上详细描述了在`ubuntu`上的编译[安装流程](https://github.com/nghttp2/nghttp2#requirements)，
但却缺少其他常用发行版`centos`的编译安装流程 (倒吸凉气一口, 公司生产服务器使用的是清一色的centos, 还是只能硬着头皮搞下去).

如果你使用的服务器发行版本是`ubuntu`, 请移步[这里](https://github.com/nghttp2/nghttp2#requirements);
下面将详细描述在`centos`上编译nghttp2的流程:

## 安装依赖

```sh
$ sudo yum -y groupinstall "Development Tools"
$ sudo yum -y install openssl-devel libxml2-devel libev-devel jemalloc-devel python-devel

$ # build libcares from source code
$ # download the latest version
$ wget https://c-ares.haxx.se/download/c-ares-1.12.0.tar.gz -O /tmp/c-ares.tar.gz
$ mkdir -p /tmp/c-ares
$ tar -zxvf /tmp/c-ares.tar.gz -C /tmp/c-ares --strip-components=1
$ cd /tmp/c-ares && ./configure --libdir=/usr/lib64
$ make
$ sudo make install

$ # build jansson from source code
$ wget http://www.digip.org/jansson/releases/jansson-2.9.tar.gz -O /tmp/jansson.tar.gz
$ mkdir -p /tmp/jansson
$ tar -zxvf /tmp/jansson.tar.gz -C /tmp/jansson --strip-components=1
$ cd /tmp/jansson && ./configure --libdir=/usr/lib64
$ make
$ make check
$ sudo make install
```

## 编译安装nghttp2

- 下载最新版本源码

```sh
$ wget https://github.com/nghttp2/nghttp2/releases/download/v1.19.0/nghttp2-1.19.0.tar.gz -O /tmp/nghttp2.tar.gz
$ mkdir -p /tmp/nghttp2
$ tar -zxvf /tmp/nghttp2.tar.gz -C /tmp/nghttp2 --strip-components=1
```

- 生成Makefile

```sh
$ cd /tmp/nghttp2 && ./configure --enable-app
```

注: 如果输出中包含`package xxx not found`, 请自行goolge安装相应的依赖或给我留言

- 编译

```sh
$ make
```

- 安装

```sh
$ sudo make install
$ # show nghttpx version
$ nghttpx -v
```

关于`nghttpx`的使用和参数配置可以参照[官方文档](https://nghttp2.org/documentation/nghttpx-howto.html)
