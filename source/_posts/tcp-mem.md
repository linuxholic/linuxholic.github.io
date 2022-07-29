---
title: tcp-mem
date: 2022-03-24 10:20:06
tags:
    - tcp
---

在一般的网络调优中，调整 TCP 协议栈的内存使用是比较常见的一种方式，可以在两个层次进行：

* 内核层
* 应用层

### 内核层

内核层次对于 TCP 内存的使用主要涉及以下两类：

[man 7 tcp](https://man7.org/linux/man-pages/man7/tcp.7.html)

```
tcp_mem
tcp_rmem
tcp_wmem
```

可以通过 `sysctl` 接口进行查看和更改：

```
net.ipv4.tcp_mem = 22152        29538   44304
net.ipv4.tcp_rmem = 4096        131072  6291456
net.ipv4.tcp_wmem = 4096        16384   4194304
```

<!-- more -->

#### tcp_mem

整个机器层面，TCP 协议栈占用的总内存限制，以内存 page 作为统计单位：

* low：TCP 协议栈内存总量低于这个阈值，则 TCP 不主动对自身内存分配进行限制；
* pressure：一旦高于这个阈值，TCP 开始节制自身的内存分配行为（进入 memory pressure 状态），直至降至 low 以下；
* high：针对 TCP 协议栈的终极限制，其他任何规则都不允许超过这个限制。

#### tcp_rmem

限制 TCP 单个 receive buffer 的大小，以下取值限制 TCP 动态调整 buffer size 的范围：

* min：只在 TCP 协议栈进入 memory pressure 模式时生效，只允许低于这个值的内存分配行为；
* default：默认大小，这里的取值会覆盖 net.core.rmem_default 的取值，作为 socket receive buffer 的最终默认取值
* max：动态调整的 buffer 最大值


#### tcp_wmem

和 `tcp_rmem` 雷同

[man 7 socket](https://man7.org/linux/man-pages/man7/socket.7.html)

```
rmem_default
rmem_max
wmem_default
wmem_max
```

可以通过 `sysctl` 接口进行查看和更改：

```
net.core.rmem_default = 212992
net.core.rmem_max = 212992
net.core.wmem_default = 212992
net.core.wmem_max = 212992
```

#### rmem_default

socket receive buffer 的默认大小，可能会被 tcm_rmem 中的第二个 defalut 值覆盖

#### rmem_max

限制应用层可以手动设置的最大 receive buffer 大小，即通过 `setsockopt(2)` 的 SO_RCVBUF 选项进行设置的最大允许值。

#### wmem_default

socket write buffer 的默认大小，可能会被 tcm_wmem 中的第二个 defalut 值覆盖

#### wmem_max

限制应用层可以手动设置的最大 write buffer 大小，即通过 `setsockopt(2)` 的 SO_SNDBUF 选项进行设置时的最大允许值。


### 应用层

针对每个 socket 都可以手动设置其 buffer 大小：

* receive buffer：`setsockopt(2)` 的 SO_RCVBUF 选项
* write buffer：`setsockopt(2)` 的 SO_SNDBUF 选项

一般来说，为了最大化网络吞吐，普遍倾向于增加 socket buffer 的大小。

### 总结

基于上述分析，可以发现，针对 socket buffer 的调整方式只有两种：手动和自动

* 手动：在应用层代码中，通过 setsockopt 系统调用进行设置；
* 自动：通过 sysctl 配置内核层参数，从而影响 TCP 协议栈的自动调整策略。

一旦在应用层进行了手动设置，那么 socket buffer 的大小就是固定的了，不再受到内核 TCP 协议栈自动调整的控制。

### 监控

对于机器层面来说，TCP 协议栈的内存使用可能会过载，这个时候可以通过 `/proc/net/netstat` 进行监控

* `TCPMemoryPressures`: 表示 TCP 协议栈进入 memory pressure 状态的次数

一旦发现这个指标出现持续性的增长，说明 TCP 协议栈的缓冲能力已经见顶，可能导致一定程度的丢包，那么下面这个指标：

* `PruneCalled`: 表示应用层 socket buffer 满，数据包被 TCP 协议栈丢弃的次数

也会开始同步增大，这里的丢包极有可能进一步引发其他业务层面的问题。

结合前述的分析，我们可以适当调整 tcp_mem 的 pressure 取值，一定程度上可以缓解 TCP 协议栈的丢包情况。

PS：TCP 协议栈的缓冲能力过载，根本原因还是整个数据链路的处理存在瓶颈，需要精确定位并解决，否则针对 tcp_mem 的调整只能是治标不治本，就算调再大，也会有被填满的时候。
