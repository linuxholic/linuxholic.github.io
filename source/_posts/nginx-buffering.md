---
title: nginx-buffering
date: 2021-11-23 20:37:06
tags: nginx
---

## nginx 缓冲

[Module ngx\_http\_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffering)

nginx 转发 upstream 数据到 client，如果上下游网速不匹配，典型的，upstream 发送数据的速度，一般远大于 client 接收数据的速度，就会导致数据在 nginx 积压，我们可以通过配置指令

```
proxy_buffering on|off
```

选择性的开启或者关闭 nginx 对 upstream 数据的缓冲功能

> 缓存和缓冲，概念是有所区分的：缓存表示数据被存了下来，存了起来，以备后续重复使用；缓冲则是表示对数据进行临时性的存储，一般就是因为上下游速度不匹配的时候，需要引入缓冲的机制，比如 nginx 将 upstream 数据临时性的缓冲在 nginx 本地，只是为了接下来把这些数据以匹配下游的速度发送出去，而并没有“永久性”存储这些数据。换个角度理解，缓冲是一次性的行为，而缓存就是为了复用。相应的，nginx 也明确区分了 `proxy_cache` 和 `proxy_buffering` 两个独立的功能。
>

<!-- more -->

进一步的，如果我们开启了 nginx buffering 功能，接下来就需要控制缓冲的策略，这里主要涉及三个 nginx 配置指令：

```
proxy_buffer_size
```

控制用来接收 upstream header 的单块 buffer 大小

```
proxy_buffers num size
```

控制用来接收 upstream body 的 buffer 大小和数量

```
proxy_temp_file_size 1024m
```

如果 upstream 速度过快，可能内存 buffer 很快就用完了，那还可以选择性地，将数据写入 nginx 本地文件中，进一步提高其缓冲能力（如果没有设置，则临时文件的默认大小为 1GB）

## 磁盘缓冲，存在的问题

这里有一个需要注意的坑，如果 nginx 缓冲写入的临时文件超过了配置的大小，就会停止从 upstream 接收数据，直到临时文件的缓冲数据全部发送完成！这就很容易导致一些 slow client 在下载大文件时出现中断的情况，而且就恰恰出现在 1GB 左右的时候

- 临时文件堆满，是因为上下游速度极度不匹配造成的
- 等待临时文件发送完，那势必会很久，因为 client 很慢
- 这段时间内，upstream 基本都会发生 send timeout，从而关闭和 nginx 的连接
- 进而引发 nginx 关闭和 client 的连接

也就出现了客户端观察到的，下载文件刚好到 1GB 左右中断的情况。

[Downloads stop after 1GB depending of network](https://trac.nginx.org/nginx/ticket/1472)

解决方式也很简单：关闭 nginx 的 disk buffer 功能

```
proxy_temp_file_size 0;
```

> nginx 一般都是和 upstream 同机房部署，甚至同机部署，所以 nginx 和 upstream 之间的连接和通信都是走内网的，而 nginx 和客户端的连接，则需要经过公网传输，所以上下游的速度天然存在不匹配的情况，而且通常都是 upstream 远快于 client
>
