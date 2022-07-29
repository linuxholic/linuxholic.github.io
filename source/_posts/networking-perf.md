---
title: networking-perf
date: 2019-03-30 16:43:29
tags:
    - network
---

# 主题

* 基于 linux 内核，机器是如何接受数据包的？
* 数据包从网卡到应用层会经过若干个组件，如何针对性的进行监控和调优？

# 基本原则

* 理论上，只需要在网络栈的各个层次监控 **丢包率**，就可以很快的定位系统当前的瓶颈；
* 这个时候如果只是参考网上的一份”最优“ sysctl 配置，往往不能达到很好的效果；
* 调优的前提是，我们需要有清晰的可监控指标来验证实际的效果。

# 概述

数据包从抵达网卡开始，一路到达套接字的 receive buffer：

* 驱动加载和初始化
* 数据包到达网卡控制器(NIC)
* 数据包被复制到内核空间（ DMA -> ring buffer ）
* 产生硬件中断，通知系统数据可读
* 驱动调用 NAPI 激活 poll 循环（如果该循环处于休眠状态）
* ksoftirpd 调用驱动注册的 poll 函数，读取 ring buffer 中的数据包
* ring buffer 对应的内存区域被解除映射（ memory region unmapped ）
* 数据包被封装为 `skb` 结构体，准备传递到上层协议栈
* 如果开启网卡多队列，数据帧会被负载均衡到多个 CPU 进行处理
* 数据帧经由队列，递交上层协议栈
* 协议栈处理（ IP -> UDP/TCP ）
* 数据被填充到套接字的 receive buffer

<!-- more -->

# 详述

我们有必要先了解驱动层的工作机制，这样才能更好的理解后半部分的内容

## 网卡驱动

### PCI 注册

设备加载时，内核会调用驱动通过 `module_init` 注册的初始化函数

```c
/**
 *  igb_init_module - Driver Registration Routine
 *
 *  igb_init_module is the first routine called when the driver is
 *  loaded. All it does is register with the PCI subsystem.
 **/
static int __init igb_init_module(void)
{
  int ret;
  pr_info("%s - version %s\n", igb_driver_string, igb_driver_version);
  pr_info("%s\n", igb_copyright);

  /* ... */

  ret = pci_register_driver(&igb_driver);
  return ret;
}

module_init(igb_init_module);
```

其中 `pci_register_driver` 具体实现了注册到内核  **PCI子系统** 的任务

```c
static struct pci_driver igb_driver = {
  .name     = igb_driver_name,
  .id_table = igb_pci_tbl,
  .probe    = igb_probe,
  .remove   = igb_remove,

  /* ... */
};
```

其中：

* `igb_pci_tbl` 定义了一个 table，包含 driver 可以控制的 Device ID 列表；
* `igb_probe` 执行一些前期准备工作（激活设备，请求 DMA，注册各种控制函数等）

### 设备初始化

`igb_probe` 除了一些 PCI 相关的工作，更重要的是网络相关的内容：

* 注册结构体 `net_device_ops`
* 注册 `ethtool` 的一系列操作
* 从 NIC 读取 MAC 地址
* 设置 `net_device` 的各种功能标志位
* 等等

`net_device_ops` 包含一系列函数指针，用来控制网卡的各种动作

```c
static const struct net_device_ops igb_netdev_ops = {
  .ndo_open               = igb_open,
  .ndo_stop               = igb_close,
  .ndo_start_xmit         = igb_xmit_frame,
  .ndo_get_stats64        = igb_get_stats64,
  .ndo_set_rx_mode        = igb_set_rx_mode,
  .ndo_set_mac_address    = igb_set_mac,
  .ndo_change_mtu         = igb_change_mtu,
  .ndo_do_ioctl           = igb_ioctl,

  /* ... */
```

`ethtool` 对应一个命令行工具，可以用来获取和设置网卡的各项参数配置；需要注意的是，`ethtool` 工具并不是直接和网卡驱动交互，而是通过 `ioctl` 经由内核来和网卡交互。

#### NOTE：IRQ 和 NAPI

数据包到达内核空间之后，网卡需要通过 irq 通知内核；在网络流量很大的情况下，就会导致大量的中断产生，CPU 忙于处理中断，就可能导致用户任务被延迟处理；为了减少 irq 产生的次数，引入了 NAPI 的机制，允许驱动注册 poll 函数来读取数据帧。下面是 NAPI 的一般流程：

1. 驱动开启 NAPI，但是一开始处于未激活状态
2. 数据包到达并被 NIC 通过 DMA 拷贝到内存
3. NIC 产生中断，触发驱动注册的中断处理函数
4. 驱动通过 softirq 激活 NAPI 子系统；这时会有专门的内核线程开始收割数据帧
5. 驱动关闭 NIC 中断，防止 NAPI 子系统在收割数据帧时被再次中断
6. 数据帧处理结束之后，NAPI 子系统挂起，并重新激活 NIC 中断信号
7. 重复执行上述第二步

可以看出，通过驱动注册的 poll 函数，NAPI 能够在一次性处理大量数据帧，而不是中断一次处理一次，大大提高了效率。poll 函数注册到 NAPI 子系统，通常发生在驱动初始化的阶段。

PS：多队列的情况下，每个 rx / tx 队列都对应一个 poll 循环，通过参数 `struct napi_struct` 区分不同的队列（ poll 函数是同一个，通过参数控制作用到不同的队列上）。

### 网卡启动

`net_device_ops->ndo_open` 对应了网卡启动时调用的函数，比如当我们执行 `ifconfig eth0 up` 的时候。

1. 分配 rx / tx 队列内存
2. 开启 NAPI
3. 注册中断处理函数
4. 开启中断
5. 等等

大部分现代 NIC 都是通过 DMA 直接将数据帧写入到内核空间，为了达到这一目的，NIC 使用了类似队列的模型，底层数据结构基于环形buffer，也就是我们通常所说的 `ring buffer`。所以驱动需要向内核申请内存，并把相应的内存地址通知到网卡，这样 NIC 就可以知道数据帧写入到哪里。

考虑到单个 `ring buffer` 只能被单个 CPU 处理，并且大小有限，现代 NIC 引入了多队列的机制，可以同时写入多个内存区域，从而使用多个 CPU 并行处理数据帧。我们可以通过 `ethtool` 工具查看和修改网卡多队列的相关设置：

* 调整队列的数目以及大小
* 调整队列的权重
* 调整用于分发数据帧到多个队列的哈希函数

虽然 poll 函数是在驱动初始化的时候注册的，但是直到网卡启动才会为每个队列开启 NAPI。

注册中断处理函数时需要注意，网卡支持的中断产生方式有多种：MSI-X, MSI, legacy interrupt。驱动需要逐个尝试，设置对应的处理函数。这里优先选择 MSI-X，尤其是多队列的情况，因为可以针对每个队列设置独立的硬件中断，从而充分利用多 CPU 并行处理。

* MSI/MSI-X: 使用 in-band 数据模拟中断，无需额外芯片引脚
* legacy interrupt：使用专用中断引脚，触发 out-of-band 控制信号

激活中断的方式取决于硬件，一般是通过写入网卡的特殊寄存器实现：

```c
static void igb_irq_enable(struct igb_adapter *adapter)
{

  /* ... */

    wr32(E1000_IMS, IMS_ENABLE_MASK | E1000_IMS_DRSTA);
    wr32(E1000_IAM, IMS_ENABLE_MASK | E1000_IMS_DRSTA);

  /* ... */
}
```

至此，网卡已经就绪，随时可以接收网络数据。

### 网卡监控

监控网卡有多种方式，各自提供不同的粒度和复杂度。

* `ethtool -S`
* `sysfs`
* `/proc/net/dev`

`ethtool` 的使用方式如下：

```sh
$ sudo ethtool -S eth0
NIC statistics:
     rx_packets: 597028087
     tx_packets: 5924278060
     rx_bytes: 112643393747
     tx_bytes: 990080156714
     rx_broadcast: 96
     tx_broadcast: 116
     rx_multicast: 20294528
     ....
```

对上述数据进行监控其实比较困难。虽然拿到这些数据很简单，但是没有统一的标准对这些字段进行定义。不同的驱动程序，甚至相同驱动程序的不同版本，对同样含义的指标可能有不一样的字段名称。

一般来说，我们只需要关注类似 `drop`,`buffer`,`miss` 这样的指标，但是我们仍然需要通过驱动程序代码来确定这些指标的真实来历。有些指标经过了软件层面的累加（比如内存不足发生的次数），有些指标则是直接从网卡寄存器读取出来的。对于这样的寄存器值，我们还需要参考网卡的 data sheet 来确定其真实含义；因为 `ethtool` 给出的很多指标字段具有误导性。

`sysfs` 提供的指标相比 `ethtool` 更高阶一些，比如这样：

```sh
$ cat /sys/class/net/eth0/statistics/rx_dropped
2
```

不同的指标被分割放到了不同的文件里，比如 `collisions`,`rx_dropped`,`rx_errors` 等等；

类似的，这些指标也取决于具体的驱动程序，每个指标的含义，什么时候累加，来自哪里等等；比如同样一种类型的错误，有些驱动算作 `drop`，有些驱动则算到 `miss` 里。

`/proc/net/dev` 提供的数据更为概括性，类似这样：

```sh
$ cat /proc/net/dev
Inter-|   Receive                                                |  Transmit
 face |bytes    packets errs drop fifo frame compressed multicast|bytes    packets errs drop fifo colls carrier compressed
  eth0: 110346752214 597737500    0    2    0     0          0  20963860 990024805984 6066582604    0    0    0     0       0          0
    lo: 428349463836 1579868535    0    0    0     0          0         0 428349463836 1579868535    0    0    0     0       0          0
```

这个文件提供的指标是 `sysfs` 的子集，可以提供一个大概的参考；同样的，我们需要通过驱动源码来确定这些值累加的时机，地点和原因。

### 网卡调优

使用 `ethtool` 工具，我们可以：

* 检查当前使用的队列数目
* 调整队列数量
* 调整队列大小
* 调整队列权重
* 调整哈希字段
* `ntuple filtering` 控制特定数据帧到指定队列

举个例子，对于一个监听 80 端口的 web server 来说：

* web server 绑定到了 CPU2
* 某个网卡接收队列的中断信号，被指定到 CPU2 处理
* 目的端口为 80 的 tcp 数据帧被 *过滤* 到 CPU2
* 这样的话，从数据帧到达网卡开始，一直到应用层，80 端口的所有流量就都被 CPU2 处理了
* 缓存命中率和网络延迟都会有改善

## 软中断

* 内核延时任务机制
* 内核线程 `ksoftirqd`
* 监控 `/proc/softirqs`

概括来说，`softirq` 用来在中断处理函数之外执行代码。考虑到在中断处理函数执行期间，一般会屏蔽掉中断信号；那么一旦中断处理耗时越长，中断信号丢失的概率就会越大。所以，对于任何需要长时间运行的代码，有必要延迟到中断处理环境之外执行，从而让中断处理尽快结束，并重新激活中断信号。

整个 `softirq` 子系统由一系列内核线程组成，每个 CPU 对应一个内核线程。在 `top` 命令里，我们可以发现有类似 `ksoftirqd/0` 这样的内核线程，就对应于 CPU0 的 `softirq` 线程。其他内核子系统（比如网络）可以通过 `open_softirq` 注册处理函数，也就达到了把任务交给 `softirq` 执行的目的。

`ksoftirqd` 线程在内核启动期间产生：

```c
static struct smp_hotplug_thread softirq_threads = {
  .store              = &ksoftirqd,
  .thread_should_run  = ksoftirqd_should_run,
  .thread_fn          = run_ksoftirqd,
  .thread_comm        = "ksoftirqd/%u",
};

static __init int spawn_ksoftirqd(void)
{
  register_cpu_notifier(&cpu_nfb);

  BUG_ON(smpboot_register_percpu_thread(&softirq_threads));

  return 0;
}
early_initcall(spawn_ksoftirqd);
```

可以看出，针对每个 CPU 产生独立的线程，并且线程名根据 cpu 数目编号；另外还注册了两个执行函数 `ksoftirqd_should_run` 和 `run_ksoftirqd`，前者用来判断是否有待处理的软中断，如果有，则执行后者。

`run_ksoftirqd` 实际是对 `__do_softirq` 的简单封装，后者执行了一些值得注意的操作：

* 检查是否有软中断等待处理
* 记录执行时间 - 用来计算耗时？
* 累加执行次数
* 执行软中断处理函数（通过 `open_softirq` 注册）

PS：通过工具查看 CPU 使用率的时候，`softirq`/`si` 部分就对应于延迟任务执行的耗时。

通过 `/proc/softirqs` 文件，我们可以监控各类事件触发软中断的频率：

```sh
$ cat /proc/softirqs
                    CPU0       CPU1       CPU2       CPU3
          HI:          0          0          0          0
       TIMER: 2831512516 1337085411 1103326083 1423923272
      NET_TX:   15774435     779806     733217     749512
      NET_RX: 1671622615 1257853535 2088429526 2674732223
       BLOCK: 1800253852    1466177    1791366     634534
BLOCK_IOPOLL:          0          0          0          0
     TASKLET:         25          0          0          0
       SCHED: 2642378225 1711756029  629040543  682215771
     HRTIMER:    2547911    2046898    1558136    1521176
         RCU: 2056528783 4231862865 3545088730  844379888
```

目前我们只需要关注 `NET_RX` 这一行，对应网络收包触发的软中断。可以看出，不同的 CPU 处理的软中断数量并不一样，如果相差悬殊，我们可以通过网卡多队列进行调优。

## Linux 网络栈

* 初始化
* 数据接收
* 数据处理
* GRO - 数据包合并，需要应用层介入
	* 提供数据包合并的前提条件
	* 以及停止合并或者合并完成的边界条件

### 初始化

执行初始化函数 `net_dev_init`，基于当前 cpu 数目创建一系列 `struct softnet_data`。这些结构体保存了很多重要的指针：

* NAPI 结构体，用来注册到当前 CPU
* backlog 队列
* 处理权重 - `processing weight`
* LRO 结构体
* RPS 配置
* 等等

此外，初始化函数还分别注册了数据接收和发送的软中断处理函数：

```c
static int __init net_dev_init(void)
{
  /* ... */

  open_softirq(NET_TX_SOFTIRQ, net_tx_action);
  open_softirq(NET_RX_SOFTIRQ, net_rx_action);

 /* ... */
}
```

### 数据接收

假设 rx queue 有足够的可用描述符，数据帧通过 DMA 写入到 RAM，网卡紧接着触发了中断信号（内核会为每个设备分配一个中断号；对于 MSI-X 则是绑定到 rx queue 的中断号）

#### 中断处理函数

一般来说，中断处理函数应该尽量延迟任务到中断环境之外执行，因为中断处理期间会屏蔽后续的中断信号。
下面这个是 MSI-X 的中断处理函数：

```c
static irqreturn_t igb_msix_ring(int irq, void *data)
{
  struct igb_q_vector *q_vector = data;

  /* Write the ITR value calculated from the previous interrupt. */
  igb_write_itr(q_vector);

  napi_schedule(&q_vector->napi);

  return IRQ_HANDLED;
}
```

够短吧！这里只做了两步操作：第一步更新硬件寄存器，第二步激活 NAPI 处理循环。

#### NAPI `napi_schedule`

引入 NAPI 的唯一目的，就是在无需 NIC 中断触发的情况”收割“数据帧。之前提到过，NAPI 的 poll 循环是被 NIC 的硬件中断 **激活** 的；也就是说，NAPI 一直处于 stand by 的模式，直到被首个数据帧触发的硬件中断启动。另外，某些情况下 NAPI 也会被关闭，并等待下一次 NIC 中断触发。

`napi_schedule` 定义在头文件中，只是对 `__napi_schedule` 的简单封装：

```c
/**
 * __napi_schedule - schedule for receive
 * @n: entry to schedule
 *
 * The entry's receive function will be scheduled to run
 */
void __napi_schedule(struct napi_struct *n)
{
  unsigned long flags;

  local_irq_save(flags);
  ____napi_schedule(&__get_cpu_var(softnet_data), n);
  local_irq_restore(flags);
}
EXPORT_SYMBOL(__napi_schedule);
```

需要关注的是 `__get_cpu_var` 这个函数，拿到了注册到当前 CPU 上的 `softnet_data` 结构体；这个结构体和 `struct napi_struct` 一起传给了 `____napi_schedule` 函数：

```c
/* Called with irq disabled */
static inline void ____napi_schedule(struct softnet_data *sd,
                                     struct napi_struct *napi)
{
  list_add_tail(&napi->poll_list, &sd->poll_list);
  __raise_softirq_irqoff(NET_RX_SOFTIRQ);
}
```

这里主要做了两个事情：

1. 把 `struct napi_struct` 追加到 `softnet_data` 的 `pool_list` 列表；
2. 触发 `NET_RX_SOFTIRQ` 软中断

对于第一点，`struct napi_struct` 存放了网卡队列对应的内存地址等信息，所以需要传递过去；另外，需要一个列表的原因是，可能存在多个网卡，或者网卡多队列的情况；对于第二点，对应处理函数 `net_rx_action` 最终会调用驱动程序当初注册的 `poll` 函数，从而处理数据。

> 为什么需要绑定中断处理到特定 CPU ？

目前为止看到的延迟任务执行都用到了当前 CPU 的数据结构，也就是说，中断处理函数交出去的任务，都绑定到了当前 CPU 上，后续这些任务也就仍然在这个 CPU 上执行。所以通过指定处理中断的 CPU，也就确保了后续 `softirq` 在相同的 CPU 上执行。

#### 监控数据接收

需要注意的是，硬件中断的变化并不能完全反应数据帧处理的变化，因为驱动程序一般会在 NAPI 执行期间屏蔽硬件中断。

```sh
$ cat /proc/interrupts
            CPU0       CPU1       CPU2       CPU3
   0:         46          0          0          0 IR-IO-APIC-edge      timer
   1:          3          0          0          0 IR-IO-APIC-edge      i8042
  30: 3361234770          0          0          0 IR-IO-APIC-fasteoi   aacraid
  64:          0          0          0          0 DMAR_MSI-edge      dmar0
  65:          1          0          0          0 IR-PCI-MSI-edge      eth0
  66:  863649703          0          0          0 IR-PCI-MSI-edge      eth0-TxRx-0
  67:  986285573          0          0          0 IR-PCI-MSI-edge      eth0-TxRx-1
  68:         45          0          0          0 IR-PCI-MSI-edge      eth0-TxRx-2
  69:        394          0          0          0 IR-PCI-MSI-edge      eth0-TxRx-3
 NMI:    9729927    4008190    3068645    3375402  Non-maskable interrupts
 LOC: 2913290785 1585321306 1495872829 1803524526  Local timer interrupts
```

除了 NAPI 的影响，硬件自身中断合并的机制也会影响这里的数据变化。所以这个文件表现出的中断数量和变化频率，并不能反映实际的数据帧接收量和处理量。为了得到更全面的指标，我们还需要进一步监控 `/proc/softirqs` 以及 `/proc` 下的其他文件。

那这个文件的还有啥用？可以用来验证网卡多队列是否生效，每个队列是不是被独立的 CPU 处理。

#### 调优数据接收

* 中断合并
* 中断亲和性

> Interrupt coalescing

中断合并用来阻止发送中断信号到 CPU，直到积累了一定数量的待处理事件，以此避免中断风暴，也可以一定程度上改善吞吐量和延迟。`ethtool` 工具对此也提供了支持：

```sh
$ sudo ethtool -c eth0
Coalesce parameters for eth0:
Adaptive RX: off  TX: off
stats-block-usecs: 0
sample-interval: 0
pkt-rate-low: 0
pkt-rate-high: 0
...
```

友情提醒：并不是每个驱动都会实现 `ethtool` 的全部设置项。对于驱动不支持的配置项，对应的配置值都会被直接忽略掉，也就不会起作用咯。

值得一提的一个选项是，自适应中断合并。这个功能一般在硬件层面实现，但需要驱动程序的配合才能真正启用。启用这个功能的效果很诱人：流量低峰期降低延时，流量高峰期提升吞吐。

```sh
$ sudo ethtool -C eth0 adaptive-rx on
```

除了直接开启这个功能，也可以实现其他更精细的控制：

* `rx-usecs`：数据帧到达后，延迟多长时间产生中断信号，单位微妙
* `rx-frames`：触发中断前积累数据帧的最大个数
* `rx-usecs-irq`：如果有中断处理正在执行，当前中断延迟多久送达 CPU
* `rx-frames-irq`：如果有中断处理正在执行，最多积累多少个数据帧
* 等等

参考：[include/uapi/linux/ethtool.h](https://github.com/torvalds/linux/blob/v3.13/include/uapi/linux/ethtool.h#L184-L255)

> IRQ affinities

支持网卡多队列的情况下，可以通过设置 CPU 亲和性提高数据本地性。具体来说就是，我们可以指定哪些中断交给哪些 CPU 处理。调整之前需要首先检查两个事情：

* 是不是有守护进程 `irqbalance` 在运行？
* 检查 `/proc/interrupts` 获取每个网卡队列对应的中断号

最后，我们修改系统文件 `/proc/irq/IRQ_NUMBER/smp_affinity` 来指定 CPU：

```sh
$ sudo bash -c 'echo 1 > /proc/irq/8/smp_affinity'
```

其中的值为十六进制的位掩码，上面表示指定 CPU 0 处理中断号 8。

### 数据处理

一旦上述的软中断过程发现有待处理的软中断信号，就会开始处理并最终执行到 `net_rx_action`

#### `net_rx_action` 处理循环

核心工作就是处理位于 DMA 内存区域的数据帧；遍历挂在当前 CPU 上的 NAPI 结构体列表，出队并逐个处理。整个循环会限制整体的工作量，以及驱动注册的 `poll` 函数的可执行时长，基于下面两点：

1. 随时检查工作量相关的 `budget`
2. 检查当前已经执行的时长

```sh
  while (!list_empty(&sd->poll_list)) {
    struct napi_struct *n;
    int work, weight;

    /* If softirq window is exhausted then punt.
     * Allow this to run for 2 jiffies since which will allow
     * an average latency of 1.5/HZ.
     */
    if (unlikely(budget <= 0 || time_after_eq(jiffies, time_limit)))
      goto softnet_break;
```

有了这样的限制，就可以阻止 CPU 被网络包处理独占。

重点是 `budget` 生效的范围：针对的是当前 CPU 上的所有 NAPI 结构体列表。如果机器的 CPU 数目不足以承担全部的网卡队列，就会导致多个队列只能由同一个 CPU 处理，消耗同一个 `budget`；这种情况下，可以考虑增大 `budget` 来更快的处理数据，虽然会增加 CPU 使用率的 `si` 部分，但是有助于降低数据帧处理的延时。

#### NAPI `poll` 函数和权重

之前提到每个 NAPI 结构体都赋予了一个权重，目前是硬编码为 64，现在看下这个值是如何起作用的：

```c
weight = n->weight;

work = 0;
if (test_bit(NAPI_STATE_SCHED, &n->state)) {
        work = n->poll(n, weight);
        trace_napi_poll(n);
}

WARN_ON_ONCE(work > weight);

budget -= work;
```

权重值被传递给 `poll` 函数，作为处理数据帧的上限，而实际的处理量通过返回值保存在 `work` 变量中，并最终从 `budget` 里扣除。

假设：

1. 驱动注册的权重为 64（这个驱动硬编码进去的）
2. `budget` 保持默认值 300

那么整个软中断处理会在下面两种情况下停止：

* `poll` 函数最多被调用五次（如果没有更多数据，可能用不了五次）
* 时长达到或者超过了 2 jiffies

#### NAPI 和网卡驱动的约定

关于这两者之前的约定，很重要的一点是：什么情况下关闭 NAPI？

* 如果驱动的 `poll` 函数用完了权重 `weight`，那么不能更改 NAPI 状态，交给 `net_rx_action` 处理
* 如果没有用完，驱动必须关闭 NAPI，等待下一次硬中断触发

#### 结束 `net_rx_action` 循环

结束处理循环之前，首先就是处理上述约定的第一种情况：把 NAPI 结构体移动到当前 CPU 的队尾，这样就可以让 CPU 紧接着处理队列上的下一个 NAPI 结构体。

达到循环结束的限制条件之后，会跳转到下面的地方执行：

```c
softnet_break:
  sd->time_squeeze++;
  __raise_softirq_irqoff(NET_RX_SOFTIRQ);
  goto out;
```

结构体 `softnet_data` 递增了统计数据 `time_squeeze`，并关闭了软中断 `NET_RX_SOFTIRQ`。

这里的 `time_squeeze` 衡量的是 `net_rx_action` 被迫中断的次数；虽然有更多的数据帧等待处理，但是已经耗尽了 `buddget` 或者指定的时长。这个指标可以用来发现网络处理的瓶颈。

关闭软中断可以理解，因为 CPU 需要释放给用户级程序使用，而不能频繁用在中断处理上。

程序最终跳转到 `out` 标签处。其实不止被限制的情况下会执行到这里，还有就是 `budget` 多于等待处理的数据帧，当前 CPU 已经没有等待处理的 NAPI 结构体了，也会执行到 `out` 标签。

`out` 标签中需要提到的一个重要步骤是 `net_rps_action_and_irq_enable`，这个函数会唤醒其他 CPU 来处理他们本地的数据帧，这个和 RPS 功能有关，后续会详细说明。

#### NAPI `poll` 函数详解

前面提到过，驱动程序负责分配内存空间给 DMA 写入数据帧，所以驱动程序也需要负责 `unmap` 这些内核空间，处理数据帧并向协议栈上层传递。

作为例子，我们看下 `igb` 的实现：

```c
/**
 *  igb_poll - NAPI Rx polling callback
 *  @napi: napi polling structure
 *  @budget: count of how many packets we should handle
 **/
static int igb_poll(struct napi_struct *napi, int budget)
{
        struct igb_q_vector *q_vector = container_of(napi,
                                                     struct igb_q_vector,
                                                     napi);
        bool clean_complete = true;

#ifdef CONFIG_IGB_DCA
        if (q_vector->adapter->flags & IGB_FLAG_DCA_ENABLED)
                igb_update_dca(q_vector);
#endif

        /* ... */

        if (q_vector->rx.ring)
                clean_complete &= igb_clean_rx_irq(q_vector, budget);

        /* If all work not completed, return budget and keep polling */
        if (!clean_complete)
                return budget;

        /* If not enough Rx work done, exit the polling mode */
        napi_complete(napi);
        igb_ring_irq_enable(q_vector);

        return 0;
}
```

这里需要关注的几个操作：

* 如果内核支持 DCA 的话，这里会预热 CPU 缓存，这样后续访问就可以直接命中
* `igb_clean_rx_irq` 处理大部分核心工作，后续详细说明
* 检查是否有更多的数据帧需要处理，如果有，返回 `budget` 并持续轮询；
* 如果没有，则通过 `napi_complete` 关闭 NAPI 子系统，并解除中断屏蔽

`igb_clean_rx_irq`

这个函数实现为一个循环，一次处理一个数据帧，直到耗尽 `budget` 或者没有更多数据帧等待处理。

1. 分配新的 buffer 加入到 ring buffer（接收队列 - RX queue）
2. 从接收队列中取走一个 buffer，加入到 `skb` 结构体
3. 检查该 buffer 是不是 `End of Packet`，如果不是则继续取下一个 buffer 追加到 `skb`
4. 校验整个数据帧的布局和头部是否完整
5. 更新统计数据：已处理的字节数（`skb->len`）
6. 设置 `skb` 结构体的 哈希值，校验和，时间戳，协议簇 等，这些值都是硬件提供的；如果硬件报告了校验和失败，这里会更新对应的统计数据 `csum_error`，继续交给上层协议处理；协议字段通过一个单独的函数 `eth_type_trans` 计算处理，并保存在 `skb` 结构体里
7. 构造完毕的 `skb` 结构体通过 `napi_gro_receive` 向协议栈上层传递
8. 更新统计数据：已处理的包数量
9. 重复上述步骤，处理下一个数据帧

#### 监控数据处理

上述过程提供的统计数据一般都输出到了系统文件：`/proc/net/softnet_stat`，但是关于这个文件的文档说明几乎没有；文件里的每个字段也没有对应的标签分类，不能的内核版本都可能不一样。目前只能通过内核源码来明确具体含义：

[net/core/net-procfs.c](https://github.com/torvalds/linux/blob/v3.13/net/core/net-procfs.c#L161-L165)

```c
  seq_printf(seq,
       "%08x %08x %08x %08x %08x %08x %08x %08x %08x %08x %08x\n",
       sd->processed, sd->dropped, sd->time_squeeze, 0,
       0, 0, 0, 0, /* was fastroute */
       sd->cpu_collision, sd->received_rps, flow_limit_count);
```

之前在 `net_rx_action` 里提到了 `squeeze_time` 这个统计数据，刚好这里一起看下：

```sh
$ cat /proc/net/softnet_stat
6dcad223 00000000 00000001 00000000 00000000 00000000 00000000 00000000 00000000 00000000
6f0e1565 00000000 00000002 00000000 00000000 00000000 00000000 00000000 00000000 00000000
660774ec 00000000 00000003 00000000 00000000 00000000 00000000 00000000 00000000 00000000
61c99331 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000 00000000
6794b1b3 00000000 00000005 00000000 00000000 00000000 00000000 00000000 00000000 00000000
6488cb92 00000000 00000001 00000000 00000000 00000000 00000000 00000000 00000000 00000000
```

关于这个文件的的几点解释：

* 每一行对应一个 `struct softnet_data` 结构体，也就对应一个 CPU
* 数值之间空格分割，输出格式是 十六进制
* 第一个值 `sd->processed` 表示处理的包数量（多网卡 bond 模式可能多于实际的收包数量）
* 第二个值 `sd->dropped` 表示丢包数量，因为处理队列满了
* 第三个值 `sd->time_squeeze` 表示软中断处理 `net_rx_action` 被迫打断的次数
* 接下来的五个值都是零
* 第九个值 `sd->cpu_collision` 表示发送数据时获取设备锁冲突，比如多个 CPU 同时发送数据
* 第十个值 `sd->received_rps` 表示当前 CPU 被唤醒的次数（通过处理器间中断）
* 最后一个值 `sd->flow_limit_count` 表示 `flow limit` 达到上限的次数

#### 调优数据处理

这里可以调整的一个参数是 `budget`，影响 `net_rx_action` 的处理效率。

```sh
$ sudo sysctl -w net.core.netdev_budget=600
```

这个值针对每个 CPU 单独生效，默认值是 300

### GRO

Generic Receive Offloading (GRO) 是一种替代硬件 Large Receive Offloading (LRO) 的软件实现。背后的主要思想是，尽量减少向上传递的包数量，进而减少上层协议栈处理的 CPU 负担。举个例子，在传输一个大文件的时候，大部分的数据帧都是文件数据块；相较于一次传递一个小数据帧，如果可以把这些数据帧组合成一个巨大的数据帧传递给上层，那么协议层就只需要处理一个协议头，间接提高用户程序的数据接收速度。

这种合并包的优化存在一个问题就是，潜在的信息丢失。比如有些数据帧设置了额外的选项，或者其他什么标志位，在被合并到其他数据帧里的时候就可能丢失这些信息。这也是 LRO 没有大范围应用的原因，因为硬件层的实现没有严格限制包合并的条件和规则。GRO 作为纯软件实现，则严格规定了什么情况下才能对数据包做合并。

PS：我们偶尔在 `tcpdump` 抓包的时候，可能会看到一些巨大的包尺寸，这就是系统开启了 GRO 的结果；因为从整个网络栈来看，抓包的过程发生在 GRO 完成之后。

这个特性可以通过 `ethtool` 工具进行调整：

```sh
$ sudo ethtool -K eth0 gro on
```

也可以查看当前系统是否开启了这个功能：

```sh
$ ethtool -k eth0 | grep generic-receive-offload
generic-receive-offload: on
```

需要注意的是，这个操作会导致网卡重启，长连接会被中断。

### `napi_gro_receive`

实现上，对于 GRO 开启的情况，会逐个遍历上层协议栈注册到 GRO 的 `filter`。通过这样的方式，协议层就可以告知设备层，这个数据包是否可以被合并（比如判断是否属于同一个网络流），以及合并以后需要做什么样的额外操作（比如 tcp 需要对一个被合并的包发送给对端 ack 响应）。细节略。

## 内部机制：负载均衡和流控

* RSS / RPS
* RFS / aRFS

### Receive Packet Steering (RPS)

一个 CPU 会同时处理硬件中断和对数据包的轮询处理。大部分现代网卡都支持多队列，也就是说，收到的数据帧可以被 DMA 拷贝到不同的内存区域，每个队列对应一份独立的空间，也就对应一个独立的 NAPI 结构体来管理这个区域。因此，可以利用到多个 CPU 来处理网络数据帧。

这个功能一般叫做 Receive Side Scaling (RSS)。

Receive Packet Steering (RPS)则是对 RSS 的软件实现，因此可以针对任意类型网卡开启，即使只有一个队列。因为是纯软件实现，RPS 起作用之前，数据帧已经被从 DMA 区域取走了，所以只能影响这个时间点之后的协议栈负载。

RPS 基本的流程是这样的：

1. 基于数据帧产生一个哈希值，进而计算出目标 CPU；
2. 然后数据帧被加入到目标 CPU 的网络接收队列（backlog）
3. 发送 Inter-processor Interrupt (IPI) 到目标 CPU
4. 唤醒目标 CPU 来处理它的 backlog 队列

之前提到过，`/proc/net/softnet_stat` 文件中包含了一个相关的统计项 `sd->received_rps`，记录了 CPU 收到的 IPI 中断的数量。

所以数据帧在递交上层协议栈之前，可能会首先经过 RPS 被负载均衡到其他 CPU，从而分散处理压力。

### 调优：启用 RPS

通过位掩码，我们可以指定 CPU 来处理指定的网卡队列：

```sh
/sys/class/net/eth0/queues/rx-0/rps_cpus

```

上面的文件指向网卡 `eth0` 的 `rx-0` 队列，格式为十六进制的数字，可以指定一个或多个 CPU。对 RSS 网卡来说，这个功能没有开启的必要；只有在队列数据少于 CPU 数目时，可以考虑开启 RPS，并且尽量保证，被均衡的 CPU 和当前处理中断的 CPU 共用同一个 CPU cache。

### Receive Flow Steering (RFS)

RFS 一般结合 RPS 一起使用，因为 RPS 没有考虑到数据本地性的问题，缓存命中率偏低。利用 RFS，我们可以把属于同一个流的数据帧，尽量引导到同一个 CPU 进行处理。

### 调优：开启 RFS

RFS 依赖于 RPS，必须首先保证 RPS 是开启的。

RFS 内部维护一个流的全局哈希表，这个哈希表的大小可以调整：

```sh
$ sudo sysctl -w net.core.rps_sock_flow_entries=32768
```

另外，针对每个网卡队列，也可以单独设置 rps 的流数量：

```sh
$ sudo bash -c 'echo 2048 > /sys/class/net/eth0/queues/rx-0/rps_flow_cnt'
```

## 数据包递交上层协议栈：`netif_receive_skb`

1. timestamp - 涉及负载均衡
2. RPS - 默认不开启
	1. 插入 backlog / 执行 flow limit
	2. backlog NAPI poller - `process_backlog`
3. packet cap - tcpdump 等工具生效的阶段
4. protocol layer

### 调优：时间戳

```sh
$ sudo sysctl -w net.core.netdev_tstamp_prequeue=0
```

这里可以调整数据帧生成时间戳的时机，默认是 1，也就是在 RPS 之前；设置为 0 的话，可以把生成时间戳的负载均衡到其他 CPU 上（会引入部分延时，毕竟数据在 CPU 之间移动了）。

### backlog 队列

RPS 首先计算目标 CPU，然后将数据加入目标 CPU 的 backlog 队列：

```c
cpu = get_rps_cpu(skb->dev, skb, &rflow);

if (cpu >= 0) {
  ret = enqueue_to_backlog(skb, cpu, &rflow->last_qtail);
  rcu_read_unlock();
  return ret;
}
```

在 `enqueue_to_backlog` 中，会检查该 CPU 的队列大小：

```c
qlen = skb_queue_len(&sd->input_pkt_queue);
if (qlen <= netdev_max_backlog && !skb_flow_limit(skb, qlen)) {
```

`input_pkt_queue` 代表的就是 cpu 的 backlog 队列，用来和 `netdev_max_backlog` 比较，如果队列过长，数据帧就会被丢弃；另外，如果 `flow limit` 生效，也会导致丢包。这里的丢包数据可以在 `/proc/net/softnet_stat` 文件中看到。

PS：如果没有开启 RPS，这里的限制条件就没有用到了。

### Flow limits

可能出现的情况是，某个流的数据巨多，挤满了 backlog 队列，进而导致其他数据流的处理被延迟；所以这里还需要进行流控，避免某个流独占 CPU。这里实现的基本原理是，在 backlog 队列达到最大值的一半时，开始检查每个流占用队列的比例，如果当前队列中某个流的数据帧超过了一半，就对当前流执行丢包。

### 监控 backlog 丢包

### 调优

调整 backlog 队列的最大值，可以减少丢包的发生：

```sh
$ sudo sysctl -w net.core.netdev_max_backlog=3000
```

调整 backlog 队列处理的权重：

```sh
$ sudo sysctl -w net.core.dev_weight=600
```

这个值和驱动注册的 weight 类似，但是可以调整（驱动一般硬编码为 64）。

流控所使用的哈希表大小也可以调整：

```sh
$ sudo sysctl -w net.core.flow_limit_table_len=8192
```

这个哈希表越大，流控的粒度越细；相反，则容易误伤。

### 抓包

在把数据帧递交上层协议栈的最后关头，会经过 `packet tap`，执行一系列过滤操作。

### 送达协议层

解析数据帧中的协议类型字段，每种协议类型都注册了各自的接收函数，从而可以把数据帧发送给特定的协议层。

## 协议层

* IP 层
* 应用层协议注册
* UDP 层

## MISC

### `SO_INCOMING_CPU`

* 作用和使用场景
* 使用内核版本

参考：<http://man7.org/linux/man-pages/man7/socket.7.html>

gettable since Linux 3.19, settable since Linux 4.4

Sets or gets the CPU affinity of a socket.  Expects an integer flag.

```c
int cpu = 1;
setsockopt(fd, SOL_SOCKET, SO_INCOMING_CPU, &cpu, sizeof(cpu));
```

Because all of the packets for a single stream (i.e., all packets for the same 4-tuple) arrive on the single RX queue that is associated with a particular CPU, the typical use case is to employ one listening process per RX queue, with the incoming flow being handled by a listener on the same CPU that is handling the RX queue.  This provides optimal NUMA behavior and keeps CPU caches hot.

目前最新版本内核已经弃用，参考这篇 [cloudflare blogpost](https://blog.cloudflare.com/perfect-locality-and-three-epic-systemtap-scripts/)
