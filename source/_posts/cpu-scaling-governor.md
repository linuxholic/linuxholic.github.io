---
title: 震惊！服务器 CPU 也可以调频？
date: 2022-09-09 14:49:42
tags:
    - cpu
    - tcp
---

# 背景

缓存服务的吞吐出现明显下降，更具体一点，**命中状态下的吐出带宽** 偏低。

# 网络瓶颈

因为数据是从本地吐出的，所以最开始怀疑是网络的因素，通过小米监控查看了 tcp 相关的指标，比如：

* TcpExt.PruneCalled：TCP 协议层主动丢包
* TcpExt.TCPTimeouts：TCP 传输超时

相比其他负载差不多的机器，这些指标都要显著偏高。结合以往经验，TCP 的这些指标异常也有可能只是表象，是其他原因导致了这些异常，我们不能简单的将这些异常直接归为根本原因。

<!-- more -->

紧接着查看了更为底层的 `/proc/net/softnet_stat` 数据：

```
02460248 00000000 000000af 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
012e9a61 00000000 00000d44 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000001
01aae54c 00000000 00000092 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000002
```

这个数据源在之前的文章 {% post_link networking-perf %} 也有提及，这里主要关注其中的第二列数据：

> time_squeeze：在一个 loop 中处理 packet 时被打断的次数

正常情况下，如果机器的网络包处理能力充足，这列数据应该保持不变（当然，绝对值可以很大，因为这个数据是从机器启动开始一直累加的，所以历史数据不代表当前情况）；

通过反复查看，线上机器的这列数据在 **持续的增加**，说明包处理存在瓶颈。

接下来有两个排查方向：

1. 一个 loop 内可以处理的包数量阈值偏低，导致频繁被打断
2. 一个 loop 内处理超时，达到了时间阈值，导致被打断

## netdev_budget

通过增加 net.core.netdev_budget 系统配置项的值，我们可以增加一个 loop 内处理 packet 的数量上限

```
sysctl net.core.netdev_budget
sysctl -w net.core.netdev_budget=4096
```

通过 sysctl 调高该值之后，观察业务指标基本没有变化 (╯‵□′)╯︵┻━┻

如果不是因为一个 loop 内处理包数量达到了限制，就只能是因为触发了时长限制；也就是一个 loop 内即使只处理了并不算多的 packet，也已经耗尽了固定 `jiffies` 时长，所以仍然会被打断，因为是 CPU 自身处理慢。

## cpu bench

为了验证这个猜想，我们使用一个简单的 python 脚本对机器 CPU 进行压测对比

```python
#!/usr/bin/env python
#-*- coding:utf8 -*-

import os
import time
import affinity
import multiprocessing


def bench_cpu(idx):

    print 'benchmark cpu%d' % idx

    pid = os.getpid()
    affinity.set_process_affinity_mask(pid, 2**idx)

    res = 0
    start = time.time()

    for idx in xrange(10000000):
        res = res + idx

    end = time.time()

    return (end-start)


def main():

    result_list = []

    for idx in range(multiprocessing.cpu_count()):
        spent_time = bench_cpu(idx)
        result_list.append((idx, spent_time))

    print
    print 'benchmark over.'
    print

    for result in result_list:
        idx, spent_time = result
        print 'cpu%d: %f' % (idx, spent_time)


if __name__ == '__main__':
    main()

```

可以发现，问题机器相比其他机器，压测脚本执行时间明显更长

```
cpu0: 0.315132 (node-1)
cpu0: 1.283741 (node-2)
```

性能相差多达四倍！事情开始变得清晰了：

- CPU 处理慢，也就导致了 packet 延时高（等待进入第二轮 loop 才能被处理），进而出现各种 tcp timeout...
- CPU 慢，也会导致各层 buffer 占满，最终通过某些指标暴露出来，比如 tcp memory pressure，tcp prune called 等等；因为在各层队列滞留，也进一步加剧了 packet 的延迟情况

一环扣一环，道理是相通的；如果不是 CPU，而是其他的某个地方慢了，其实也就是无非这两种结果：延迟变高 & buffer 溢出；整个软件系统，其实就是一层层的队列堆出来的，每层封装，都对应其自身的队列，从而才能达到解耦的设计目的。

## tcp memory

为了进一步验证 tcp buffer 的猜想，我们调整了机器的 tcp memory 相关设置：

```
net.core.rmem_default = 212992
net.core.rmem_max = 212992
net.core.wmem_default = 212992
net.core.wmem_max = 212992
net.ipv4.tcp_mem = 22152        29538   44304
net.ipv4.tcp_rmem = 4096        131072  6291456
net.ipv4.tcp_wmem = 4096        16384   4194304
```

这其中配置的含义，也有在之前的 {% post_link tcp-mem %} 中提到过，此处不再赘述。

调整生效之后，观察到业务指标明显改善，基本符合之前的预期 ：CPU 慢会导致 packet 在各个层级堆积，如果某个层级的 buffer 能力不足，就会开始拖后腿，并通过一定的监控指标暴露出来；tcp 协议栈表现出来的 memory pressure，就说明堆积已经超过了其当前的 buffer 能力，所以可以通过增加 tcp mem 来一定程度上缓解当前的问题（但再大的内存也还是会被 slow cpu 来不及处理而填满）

# CPU 频率

虽然基本定位到是 CPU 的原因，但是更换硬件必然很慢，所以还是继续尝试看下，是否存在一些可以立即调整的软限制。

通过 `cpuinfo` 查看  CPU 相关信息：

```
cat /proc/cpuinfo | grep "model name"
```

原本想看下 CPU 型号信息，看问题机器的是否不太一样，结果却发现了其他有意思的地方

```
model name      : Intel(R) Xeon(R) CPU E5-2620 v4 @ 2.10GHz
```

问题机器的 CPU 主频（最后一个字段）竟然还高于其他机器！

## Inter SpeedStep

这和我们先前的猜想背道而驰，不太对。CPU 频率代表了其执行 instruction 的速度，频率越高，单位时间内可以执行更多的指令，也就意味着性能更高。带着这个疑点，进一步查看了代表 CPU 实际运行频率的指标：

```
cat /proc/cpuinfo | grep "cpu MHz"
```

发现两个值得注意的地方

1. 问题机器的 CPU 运行频率 **一直在变化**
2. 问题机器的 CPU 运行频率明显 **低于其他正常机器**

基于 CPU 频率变化这一事实，我们也就很快发现了 Inter [SpeedStep](https://en.wikipedia.org/wiki/SpeedStep) 这一 CPU 调频机制。

## 节能模式 vs 性能模式

Inter CPU 可以设置不同的频率模式，以适应不同的场景需求；对应的，内核也提供了相应的接口进行调整

```
cat /sys/devices/system/cpu/cpu11/cpufreq/scaling_available_governors
cat /sys/devices/system/cpu/cpu11/cpufreq/scaling_governor
cat /sys/devices/system/cpu/cpu11/cpufreq/scaling_cur_freq
cat /sys/devices/system/cpu/cpu11/cpufreq/scaling_max_freq
cat /sys/devices/system/cpu/cpu11/cpufreq/scaling_min_freq
```

linux kernel 目前支持的模式主要有 performance，powersave 等几种，具体可以参考 [内核文档](https://www.kernel.org/doc/html/latest/admin-guide/pm/cpufreq.html#generic-scaling-governors)

其中，在 powersave，也即 节能模式 下，就会导致 CPU 开启 SpeedStep 功能，动态调整 CPU 频率；本意是为了降低功耗，讲道理，这种模式一般只适用于桌面场景，而对于服务器场景，都是需要 performance 模式的，才能最大化服务器的利用效率。

# 结论

问题机器开启了 CPU `powersave` 节能模式，导致 CPU 运行频率持续偏低，从而引发业务上网络处理出现瓶颈。

解决方式就很简单了，直接写入 `scaling_governor` 的 sys 接口，更改模式为 `performance` 即可，调整之后立竿见影：

* 业务指标恢复到和其他正常机器一致的水平
* TCP 相关指标也恢复正常
  * `TcpExt.PruneCalled` 主动丢包，完全消失
  * `TcpExt.TCPTimeouts` 超时基本消失，和其他机器同一水平``

PS：无图无真相，附带几张小米监控的截图

> 业务指标

![](/images/hit-speed.png)

> TcpExt.PruneCalled

![](/images/TcpPruned.png)

> TcpExt.TCPTimeouts

![](/images/TcpTimeouts.png)
