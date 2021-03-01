---
title: python-os-popen
date: 2021-03-01
tags: python
---

# os.popen 的若干问题

## 背景

`os.popen` 是 Python 标准库 `os` 的一个函数，一般用来在程序中 **调用 `shell` 命令，并需要读取输出** 时使用。

```python
os.popen(command[, mode[, bufsize]])
```

> **Open a pipe** to or from command. The return value is an open **file object** connected to the pipe, which can be read or written depending on whether mode is 'r' (default) or 'w'. The bufsize argument has the same meaning as the corresponding argument to the built-in open() function. The exit status of the command (encoded in the format specified for wait()) is available as the return value of the close() method of the file object, except that when the exit status is zero (termination without errors), None is returned.

理论上，除了读取输出，我们也可以写入，因为 `os.popen` 本质上是提供了一个管道，允许双向通信。

## 问题

这个函数在 Python 2.6 之后已经被标记为 [deprecated](https://docs.python.org/2.7/library/os.html#os.popen)，但是因为 **使用上的便利**，还是有很多 Python 程序在需要调用 shell 命令时使用该函数。比如：

```python
now = os.popen('date').read()
# do something with @now ...
```

我们一般都只会关心 shell 命令的输出，也就是上述代码中的 `now`；

容易掉坑的地方就在这里，我们忽略了对 `os.popen()` 返回值的处理！

为什么需要处理返回值？有两点原因：

* 检查 shell 命令是否执行成功
* 回收子进程的资源

对于第一点，如果我们只是调用一些简单的 shell 命令，比如 `ls/date/git` 之类的话，倒是一般不需要担心执行可能会失败；但是对于第二点，则必须加以注意，否则会产生 [zombie process](https://en.wikipedia.org/wiki/Zombie_process) （僵尸进程的诸多弊端，这里不再赘述）

那么如何做到回收子进程的资源呢？调用 `close()` 即可。因为 `os.popen` 返回的是一个 `file object`，抽象了一层 **file-like** 的接口，这也是我们可以对其调用 `read()`，甚至 `write()` 的原因；同理，类似 file 的操作，我们可以在 `os.popen()` 之后调用 `close()`：

```python
sh_pipe = os.popen('date')
now = sh_pipe.read()
# do something with @now ...
sh_pipe.close()
```

这里调用的 `close()` ，内部封装了 `wait()` 调用，完成对子进程的[资源回收](https://man7.org/linux/man-pages/man2/waitid.2.html)：

```python
def popen(cmd, mode="r", buffering=-1):
    if not isinstance(cmd, str):
        raise TypeError("invalid cmd type (%s, expected string)" % type(cmd))
    if mode not in ("r", "w"):
        raise ValueError("invalid mode %r" % mode)
    if buffering == 0 or buffering is None:
        raise ValueError("popen() does not support unbuffered streams")
    import subprocess, io
    if mode == "r":
        proc = subprocess.Popen(cmd,
                                shell=True,
                                stdout=subprocess.PIPE,
                                bufsize=buffering)
        return _wrap_close(io.TextIOWrapper(proc.stdout), proc)
    else:
        proc = subprocess.Popen(cmd,
                                shell=True,
                                stdin=subprocess.PIPE,
                                bufsize=buffering)
        return _wrap_close(io.TextIOWrapper(proc.stdin), proc)

# Helper for popen() -- a proxy for a file whose close waits for the process
class _wrap_close:
    def __init__(self, stream, proc):
        self._stream = stream
        self._proc = proc
    def close(self):
        self._stream.close()
        returncode = self._proc.wait()
        if returncode == 0:
            return None
        if name == 'nt':
            return returncode
        else:
            return returncode << 8  # Shift left to match old behavior
    def __enter__(self):
        return self
    def __exit__(self, *args):
        self.close()
    def __getattr__(self, name):
        return getattr(self._stream, name)
    def __iter__(self):
        return iter(self._stream)
```

详见 Python 标准库代码 [os.py](https://github.com/python/cpython/blob/master/Lib/os.py#L976)

## 建议

所以，更健壮的 `os.popen()` 的使用方式应该是：

```python
sh_pipe = os.popen('date')
exit_status = sh_pipe.close()
if exit_status == 0:
    now = sh_pipe.read()
    # do something with @now
else:
    # do some exception work here..
```

或者，我们可以借助 `with` 语法糖，写出更简洁的代码：

```python
with os.popen('date') as sh_pipe:
    now = sh_pipe.read()
    # do something with @now
```

因为 `os.popen()` 返回的 file-like object 封装了 `__enter__` 和 `__exit__` 方法，我们借助 context 机制实现对 `close()` 的自动调用。

## 更进一步

上面提到，`os.popen()` 已经不再建议使用，那么我们在新的代码中，应该使用什么接口呢？

[subprocess](https://docs.python.org/2.7/library/subprocess.html#module-subprocess) 是 Python 官方建议的 `os.popen()` 的替代方式，甚至提供了[迁移示例](https://docs.python.org/2.7/library/subprocess.html#replacing-os-popen-os-popen2-os-popen3)：

```python
pipe = os.popen("cmd", 'r', bufsize)
==>
pipe = Popen("cmd", shell=True, bufsize=bufsize, stdout=PIPE).stdout
```

如果仅仅是为了实现本文的场景

> 执行 shell 命令，并读取其输出

subprocess 模块提供了更直接的方式：

[subprocess.check_output(args, *, stdin=None, stderr=None, shell=False, universal_newlines=False)](https://docs.python.org/2.7/library/subprocess.html#subprocess.check_output)

同样是上文的示例，我们这样重写：

```python
now = subprocess.check_out(['date'])
# do something with @now
```

bingo!

对于之前提供的两个问题：

* 命令执行出错如何处理？函数会抛出 `CalledProcessError` 异常，捕获处理即可。
* zombie process 如何避免？subprocess.check_out() 内部做了完善的处理，无需再额外关注

这才是现代 Python 程序该有的样子！简洁 & 可靠 :)

## 彩蛋

在目前的 Python 版本中（2.7+），`os.popen()` 虽然是通过在内部调用 `subprocess.Popen()` 实现的，但是 subprocess 的这个接口过于原始，os.popen() 并没有进一步封装其 `subprocess.wait()` 接口，所以不加以小心的话，还是会出现上文提到的僵尸进程的问题。
