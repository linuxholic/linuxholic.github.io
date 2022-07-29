---
title: aio-overflow
date: 2022-07-29 09:40:58
tags:
    - kernel
    - aio
---

# 起因

在使用 linux kernel aio 接口的过程中，发现一个诡异的问题：如果单次 aio 请求的数据大小超过了某一个阈值，kernel aio 就总是处理失败：不管请求数据多大，aio 返回的结果一直都是 2147479552 ！

# AIO

一般情况下我们使用的 IO 接口都是同步类型的，这样的接口会同步阻塞调用程序，直到底层数据准备好，才会返回。比如调用 read 函数读取磁盘文件的数据，因为涉及到数据从物理磁盘传输到内存，这个过程一般耗时在毫秒级别，期间调用程序无事可做，就会被 OS 阻塞挂起，等待直到数据读取完成之后，read 函数返回，控制权才会重新回到调用程序。

AIO 则是采用异步的方式，发起 IO 请求之后不再阻塞，而是立即返回，这样调用程序就可以在 IO 执行期间做更多的其他事情，从而实现并行，提升处理效率。

具体来说，在 linux 上实现 AIO 主要有两种方式：Posix 和 kernel

<!-- more -->

## POSIX AIO

posix 方式通过 libc 实现，内部通过 thread pool 将同步的 IO 请求转为后台线程的操作，并在合适的时候通知业务线程，从而实现异步的效果。可以看出，这种方式完全在用户态实现，不需要内核的配合，存在很多局限性，比如多线程操作的并发问题，数据共享问题，等等。

## kernel AIO

相反，直接在内核态实现 AIO 可以实现更高的效率，并简化用户态的实现。相应的，AIO 额外增加了若干系统调用接口，核心是这两个：

* `io_submit`
* `io_getevents`

一个用来提交 aio 请求，一个用来获取 aio 结果。

单个 aio 请求用如下结构体表示(`man io_submit`)：

```c
#include <linux/aio_abi.h>

struct iocb {
   __u64   aio_data;
   __u32   PADDED(aio_key, aio_rw_flags);
   __u16   aio_lio_opcode;
   __s16   aio_reqprio;
   __u32   aio_fildes;
   __u64   aio_buf;
   __u64   aio_nbytes;
   __s64   aio_offset;
   __u64   aio_reserved2;
   __u32   aio_flags;
   __u32   aio_resfd;
};

```

[struct iocb](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/aio_abi.h#L73) 结构体中的 `aio_nbytes` 成员用来表示请求大小

表示单个 aio 请求结果的结构体如下：

```c
/* read() from /dev/aio returns these structures. */
struct io_event {
    __u64       data;       /* the data field from the iocb */
    __u64       obj;        /* what iocb this event came from */
    __s64       res;        /* result code for this event */
    __s64       res2;       /* secondary result */
};
```

[struct io_event](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/aio_abi.h#L60) 结构体中的 `res` 一般用来表示 aio 请求的结果

* `res > 0` 表示 aio 请求成功的字节数
* `res < 0` 表示 aio 请求失败的错误码

所以，我们就可以通过提交的 `aio_nbytes` 和获取结果的 `res` 对比，来判断 aio 请求是否完整。

# 现象

当通过 AIO 请求的数据大小超过 2G 左右时，请求大小和结果大小就开始发生不匹配的情况，并且结果大小永远都是 2147479552。

## 可能的原因

第一个可疑点是 [整数溢出](https://en.wikipedia.org/wiki/Integer_overflow)，因为 2G 左右刚好和 int max 接近：

|缩写|数值|
|---|---|
|2G|2147483648|
|int max|2147483647|
|aio magic| 2147479552|

但是一通检查下来，程序内部都是使用的 `int64_t` 表示，足以容纳远大于 2G 的大小，不可能发生溢出的情况。

整数溢出还有第二种途径：乘法或者加法。比如两个 int32 相乘，预期得到一个 int64 的结果：

```c
a = b * c
```

其中 b 和 c 都是 32 位整型，a 是 64 位整型，直觉来看，用来保存结果的 a 足够大，应该没问题；事实上是，`b*c` 的中间结果使用了 int32 存储，就导致了整数溢出，最终 a 也就不会是正确的乘积。应用程序之前倒是有过这样的 bug，不过早已经修复了，最新的代码检查之后，也并没有发现类似可能的 bug。

排除了第一个可疑点之后，开始怀疑这是系统级别本身的限制，而不是应用程序的问题了；也考虑到 2147479552 虽然和 2147483647 接近，但是毕竟不相等：

|number|binary|
|---|---|
| 2147483647|0b1111111111111111111111111111111|
| 2147479552|0b1111111111111111111000000000000|

那就需要重点排查系统层面限制的可能了。

# `MAX_RW_COUNT`

作为面向 google 编程的程序员，直接搜索 2147479552，果然找到了[线索](https://stackoverflow.com/questions/70368651/why-cant-linux-write-more-than-2147479552-bytes)。内核限制了单次 read 或者 write 的最大数据范围：

```c
#define MAX_RW_COUNT (INT_MAX & PAGE_MASK)
```

这个[限制值](https://elixir.bootlin.com/linux/latest/source/include/linux/fs.h#L2493)刚好就是 2147479552，这个限制也可以在 `man read` 或者 `man write` 手册页中找到。

>  On Linux, read() (and similar system calls) will transfer at most 0x7ffff000 (2,147,479,552) bytes, returning the number of bytes actually transferred.  (This is true on both 32-bit and 64-bit systems.)

也就是说，内核对于超过这个限制的 IO 请求大小，会**静默截断**到这个限制值，而不会有任何错误标识！内核的逻辑是，单次 IO 本来就存在 partial write 或者 partial read 的情况，应用层应该通过检查实际的返回值来判断，读写是否完整；如果只有部分成功，应用层需要负责进行重试。

到这里为止，那应该就是应用程序的 bug 了，没有妥善处理 partial read 或者 write 的情况。但还有个问题是，应用程序用到的是 kernel AIO 接口，而不是上述普通的读写接口；也就是说，read 和 write 系统调用确实存在上述限制，但是我们没有用到这两个接口呀 (╯‵□′)╯︵┻━┻

不过有了这个抓手，还是可以顺藤摸瓜的，我们有理由怀疑 AIO 的内核实现中也极有可能存在类似的限制，[果不其然](https://elixir.bootlin.com/linux/latest/source/fs/aio.c#L1541)：

```c
/*
 * fs/aio.c 中的 aio_read() 函数，顾名思义，负责 AIO read 操作，涉及的调用链如下：
 *
 *     aio_read() -> aio_setup_rw() -> import_single_range()
 */
int import_single_range(int rw, void __user *buf, size_t len,
         struct iovec *iov, struct iov_iter *i)
{
    if (len > MAX_RW_COUNT)
        len = MAX_RW_COUNT;
    if (unlikely(!access_ok(buf, len)))
        return -EFAULT;

    iov->iov_base = buf;
    iov->iov_len = len;
    iov_iter_init(i, rw, iov, 1, len);
    return 0;
}
```

AIO 读写路径中，也限制了最大数据范围，所以这就终于解释了我们开头遇到的那个问题。

# 解决方案

搞清楚了问题根因，接下来就需要看下如何解决了。既然 `MAX_RW_COUNT` 是针对单次读写的限制，那么想当然的，我们可以考虑拆分过大的单次 IO 为多次的小份 IO，从而绕过内核的这个限制。

## vector IO

第一种方式，直接利用系统提供的 vector IO 能力，将多个子 IO 请求封装在多个结构体中，然后通过 vector 的方式一次性提交即可。

> [Vectored I/O
](https://en.wikipedia.org/wiki/Vectored_I/O) 也叫做 scatter/gather I/O，可以方便的一次性提交涉及多个位置的 IO 请求。

吊诡的是，AIO 手册页中并没有说明如何使用 vectored IO，但是我们可以在 [libaio](https://pagure.io/libaio/blob/master/f/src/libaio.h#_200) 的封装库中找到其用法

```c
static inline void io_prep_preadv(struct iocb *iocb, int fd, const struct iovec *iov, int iovcnt, long long offset)
{
    memset(iocb, 0, sizeof(*iocb));
    iocb->aio_fildes = fd;
    iocb->aio_lio_opcode = IO_CMD_PREADV;
    iocb->aio_reqprio = 0;
    iocb->u.c.buf = (void *)iov;
    iocb->u.c.nbytes = iovcnt;
    iocb->u.c.offset = offset;
}
```

换句话说，赋予了之前两个字段不同的含义：

* `buf`：之前表示用户内存空间地址，现在表示 vector array 的地址
* `nbytes`：之前表示 IO 请求大小，现在表示 vector array 的长度（包含多少个子请求）

撸起袖子就干，改造代码使用 AIO vector 方式重新提交请求，结果，还是翻车了 (╯‵□′)╯︵┻━┻

继续查看内核代码中 vector IO 相关的逻辑实现，才发现竟然也有类似的限制

```c
/*
 *  调用路径：aio_read() -> aio_setup_rw() -> __import_iovec()
 */
ssize_t __import_iovec(int type, const struct iovec __user *uvec,
         unsigned nr_segs, unsigned fast_segs, struct iovec **iovp,
         struct iov_iter *i, bool compat)
{
    /* 省略一些无关代码 */
    for (seg = 0; seg < nr_segs; seg++) {
        ssize_t len = (ssize_t)iov[seg].iov_len;

        if (!access_ok(iov[seg].iov_base, len)) {
            if (iov != *iovp)
                kfree(iov);
            *iovp = NULL;
            return -EFAULT;
        }

        if (len > MAX_RW_COUNT - total_len) {
            len = MAX_RW_COUNT - total_len;
            iov[seg].iov_len = len;
        }
        total_len += len;
    }
    /* 省略一些无关代码 */
}
```

在 `aio_setup_rw()` [函数](https://elixir.bootlin.com/linux/latest/source/fs/aio.c#L1505) 中，会根据用户提交的 AIO 请求类型，是否为 vectored 方式，调用两个不同的函数：

* `import_single_range()`
* `__import_iovec()`

虽然是不同的代码路径，但是都加上了 `MAX_RW_COUNT` 的限制，并且 vectored 方式下是针对**所有子请求加和**进行限制的，而不是针对单个子请求的限制。

## 链式回调：chained callback

既然并行的拆分无法奏效，那就只能采取串行的方式了。也就是，在每次完成 `MAX_RW_COUNT` 数量的 IO 请求之后，针对剩余数据再次发起 IO 请求，直到所有期望数据全部完成（或者碰到失败）为止。这种方式相对同步阻塞的 read 或者 write 而言，相对复杂一些，因为需要在前后 callback 之间记住一些信息：

* 预期数据量大小
* 已完成数据量大小
* 剩余数据量大小

等等；而同步读写接口，只需要在一个 loop 中循环重试即可。

# wrap up

针对开头提到的问题，其实如果对 syscall 手册页足够熟悉的话，或者很熟悉 kernel IO subsystem，应该可以很快地意识到 2147479552 这个 magic number 的特殊含义所在，也就不用在其他方向上花费太多时间排查了。

2147479552, good luck!
