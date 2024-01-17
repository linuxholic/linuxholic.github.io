---
title: XDP 尝鲜
date: 2024-01-17 11:39:44
tags:
    - xdp
    - bpf
---

# xdp-tutorial

[xdp-tutorial](https://github.com/xdp-project/xdp-tutorial) 是 XDP 官方提供的入门指南，还是比较权威的。

但是，有一说一，这个仓库的更新并不是很及时，不太能跟得上 XDP 自身的迭代速度。其中存在的一些问题，虽然可以在 issue 列表里搜到，但都是说上游已经解决，就是还没来得及更新这个 xdp-tutorial 仓库依赖的 libxdp 或者 libbpf 版本。

## lesson 回顾

xdp-tutorial 包含了三大类 lesson：

* basic
* packet
* advanced

看似循序渐进，实则不然；前两个部分 basic 和 packet 还是比较正常的，第三个部分 advanced 基本没啥可实操的，建议看看就好，不要逗留。另外，repo readme 里忽略了 tracing 系列的 lesson，算是第四个部分。讲道理，tracing 部分比 advanced 有用多了，建议上点心。

<!-- more -->

### basic04

这里对 reuse bpf map fd 的方式，不太行了，至少我自己反复测试都行不通；新版本的 libbpf 已经支持了 auto pin map，不建议在这块浪费时间

### packet03

这个 lesson 是整个 repo 里含金量最高的，只是有一些坑需要注意：

* checksum 的计算，类似 crc-16，但又不是 crc-16，建议直接看 solution，不要浪费时间在这类有点 magic 的逻辑上
* tx-port 出现在代理里，可是 TM 的根本没有用到，不要在意
* devmap 的使用方式，仅仅是为了这里教学的目的，一般没有这么用的；实际上，使用 bpf_redirect 而不是 bpf_redirect_map 会更好
* tcp redirect 会有莫名其妙的问题，省流解法：`ethtool -K eth0 tx off`；具体可以看这两个 [issue1](https://github.com/xdp-project/xdp-tutorial/issues/160#issuecomment-713532279) [issue2](https://github.com/xdp-project/xdp-tutorial/issues/160#issuecomment-722355532)

### advanced03

AF_XDP 的使用，主要还是对标 DPDK 的场景，放在 tutorial 里还是显得太重了，用起来有点复杂，有很多新的概念需要厘清，建议跳过。

## 依赖安装

按照 xdp-tutorial 的指引安装即可，但是从我的情况来看，至少还需要额外安装一下依赖：

```sh
sudo apt install clang-11 # apt 默认安装的 clang 版本太低了
sudo apt install libc6-dev-i386 # 这个说实话有点奇怪，有社区用户提 PR 希望加入依赖列表里，但是被拒了
sudo apt install m4 # 这个是编译必须的，讲道理，不知道为什么没有提到
sudo apt install ethtool # 操作测试环境需要用到，可能官方觉得大家默认都有吧
```

## Makefile

粗看可能东西比较多，实际只需要把握两个部分：

1. 编译 libbf 和 libxdp 两个依赖库
2. 编译所有的 lessons

对应的，在 lib 文件夹里有一个 Makefile 负责编译 libxdp 和 libbpf，每个 lesson 内都有一个单独的 Makefile 负责 lesson 内部的二进制编译。

在完成特定 lesson 的过程中，我们只需要在 lesson 目录内执行 make，即可更新对应的 assignment 结果，非常方便。

## common

这个目录内放了一些跨 lesson 会用到的公共函数，比如命令行参数解析，结构体定义，对 libbpf 和 libxdp 的进一步封装，等等。

还有就是，前置 lesson 完成后，对应的函数也会挪到这里，后置 lesson 会默认使用这里的函数，而不是我们在做 assignment 过程中在 lesson 内部完成的版本。

# 编码模式

XDP 是 eBPF 的一个特殊 hook 点，所以仍然遵循 bpf 程序的一般开发思路：内核侧和用户侧，独立开发，并通过 bpf map 进行协作和通信。

## 头文件

一开始可能会不太适应，需要同时编写内核侧和用户侧的代码，因为两者的**运行环境不同**，所以会有一些细节上需要注意的点，头文件首当其冲。

内核侧代码通常必须 include 以下头文件：

```c
#include <linux/bpf.h>
#include <bpf/bpf_helpers.h> /* 这里定义的实际是 bpf syscall number */
```

用户侧则是有所不同：

```c
#include <bpf/bpf.h>
#include <bpf/libbpf.h> /* 仔细看可以发现，这里和内核侧的 syscall number 是一样的（对同一个函数来说）*/
```

可以发现，用户侧需要依赖于 libbpf，这个库的开发在内核中进行，[github](https://github.com/libbpf/libbpf) 上有一份定期同步的 mirror 仓库。

## bpf map

关于 bpf map 的使用，这里主要提三个方面：reuse bpf map fd，common header，abort

复用 bpf map fd 很多时候是必须的，一般分两种情况处理：

1.  加载 eBPF 程序和运行用户侧程序，放在同一个主程序中，这样 fd 自然就可以共用了
2. 反之，如果二者分开的话，就需要用到 pin map 的机制了：加载程序将 bpf map pin 到一个固定的路径，比如 /sys/fs/bpf/xdp/xxxxxxxx 这样的，然后用户侧程序就可以 open 这个路径对应的 bpf map，从而拿到和 eBPF 内核侧程序对应的同一个 bpf map

还有一个常见的问题是，内核侧和用户侧需要对通信涉及的结构体达成一致，所以最便捷的方式就，定义一个 common header，其中包含了通过 bpf map 进行传递的结构体的具体定义，我们只需要在两侧都 include 这同一个 header，就可以实现精确匹配的读写操作。

最后一点就是程序 abort 对于 bpf map 的影响了。如果没有捕获一些导致程序退出的信号，程序直接异常终止了，那么遗留的 bpf map 就会残留在系统中，可以通过 bpftool 确认这一点。可惜的是，bpftool 没有提供 **删除整个 bpf map** 的操作，而是只能通过 bpftool map delete 删除 map 中指定的 key。目前实操下来看，重启机器可以清理掉这些残留的 bpf map，暂时还没有找到更好的方式。至于 bpftool 为什么不提供删除整个 bpf map 的操作，我也还不是很懂，了解这块儿的筒子们欢迎指出。

## bpf verifier

xdp-tutorial 中提到了几点，比如：

* packet data 边界检查
* 有界循环 unroll loop

第一点比较好理解，和我们通常见到的内存越界检查类似，因为 XDP hook 点提供的回调函数参数中，包含了 packet 首尾的信息，XDP 程序需要保证自己不会读写到 packet 范围之外的地址；第二点其实是 eBPF 的限制，XDP 程序中不能使用变长的循环体，只能使用编译时定长的循环，并通过 unroll loop 进行展开。

除了以上两点，我还在实践中碰到了更多的触发 bpf verifier 限制的情况。

* 结构体完全初始化之后，才能传给内核函数使用。比如 bpf map update 的时候，如果结构体有三个成员，但是只赋值了其中两个，因为栈变量的初始化是未定义的，或者说是随机的，所以未初始化的那个结构体成员就有问题了，bpf verifier 就会检测到这种情况，拒绝加载进入内核。

* 内核函数的返回值（如果有的话）需要进行显式检查和处理。比如 bpf map lookup 的时候，我们需要通过检查返回值确定是否查找成功；这其实还算是比较明显的情况，逻辑上也说的通，但其他一些只返回 int 的函数，我们也要严格注意，时刻记住检查返回值，并进行响应的处理，否则 bpf verifier 也会校验不通过。不过有一说一，这其实算是系统编程的最佳实践，对程序里涉及的每一个 syscall 都需要进行结果检查。

* 所有的变量声明都放到函数开头。内核侧程序需要使用 clang 编译，指定 arch target 为 bpf，因为 **eBPF 支持的只是 C 的一个子集**，而 gcc 不支持 bpf 后端，所以只能是 LLVM Clang；相反，用户侧程序则可以使用 gcc 进行编译。这点我们可以在 xdp-tutorial 的 Makefile 中看到：所有 lesson 中的 kern.c 使用 `CLANG:=clang` 进行编译，而 user.c 则使用 `CC:=gcc` 进行编译。



