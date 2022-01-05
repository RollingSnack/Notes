---
Title: Python 幕后 13 - GIL 及其对 Python 多线程的影响
Tags: [Type/Translation, 编程语言/Python, 操作系统/线程]
Original Link: https://tenthousandmeters.com/blog/python-behind-the-scenes-13-the-gil-and-its-effects-on-python-multithreading/
Original Title: Python behind the scenes #13: the GIL and its effects on Python multithreading
Original Author: Victor Skvortsov
---

# Python 幕后 13 - GIL 及其对 Python 多线程的影响

---

你或许听过，GIL 指的是全局解释器锁（*Global Interpreter Lock*），它的工作是保障 CPython 解释器线程安全。
GIL 在任何时候都只允许单个操作系统线程执行 Python 字节码，这么做的后果是，不可能把工作分配到多个线程上来加速 CPU 密集型的 Python 代码。
然而，这还不是 GIL 的唯一负面效应。
GIL 带来的开销使多线程程序更慢，且更令人惊讶的是，它甚至会对 I/O 密集型线程造成影响。

在本文中，我会和你聊聊更多有关 GIL 的不甚明显的影响。
在此过程中，我们会讨论 GIL 到底是什么，它为什么存在，它是如何工作的，以及它在未来将如何影响 Python 的并发。

**注意：** 本文所指为 CPython 3.9。
随着 CPython 的发展，一些实现细节必然会变化。
我会尝试追踪重要的变化并添加更新说明。

## 操作系统线程、Python 线程和 GIL

首先让我带你回顾什么是 Python 线程，以及多线程在 Python 中是如何工作的。
当你执行 `python` 程序时，操作系统会创建新进程，其中有一个被称作主线程的执行线程。
与其他所有 C 程序一样，主线程通过进入 `python` 的 `main()` 函数来开始执行。
主线程接下来所做的一切可以总结为三个步骤：

1. [初始化解释器](https://tenthousandmeters.com/blog/python-behind-the-scenes-3-stepping-through-the-cpython-source-code/)
2. [编译 Python 代码为字节码](https://tenthousandmeters.com/blog/python-behind-the-scenes-2-how-the-cpython-compiler-works/)
3. [进入求值循环（*evaluation loop*）执行字节码](https://tenthousandmeters.com/blog/python-behind-the-scenes-4-how-python-bytecode-is-executed/)

主线程是一个常见的操作系统线程，执行已编译过的 C 代码。
其线程状态包括 CPU 寄存器的值和 C 函数的调用栈。
不过，一个 Python 线程必须采集 Python 函数的调用栈、异常状态和其他与 Python 相关的东西。
所以，CPython 所做的是把这些东西放进一个 [线程状态结构体](https://github.com/python/cpython/blob/5d28bb699a305135a220a97ac52e90d9344a3004/Include/cpython/pystate.h#L51) 中，并把线程状态和操作系统线程关联起来。
换言之，Python 线程 = 操作系统线程 + Python 线程状态。

求值循环是一个无限循环，它包含一个巨大的 switch 语句，涵盖了所有可能的字节码指令。
若要进入这个循环，线程须持有 GIL。
主线程在初始化时获得了 GIL，所以它可以自由进入。
当它进入循环，它就根据 switch 语句一个个地执行字节码指令。

线程时不时要暂停执行字节码。
它会在求值循环的每个迭代开始前，检查是否有任何理由去暂停执行。
一个原因是我们关注：另一个线程请求了 GIL。
下面是这个逻辑在代码中的实现方式：

```C
PyObject*
_PyEval_EvalFrameDefault(PyThreadState *tstate, PyFrameObject *f, int throwflag)
{
    // ... 局部变量的定义和其他乏味事物

    // 求值循环
    for (;;) {

        // `eval_breaker` 说明是否应该暂停执行字节码
        // 例如，其他线程请求 GIL
        if (_Py_atomic_load_relaxed(eval_breaker)) {

            // `eval_frame_handle_pending()` 暂定执行字节码
            // 例如，当另一个线程请求 GIL
            // 该函数会丢弃 GIL 然后重新等待 GIL
            if (eval_frame_handle_pending(tstate) != 0) {
                goto error;
            }
        }

        // 获取下一个字节码指令
        NEXTOPARG();

        switch (opcode) {
            case TARGET(NOP) {
                FAST_DISPATCH(); // 下一轮迭代
            }

            case TARGET(LOAD_FAST) {
                // ... 加载局部变量的代码
                FAST_DISPATCH(); // 下一轮迭代
            }

            // ... 其余 117 种情况，适配每个可能的操作码（opcode）
        }

        // ... 异常处理
    }

    // ... 终止
}
```

在一个单线程 Python 程序里，主线程就是这唯一的线程，并且绝不释放 GIL。
现在让我们看看在多线程程序里会发生些什么。
我们使用 [`threading`](https://docs.python.org/3/library/threading.html) 标准模块来开启新的 Python 线程。

```Python
import threading

def f(a, b, c):
    # 做点什么
    pass

t = threading.Thread(target=f, args=(1, 2), kwargs={'c': 3})
t.start()
```

`Thread` 实例的 `start()` 方法会创建新的操作系统线程。
在包括 Linux 和 macOS 在内的类 Unix 系统上，它调用 [`pthread_create()`](https://man7.org/linux/man-pages/man3/pthread_create.3.html) 函数来实现。
新创建的线程从执行带有 `boot` 参数的 `t_bootstrap()` 函数开始。
参数 [`boot`](https://github.com/python/cpython/blob/5d28bb699a305135a220a97ac52e90d9344a3004/Modules/_threadmodule.c#L1019) 是个结构体，其中包含目标函数、传递的参数以及新的操作系统线程的线程状态。
函数 [`t_bootstrap()`](https://github.com/python/cpython/blob/5d28bb699a305135a220a97ac52e90d9344a3004/Modules/_threadmodule.c#L1029) 做了许多事情，不过最重要的，它获取 GIL，然后进入求值循环执行目标函数的字节码。

为获取 GIL，某个线程首先要去检查是否有其他线程持有 GIL。
如果不是该情况，该线程将立刻获得 GIL。
否则，它要等待 GIL 被释放。
它等待的时间固定，称作 **切换间隔（*switch interval*）**（默认 5 毫秒），倘若这段时间内 GIL 没被释放，它就会设置 `eval_breaker` 和 `gil_drop_request` 标志。
标志 `eval_breaker` 告诉持有 GIL 的线程去暂停执行字节码，`gil_drop_request` 解释原因。
这还会通知等待 GIL 中的线程，其中之一将得到 GIL。
唤醒哪个线程是由操作系统决定的，所以它可以是也可以不是设置标志的那个线程。

这是我们需要了解 GIL 的最基本知识。
现在让我展示一下我前面谈到的它的影响。
如果你觉得有趣，继续下一章节吧，我们将在那里更加详细地研究 GIL。

## GIL 的影响

GIL 的第一个影响广为人知：多个 Python 线程不可并行。
那么，即便在多核机器上，多线程程序也不会比等价的单线程程序更快。
作为 Python 代码并行化的一次幼稚尝试，考虑如下的 CPU 密集型函数，它按给定次数执行递减操作：

```Python
def countdown(n):
    while n > 0:
        n -= 1
```

现在假设我们希望执行一亿次（100,000,000）递减。
我们可以在单线程中运行 `countdown(100_100_000)`，或两个线程中的 `countdown(50_000_000)`，或四个线程中的 `countdown(25_000_000)` 等等。
在诸如 C 这类没有 GIL 的语言中，我们是可以看见增加线程数时的提速效果的。
在我双核 [超线程（hyper-threading）](http://www.lighterra.com/papers/modernmicroprocessors/) 的 MacBook Pro 中运行 Python，我的观察如下：

| 线程数 | 每个线程递减次数 | 耗时（秒/三次中取最佳） |
| ------ | ---------------- | ----------------------- |
| 1      | 100,000,000      | 6.52                    |
| 2      | 50,000,000       | 6.57                    |
| 4      | 25,000,000       | 6.59                    |
| 8      | 12,500,000       | 6.58                    |

耗时没有变。
事实上，由于 [上下文切换（context switching）](https://en.wikipedia.org/wiki/Context_switch) 的开销，多线程程序可能运行得更慢。
默认切换间隔是 5 毫秒，所以上下文切换得没有这么频繁。
但如果我们缩短切换间隔，我们会观察到速度下降。
后文会解释，我们为什么要这么做。

尽管 Python 线程不能帮助我们加速 CPU 密集型的代码，但当我们希望同步执行多个 I/O 密集型任务时，它们还是有用的。
设想一个服务器，它监听传入的连接，并在收到连接时于单独的线程内运行处理函数。
处理函数借由从客户端的套接字中读取或往其中写入，来与客户端对话。
当从套接字中读取时，线程只是挂起，直到客户端发送些东西过来。
这就是多线程的用处：另一个线程可以在这期间运行。

在等待 I/O 操作的线程持有 GIL 时，为了让其他线程也可以运行，CPython 以如下模式实现所有 I/O 操作：

1. 释放 GIL。
2. 执行诸如 [write()](<[`write()`](https://man7.org/linux/man-pages/man2/write.2.html)>)、[recv()](<, [`recv()`](https://man7.org/linux/man-pages/man2/recv.2.html)>) 和 [accept()](<, [`accept()`](https://man7.org/linux/man-pages/man2/accept.2.html);>) 等操作。
3. 获取 GIL。

如此，某个线程可能会在另一个线程设置 `eval_breaker` 和 `gil_drop_request` 之前主动释放 GIL。
通常，线程只有在操作 Python 对象时才需要持有 GIL。
所以 CPython 不仅仅将 “释放-执行-获取” 模式应用在 I/O 操作上，还应用在如 [`select()`](https://man7.org/linux/man-pages/man2/select.2.html) 和 [`pthread_mutex_lock()`](https://linux.die.net/man/3/pthread_mutex_lock) 等操作系统的其他阻塞调用，以及纯 C 的大量计算上。
举例说，[`hashlib`](https://docs.python.org/3/library/hashlib.html) 标准模块中的散列函数就会释放 GIL。
这使得我们实际上可以为调用此类函数的多线程 Python 代码加速。

假设我们想要计算 8 条 128 MB 大小信息的 SHA-256 哈希值。
我们可以在单个线程中为每条消息计算 `hashlib.sha256(message)`，但我们也可以将这项作业分布到多个线程上。
如果在我的机器上做比较，我得到的结果如下：

| 线程数 | 每个线程上的消息大小 | 耗时（秒/三次中取最佳） |
| ------ | -------------------- | ----------------------- |
| 1      | 1 GB                 | 3.30                    |
| 2      | 512 MB               | 1.68                    |
| 4      | 256 MB               | 1.50                    |
| 8      | 128 MB               | 1.60                    |

从单线程到双线程，速度几乎翻倍，因为线程得以并行执行。
添加更多的线程没什么帮助了，因为我的机器只有两个物理内核。
这部分的结论是，如果代码调用会释放 GIL 的 C 函数，那么使用多线程就可以加速 CPU 密集型 的 Python 代码。
注意，你不仅可以在标准库中找到这类函数，重计算的第三方库中也存在，比如 [NumPy](https://github.com/numpy/numpy)。
你也可以自己写一个 [释放 GIL 的 C 拓展](https://docs.python.org/3/c-api/init.html?highlight=gil#releasing-the-gil-from-extension-code)。

我们提到了 CPU 密集型线程——大部分时间都在计算的线程，和 I/O 密集型线程——大部分时间在等待 I/O 的线程。
GIL 最有趣的影响发生在我们将二者混合时。
设想一个简单的 TCP 回显服务器（*TCP echo server*），它监听传入的连接，并在一个客户端连入时生成一个新线程处理之：

```Python
from threading import Thread
import socket


def run_server(host='127.0.0.1', port=33333):
    sock = socket.socket()
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind((host, port))
    sock.listen()
    while True:
        client_sock, addr = sock.accept()
        print('Connection from', addr)
        Thread(target=handle_client, args=(client_sock,)).start()


def handle_client(sock):
    while True:
        received_data = sock.recv(4096)
        if not received_data:
            break
        sock.sendall(received_data)

    print('Client disconnected:', sock.getpeername())
    sock.close()


if __name__ == '__main__':
    run_server()
```

这个服务器每秒可以处理多少请求？
我写了个 [简单的客户端程序](https://github.com/r4victor/pbts13_gil/blob/master/effect2_client.py)，它只与服务器间以尽可能快的速度收发 1 字节信息，然后达到了大概 30k RPS（*Requests per second*）。
度量或许不准，因为客户端和服务器运行在同一台机器上，不过这非重点。
重点在于观察服务器在执行一些 CPU 密集型任务时，RPS 下降得如何。

设想另一个完全一致的服务器，但多了个额外的虚拟线程，它在一个无限循环中递增递减某一变量（任何 CPU 密集型任务都做同样的事）:

```Python
# ... the same server code

def compute():
    n = 0
    while True:
        n += 1
        n -= 1

if __name__ == '__main__':
    Thread(target=compute).start()
    run_server()
```

你觉得 RPS 会如何减少？
稍微缩小？
2 倍缩小？
还是 10 倍缩小？
RPS 降到了 100，是 300 倍缩小！
如果你熟悉操作系统调度线程的方式，这就太令人惊讶了。
为了让你理解我的意思，我们跑个让 CPU 密集型线程作为独立进程的服务器，以令他们不会受到 GIL 的影响。
可以把代码分开到两个不同文件，或使用 [`multiprocessing`](https://docs.python.org/3/library/multiprocessing.html) 标准模块来开辟个新进程，像这样：

```Python
from multiprocessing import Process

# ... 同样的服务器代码

if __name__ == '__main__':
    Process(target=compute).start()
    run_server()
```

这大约能达到 20k RPS。
不仅如此，如果我们启动两个、三个或四个 CPU 密集型进程，RPS 也几乎保持不变。
操作系统调度器优先考虑了 I/O 密集型线程，这是正确做法。

服务器例子中的 I/O 密集型线程是在等待套接字准备读写，但任何其他 I/O 密集型线程性能同样也会下降。
设想一个等待用户输入的 UI 线程。
如果你让它和 CPU 密集型线程一起运行，那它通常是冻结的。
显然，这不是正常操作系统线程的工作模式，原因就在于 GIL。
它干扰了操作系统的调度。

CPython 的开发者们都知道这个问题。
他们称之为 “**护航效应**”（***convoy effect***）。
David Beazley 于 2010 年就它发表了一场 [演讲](https://www.youtube.com/watch?v=Obt-vMVdM8s)，并开了个 [与之有关的 issue，就在 bugs.pythong.org 上](https://bugs.python.org/issue7946)。
11 年后的 2021 年，这个 issue 被关闭了。
然而，这个问题没被修复。
本文的剩余部分，我们将试着去找出原因。

## 护航效应

护航效应的发生，是由于 I/O 密集型线程每次执行 I/O 操作时都会释放 GIL，且当它操作完后尝试重新获取 GIL 时，GIL 很可能已经被 CPU 密集型线程占据。
所以 I/O 密集型的线程必须至少等待 5 毫秒，然后才能设置 `eval_breaker` 和 `gil_drop_request` 来强制 CPU 密集型进程释放 GIL。

一旦 I/O 密集型线程释放了 GIL，操作系统可以立刻安排上 CPU 密集型线程。
而只有当 I/O 密集型线程完成了 I/O 操作后才可以被调度，所以它优先拿下 GIL 的机会少得多。
如果操作真得很快，如非阻塞的 [`send()`](https://man7.org/linux/man-pages/man2/send.2.html)，那么机会相当不错，但这也仅限于操作系统必须决定调度哪个线程的单核机器上。

在一个多核机器上，操作系统没必要决定调度两个线程中的哪一个。
它可以在不同的内核上安排二者。
结果就是 CPU 密集型线程几乎能保证优先获取 GIL，而 I/O 密集型线程中的每个 I/O 操作都要额外花费 5 毫秒。

请注意，被强制释放 GIL 的线程会等待另一个线程拿走 GIL，所以 I/O 密集型线程在一个切换间隔后能获得 GIL。
倘若没有这样的逻辑，护航效应会更严重。

接着，5 毫秒是多长呢？
这取决于 I/O 操作所花费的时间。
如果线程花费数秒才等到套接字上的数据可读，那么额外的 5 毫秒影响不大。
不过一些 I/O 操作的确快。
例如 [`send()`](https://man7.org/linux/man-pages/man2/send.2.html)，只有在缓冲区满了才会阻塞，否则会立即返回。
所以如果 I/O 操作开销只有数微秒，那么等待 GIL 的几毫秒影响巨大。

没有 CPU 密集型线程的回显服务器承载 30k RPS，这意味着单个请求开销大概 1/30k = 30 µs。
有了 CPU 密集型线程后，`recv()` 和 `send()` 为每个请求额外添上了 5 ms = 5,000 µs，现在单一请求开销达到了 10,030 µs。
约是 300 多倍。
因此吞吐量减少了 300 倍。
数字对得上。

你或许想问：护航效应是一个实际应用中的问题吗？
我不知道。
我从未遇到过这个问题，也找不到其他人遇到过的证据。
人们不会抱怨，这是该 issue 未被修复的一部分原因。

但若护航效应的的确确在你的应用中造成了性能问题该怎么办？
这里有两种方式解决它。

## 修复护航效应

由于问题出在 I/O 密集型线程上，它要等待一个切换间隔后才请求 GIL，所以我们可以为切换间隔设置一个更小的值。
Python 为此提供了 [`sys.setswitchinterval(interval)`](https://docs.python.org/3/library/sys.html#sys.setswitchinterval) 方法。
参数 `interval` 是个代表秒数的浮点数。
切换间隔以微秒计，故最小值为 `0.000001`。
这是我在改变切换间隔和 CPU 型的线程个数后得到的 RPS：

| 切换间隔（秒） | RPS (无 CPU 型线程) | RPS (1-CPU 型) | RPS (2-CPU 型) | RPS (4-CPU 型) |
| -------------- | ------------------- | -------------- | -------------- | -------------- |
| 0.1            | 30,000              | 5              | 2              | 0              |
| 0.01           | 30,000              | 50             | 30             | 15             |
| **0.005**      | **30,000**          | **100**        | **50**         | **30**         |
| 0.001          | 30,000              | 500            | 280            | 200            |
| 0.0001         | 30,000              | 3,200          | 1,700          | 1,000          |
| 0.00001        | 30,000              | 11,000         | 5,500          | 2,800          |
| 0.000001       | 30,000              | 10,000         | 4,500          | 2,500          |

结果说明了一些事情：

- 如果只有 I/O 密集型线程，则切换间隔无关紧要。
- 当我们加上一个 CPU 密集型线程时，RPS 显著下降。
- 当我们成倍添加 CPU 密集型线程数量时，RPS 减半。
- 当我们减少切换间隔，RPS 几乎成比例增加，直到切换间隔太小。这是由于上下文切换的开销变得突出。

更小的切换间隔为 I/O 密集型线程带来更灵敏的响应。
但过于小的切换间隔会有大量的上下文切换，进而带来巨大的系统开销。
回顾一下 `countdown()` 函数。
我们知道我们不能用多线程为它加速。
如果我们设置了很小的切换间隔，那么我们会看到速度变慢：

| 切换间隔（秒） | 耗时（1 线程） | 耗时（2 线程） | 耗时（4 线程） | 耗时（8 线程） |
| -------------- | -------------- | -------------- | -------------- | -------------- |
| 0.1            | 7.29           | 6.80           | 6.50           | 6.61           |
| 0.01           | 6.62           | 6.61           | 7.15           | 6.71           |
| **0.005**      | **6.53**       | **6.58**       | **7.20**       | **7.19**       |
| 0.001          | 7.02           | 7.36           | 7.56           | 7.12           |
| 0.0001         | 6.77           | 9.20           | 9.36           | 9.84           |
| 0.00001        | 6.68           | 12.29          | 19.15          | 30.53          |
| 0.000001       | 6.89           | 17.16          | 31.68          | 86.44          |

同样，只有一个线程时，切换间隔不造成什么影响。
而且，如果切换间隔足够大，线程的个数也不重要。
性能不佳出现在切换间隔小且线程数多时。

结论就是修改切换间隔是个修复护航效应的可选项，不过你要小心衡量这些变化会如何影响你的应用。

修复护航效应的第二个方式更笨拙。
由于问题在单核机器上不是那么严重，我们可以尝试将所有 Python 线程限制为单核。
这将会强制操作系统选择调度哪个线程，I/O 密集型线程将有优先权。

不是每个操作系统都提供了对应的方式，来限制一组线程运行在特定内核。
据我所知，macOS 只为操作系统的调度器提供了一个 [提醒机制](https://developer.apple.com/library/archive/releasenotes/Performance/RN-AffinityAPI/)。
而我们需要的那种机制在 Linux 上是可用的。
它就是 [`pthread_setaffinity_np()`](https://man7.org/linux/man-pages/man3/pthread_setaffinity_np.3.html) 函数。
它接受一个线程及一个 CPU 内核的掩码（*mask*），并告诉操作系统只在掩码指定的内核上调度线程。

`pthread_setaffinity_np` 是一个 C 函数。
要从 Python 调用它，你可以借助类似 [`ctypes`](https://docs.python.org/3/library/ctypes.html) 之类的东西。
我不想弄乱 `ctypes`，所以只修改了 CPython 的源代码。
之后，编译出可执行文件，并且在双核 Ubuntu 机器上运行回显服务器，得到的结果如下：

| CPU 密集型线程数 | 0   | 1   | 2   | 4   | 8   |
| ---------------- | --- | --- | --- | --- | --- |
| RPS              | 24k | 12k | 3k  | 30  | 10  |

该服务可以很好地容忍有一个 CPU 密集线程线程的情况。
不过由于 I/O 密集型线程需要和全部 CPU 密集型线程竞争 GIL，当我们添加更多线程，性能下降极大。
这种修复过于侵入式了。
为什么 CPython 的开发者们不直接实现一个合理的 GIL 呢？

**2021 年 10 月 7 日的更新：** 我现在了解到把线程限制在单核上的做法，只有当客户端也在同一个内核上时，才有助于改善护航效应，而这是我在对比实验里的做法。
详见 [后记](#后记)。

## 一个合理的 GIL

GIL 的根本问题在于它干扰了操作系统的调度器。
理想状态下，你希望在 I/O 密集型线程等待的 I/O 操作结束时，立刻运行该线程。
并且这就是操作系统调度器通常所为。
然而，在 CPython 中，线程在（I/O 操作结束）之后会立刻陷入等待 GIL 的状态中，所以操作系统调度器的决定实际上没什么意义。
你可以尝试舍弃掉切换间隔，如此，想要得到 GIL 的线程能毫无延迟地拿到 GIL，但这样你又会遇到 CPU 密集型线程的问题，因为它们无时无刻不想要 GIL。

合理的解决方案是区分线程。
I/O 密集型线程应该无需等待就从 CPU 密集型线程处拿走 GIL，但是相同优先级的线程应该互相等待。
操作系统调度器已经区分了线程，不过你不能依赖它，因为它对 GIL 是一无所知的。
似乎唯一的选择是在解释器里实现调度逻辑了。

自从 David Beazley 创建了这个 [issue](https://bugs.python.org/issue7946) 后，CPython 的开发者们多次尝试解决它。
Beazley 自己提出了个 [简易的补丁](http://dabeaz.blogspot.com/2010/02/revisiting-thread-priorities-and-new.html)。
简单来说，该补丁允许 I/O 密集型线程抢占 CPU 密集型线程的资源。
默认情况下，所有线程都被认为是 I/O 密集型的。
一旦一个线程被强制释放 GIL，它就被标记为 CPU 密集型的。
当一个线程主动释放 GIL 时，这个标记会被重置，线程又被视作是 I/O 密集型的了。

Beazley 的补丁解决了我们今时讨论的所有 GIL 问题。
为什么它没被合入呢？
大家的共识似乎是任何简单的 GIL 实现都存在一些使之失效的病态案例（*pathological case*）。
找到这些案例顶多需要多费点力气。
一个合适的解决方法必须做到调度得像操作系统那样，或者像 Nir Aides 所说：

> ... Python really needs a scheduler, not a lock.
> 
> ... Python 确切需要一个调度器，而不是一把锁。

所以 Aides 也在 [他的补丁](https://bugs.python.org/issue7946#msg101612) 里实现了一个成熟的调度器。
补丁是有效的，但调度器不是件小事，因此合并入 CPython 需要付出许多努力。
最终，这项工作被废止了，因为当时没有足够的证据表明这个 issue 在生产代码中造成了困扰。
[讨论](https://bugs.python.org/issue7946) 中有更多的细节。

GIL 从未有过很大的粉丝群体。
我们今日所见只是它的情况更糟了。
回到一直以来的那个问题吧。

## 我们不能移除 GIL 吗？

删除 GIL 的第一步是理解它为什么存在。
想想为什么你往往要在多线程程序里使用锁，你就得到答案了。
它是为了防止竞态条件（*race condition*），并使某些操作从其他线程来看具有原子性（*atomic*）。
假设你有一个修改某些数据结构的语句序列。
如果你不为这段序列加锁，那么另一个线程就能在修改过程中（执行到）某语句处时访问到这些数据结构，进而所得的将会是破损的、不完整的。

或者假设你在多个线程里对同一个变量执行增量（*increment*）运算。
如果增量操作是非原子性的，并且也没有被锁保护，那么变量的最后的数值会小于增量的总次数。
以下是典型的数据争用（*data race*）：

1. 线程 1 读取 `x` 的值。
2. 线程 2 读取 `x` 的值。
3. 线程 1 写入 `x + 1` 的值。
4. 线程 2 写入 `x + 1` 的值，从而丢弃了线程 1 所做的改动。

在 Python 中，`+=` 操作符是非原子性的，因为它由多个字节码指令构成。
将切换间隔设置为 0.000001 并且在多线程内执行以下函数，来看看它是如何导致数据争用的：

```Python
sum = 0
def f():
    global sum
    for _ in range(1000):
        sum += 1
```

类似地，C 中诸如 `x++` 或 `++x` 的整型增量也是非原子性的，因为编译器会将类似的操作翻译成一系列机器指令。
线程之间互有交错。

GIL 如此有必要，得益于 CPython 到处都在增减那些在多个线程内共享的整型。
这是 CPython 实现垃圾回收的方式。
每个 Python 对象都有一个引用计数（*reference count*）的字段。
该字段统计了引用相应对象的次数，引用来源包括：其他 Python 对象、局部和全局 C 变量。
多一个引用处会增加引用计数。
少一个引用处会减少。
当引用计数达到零时，对象就会被释放。
如果不是 GIL，一个减量（*decrement*）操作会互相覆盖，对象将永远留在内存中。
更糟糕的是，被覆盖的增量操作可能会导致释放了仍在被引用的对象。

GIL 还简化了内置的可变数据结构的实现。
列表（`list`）、字典（`dict`）和集合（`set`）内部不使用锁，因为 GIL 的存在，它们可被安全地用于多线程程序中。
同理，GIL 允许线程安全地访问全局或解释器范围内的数据：加载的模块、预分配的对象、驻留的字符串（*interned strings*）等等。

最后一点，GIL 简化了 C 拓展的编写。
开发者们可以假设在任意时刻只有一个线程执行他们的 C 拓展。
故而，他们不需要使用额外的锁来保证代码的线程安全。
若他们的确希望并行运行代码时，可以释放 GIL。

总结一下，GIL 所做的，是保证了以下内容的线程安全：

1. 引用计数；
2. 可变数据结构；
3. 全局和解释器范围内的数据；
4. C 拓展。

若要移除 GIL 而维持一个可用的解释器，你需要为线程安全提供替代方案。
过去人们试图这么做了。
最引人瞩目的是 Larry Hastings 始于 2016 年的 Gilectomy 项目。
Hastings [克隆](https://github.com/larryhastings/gilectomy) 了 CPython，[移除](https://github.com/larryhastings/gilectomy/commit/4a1a4ff49e34b9705608cad968f467af161dcf02) 了 GIL，并用具有 [原子性](https://gcc.gnu.org/onlinedocs/gcc-4.1.1/gcc/Atomic-Builtins.html) 的增减操作修改引用计数，并放置了大量细粒度的（*fine-grained*）锁来保护可变数据结构及解释器范围内的数据。

Gilectomy 可以运行部分 Python 代码，且是并行运行的。
然而 CPython 的单线程性能受到了影响。
光是原子性的增减操作就增加了大约 30% 的系统开销。
Hastings 试图通过实现带缓冲的引用计数来解决这个问题。
简单来说，那项技术把所有引用计数的更新都限制在一个特殊线程里。
其他线程只把增量和减量提交到日志，而特殊线程读取日志。
它有效，但开销还是很明显。

到了最后，[显然](https://lwn.net/Articles/754577/) Gilectomy 不会被合入 CPython 中。
Hastings 停止了该项目上的工作。
尽管如此，这不算彻底失败。
它教会了我们为什么移除 GIL 是那么难。
主要原因有二：

1. 基于引用计数的垃圾回收不大适合于多线程。唯一的解决方案是实现一个 [追踪式垃圾回收器](https://en.wikipedia.org/wiki/Tracing_garbage_collection)，一如 JVM、CLR、GO 和其他没有 GIL 实现的运行时（*runtime*）那样。
2. 移除 GIL 会破坏现有的 C 语言拓展。这是没办法的事。

现在没人在认真考虑移除 GIL 这件事。
这是否意味着我们得与 GIL 永远共存了？

## GIL 和 Python 并发的未来

听起来很骇人，但相比完全没有 GIL，CPython 将有多个 GIL 的可能性要大得多。
有一项倡议要将多个，字面意义上的，GIL 引入 CPython。
这被称为子解释器（*subinterpreter*）。
其思路是在同个进程内拥有多个解释器。
同一解释器下的线程仍共享 GIL，不过多个解释器可以并行运行。
这不需要 GIL 来同步解释器，因为它们没有共同的全局状态也不共享 Python 对象。
所有全局状态分布到各个解释器中，解释器间的通信只靠消息传递。
最终目标是在 Python 中引入基于 Go、Clojure 等语言中的通信顺序进程（*communicating sequential process*）的并发模型。

解释器自 1.5 版本起就已经成为了 CPython 的一部分了，不过只作为一种隔离机制。
它们存储特定某组线程的数据：载入的模块、内建的程序和导入的设置等等。
它们没有暴露给 Python，但 C 拓展可以借由 Python/C API 使用之。
一些拓展确实这样做了，[`mod_wsgi`](https://modwsgi.readthedocs.io/en/develop/index.html) 是个值得注意的例子。

现在解释器受到了限制，因为它们不得不共享 GIL。
只有当全局状态分散到各个解释器上进行，这才可能改变。
工作在朝这个方向 [进行](https://pythondev.readthedocs.io/subinterpreters.html)，不过有些东西还是全局的：一些内建类型、`None`、`False`、`True` 等单例，和部分内存分配器。
C 拓展在用上子解释器前，也需要 [摆脱全局状态](https://www.python.org/dev/peps/pep-0630/)。

Eric Snow 写了 [PEP 554](https://www.python.org/dev/peps/pep-0554/)，往标准库中添加了 `interpreters` 模块。
该提案是为了把解释器现有的 C API 暴露给 Python，并在解释器间提供一套通信机制。
它规划在 Python 3.9，但被推迟了，得等到 GIL 被做到各个解释器上才行。
即便如此也还不能保证成功。
[争论](https://mail.python.org/archives/list/python-dev@python.org/thread/3HVRFWHDMWPNR367GXBILZ4JJAUQ2STZ/) 的要点在于 Python 是否确实需要另外的并发模型。

如今进行中的另一个鼓舞人心的项目叫做 [Faster CPython](https://github.com/faster-cpython)。
在 2020 年 10 月， Mark Shannon 提出了一个 [计划](https://github.com/markshannon/faster-cpython)，要在几年内让 CPython 速度加快到 5 倍。
而且实际上，这远比听起来更现实，因为 CPython 有巨大的优化空间。

以前有过类似的项目，因为资金或专业知识的匮乏而失败。
这一次，微软主动 [赞助](https://lwn.net/Articles/857754/) 了 Faster CPython 并让 Mark Shannon，Guido van Rossum 和 Eric Snow 从事该项目。
一些增量变动已经并入了 CPython 中——它们不会沉寂在分支里了。

Faster CPython 专注于单线程的性能。
该团队没有计划修改或移除 GIL。
尽管如此，如果项目成功，Python 主要的痛点之一将被解决，GIL 问题可能会比以前变得更有意义。

## 附言

本文使用的各个测试对比实验可以 [在 GitHub 上找到](https://github.com/r4victor/pbts13_gil)。
特别感谢 David Beazley 的 [精彩演讲](https://dabeaz.com/talks.html)。
Larry Hastings 关于 GIL 和 Gilectomy 的演讲（[1](https://www.youtube.com/watch?v=KVKufdTphKs)、[2](https://www.youtube.com/watch?v=P3AyI_u66Bw)、[3](https://www.youtube.com/watch?v=pLqv11ScGsQ)）也非常有趣，值得一看。
为了解现代操作系统调度器是如何工作的，我还读了 Robert Love 的书[《Linux 内核设计与实现》*（Linux Kernel Development）*](https://www.amazon.com/Linux-Kernel-Development-Robert-Love/dp/0672329468)。
强烈推荐！

如果你想更详细地研究 GIL，你应该读读源码。
[`Python/ceval_gil.h`](https://github.com/python/cpython/blob/3.9/Python/ceval_gil.h) 文件是一个完美的起点。
我在后文附带的章节里，为你的这项冒险提供了点帮助。

## GIL 的实现细节 *

技术层面来说，GIL 是个标识 GIL 是否被锁定的标志，一组控制该标志如何被设置的互斥量（*mutex*）和条件变量（*conditional variable*），以及一些如切换间隔的其他实用变量。
所有这些都存在 `_gil_runtime_state` 结构体中：

```C
struct _gil_runtime_state {
    /* 毫秒（不过 Python API 用的是秒） */
    unsigned long interval;
    /* 最后一个持有或曾持有 GIL 的 PyThreadState。
       这帮助我们理解在丢弃 GIL 后是否有其他线程被调度。 */
    _Py_atomic_address last_holder;
    /* GIL 是否被取走了（未初始化时为 -1）。
       这是原子性的，因为在 ceval.c 里可以不加任何锁就读取它。 */
    _Py_atomic_int locked;
    /* 开始以来 GIL 切换的次数 */
    unsigned long switch_number;
    /* 该条件变量允许一个或多个线程等待 GIL 的释放。
       此外，此互斥量还会保护上面的条件变量。 */
    PyCOND_T cond;
    PyMUTEX_T mutex;
#ifdef FORCE_SWITCHING
    /* 该条件变量帮助释放 GIL 的线程等待，
       直到某个等待 GIL 的线程被调度并取走 GIL。 */
    PyCOND_T switch_cond;
    PyMUTEX_T switch_mutex;
#endif
};
```

`_gil_runtime_state` 结构体是全局状态的一部分。
它被存储在 `_ceval_runtime_state` 结构体里，作为 `PyRuntimeState` 的一部分，可被所有 Python 线程访问：

```C
struct _ceval_runtime_state {
    _Py_atomic_int signals_pending;
    struct _gil_runtime_state gil;
};
```

```C
typedef struct pyruntimestate {
    // ...
    struct _ceval_runtime_state ceval;
    struct _gilstate_runtime_state gilstate;

    // ...
} _PyRuntimeState;
```

注意 `_gilstate_runtime_state` 是个与 `_gil_runtime_state` 不同的结构体。
它存放的信息，是持有 GIL 的线程的：

```C
struct _gilstate_runtime_state {
    /* bpo-26558: Flag to disable PyGILState_Check().
       如果设置为非零值，PyGILState_Check() 总会返回 1。 */
    int check_enabled;
    /* 假设当前线程持有 GIL，PyThreadState 指向的就是当前线程。 */
    _Py_atomic_address tstate_current;
    /* 进程的 GILState 实现所使用的单个 PyInterpreterState */
    /* TODO: Given interp_main, it may be possible to kill this ref */
    PyInterpreterState *autoInterpreterState;
    Py_tss_t autoTSSkey;
};
```

最后还有个 `_ceval_state` 结构体，是 `PyInterpreterState` 的一部分。
它存储 `eval_breaker` 和 `gil_drop_request` 标志：

```C
struct _ceval_state {
    int recursion_limit;
    int tracing_possible;
    /* 该单个变量整合了所有请求，以快速跳出求值循环。 */
    _Py_atomic_int eval_breaker;
    /* 丢弃 GIL 的请求 */
    _Py_atomic_int gil_drop_request;
    struct _pending_calls pending;
};
```

Python/C API 提供了 [`PyEval_RestoreThread()`](https://docs.python.org/3/c-api/init.html#c.PyEval_RestoreThread) 和 [`PyEval_SaveThread()`](https://docs.python.org/3/c-api/init.html#c.PyEval_SaveThread) 函数来获取或释放 GIL。
这些函数还负责设置 `gilstate->tstate_current`。
在这背后，是 [`take_gil()`](https://github.com/python/cpython/blob/5d28bb699a305135a220a97ac52e90d9344a3004/Python/ceval_gil.h#L211) 和 [`drop_gil()`](https://github.com/python/cpython/blob/5d28bb699a305135a220a97ac52e90d9344a3004/Python/ceval_gil.h#L144) 完成了所有的工作。
当持有 GIL 的线程暂停执行字节码时，它们会被调用：

```C
/* 处理信号，挂起调用，请求丢弃 GIL 和异步异常 */
static int
eval_frame_handle_pending(PyThreadState *tstate)
{
    _PyRuntimeState * const runtime = &_PyRuntime;
    struct _ceval_runtime_state *ceval = &runtime->ceval;

    /* 挂起信号 */
    // ...

    /* 挂起调用 */
    struct _ceval_state *ceval2 = &tstate->interp->ceval;
    // ...

    /* 请求丢弃 GIL */
    if (_Py_atomic_load_relaxed(&ceval2->gil_drop_request)) {
        /* 给其他线程一个机会 */
        if (_PyThreadState_Swap(&runtime->gilstate, NULL) != tstate) {
            Py_FatalError("tstate mix-up");
        }
        drop_gil(ceval, ceval2, tstate);

        /* 其他线程现在可以运行了 */

        take_gil(tstate);

        if (_PyThreadState_Swap(&runtime->gilstate, tstate) != NULL) {
            Py_FatalError("orphan tstate");
        }
    }

    /* 检查异步异常。 */
    // ...
}
```

在类 Unix 系统中，GIL 的实现依赖于 [pthreads](https://man7.org/linux/man-pages/man7/pthreads.7.html) 库提供的原语（*primitives*）。
这包括了互斥量和条件变量。
简而言之，他们的工作方式如下。
一个线程调用 [`pthread_mutex_lock(mutex)`](https://linux.die.net/man/3/pthread_mutex_lock) 来锁定互斥量。
当另一个线程执行相同操作时，就会阻塞。
操作系统会把它放进等待互斥量的线程队列中，并在第一个线程调用 [`pthread_mutex_unlock(mutex)`](https://linux.die.net/man/3/pthread_mutex_unlock) 时唤醒它。
一次只有一个线程可以运行受保护的代码。

条件变量允许某线程等待，直到另一线程使某些条件为真。
要等待某个条件变量，线程锁定一个互斥量并调用 [`pthread_cond_wait(cond, mutex)`](https://linux.die.net/man/3/pthread_cond_wait) 或 [`pthread_cond_timedwait(cond, mutex, time)`](https://linux.die.net/man/3/pthread_cond_wait)。
这些调用原子化地释放了互斥量并阻塞线程。
操作系统将它放入等待队列，并在另一个线程调用 [`pthread_cond_signal()`](https://linux.die.net/man/3/pthread_cond_signal) 时唤醒它。
被唤醒的线程再次锁定互斥量并继续执行。
以下是条件变量的通常用法：

```Python
# 等待线程

mutex.lock()
while not condition:
    cond_wait(cond_variable, mutex)
# ... 条件为真，做点什么
mutex.unlock()
```

```Python
# 信号线程

mutex.lock()
# ... 做点什么并使条件为真
cond_signal(cond_variable)
mutex.unlock()
```

注意，等待中的线程需要循环检查其条件，因为 [不能保证](https://stackoverflow.com/questions/7766057/why-do-you-need-a-while-loop-while-waiting-for-a-condition-variable) 在线程被通知后条件仍为真。
互斥量只确保等待线程不会错过条件从假变为真的时刻。

`take_gil()` 和 `drop_gil()` 函数使用 `gil->cond` 条件变量来提醒等待 GIL 的线程：GIL 已经被释放；并使用 `gil->switch_cond` 来提醒持有 GIL 的线程：其他线程取走了 GIL。
这俩条件变量受两个互斥量保护：`gil->mutex` 和 `gil->switch_mutex`。

以下是 [`take_gil()`](https://github.com/python/cpython/blob/5d28bb699a305135a220a97ac52e90d9344a3004/Python/ceval_gil.h#L211) 的步骤：

1. 锁定 GIL mutex：`pthread_mutex_lock(&gil->mutex)`。
2. 检查 `gil->locked`。如果未被锁，跳转到 4。
3. 等待 GIL。若 `gil->locked`：
    1. 记住 `gil->switch_number`。
    2. 等待持有 GIL 的线程舍弃 GIL：`pthread_cond_timedwait(&gil->cond, &gil->mutex, switch_interval)`。
    3. 如果超时，而 `gil->locked`，并且 `gil->switch_number` 还未变化，告知持有 GIL 的线程舍弃 GIL：设置 `ceval->gil_drop_request` 和 `ceval->eval_breaker`。
4. 取得 GIL 并通知持有 GIL 的线程，我们拿走了 GIL：
    1. 锁定 switch mutex：`pthread_mutex_lock(&gil->switch_mutex)`。
    2. 设置 `gil->locked`。
    3. 如果我们不是 `gil->last_holder` 线程，更新 `gil->last_holder` 并增加 `gil->switch_number`。
    4. 通知释放了 GIL 的线程，我们拿走了 GIL：`pthread_cond_signal(&gil->switch_cond)`。
    5. 释放 switch mutex：`pthread_mutext_unlock(&gil->switch_mutex)`。
5. 重置 `ceval->gil_drop_request`。
6. 重新计算 `ceval->eval_breaker`。
7. 释放 GIL mutex：`pthread_mutex_unlock(&gil->mutex)`。

注意，当某线程在等待 GIL 时，另一个（也在等待的）线程可以取走它，所以检查 `gil->switch_number` 来确保刚取走 GIL 的线程不会被强制释放。

最后，这是 [`drop_gil()`](https://github.com/python/cpython/blob/5d28bb699a305135a220a97ac52e90d9344a3004/Python/ceval_gil.h#L144) 的步骤：

1. 锁定 GIL mutex：`pthread_mutex_lock(&gil->mutex)`。
2. 重置 `gil->locked`。
3. 通知等待 GIL 的线程，我们舍弃了 GIL：`pthread_cond_singal(&gil->cond)`。
4. 释放 GIL mutex：`pthread_mutex_unlock(&gil->mutex)`。
5. 如果 `ceval->gil_drop_request`，等待另一个线程取走 GIL：
    1. 锁定 switch mutex：`pthread_mutext_lock(&gil->switch_mutex)`。
    2. 如果我们还是 `gil->last_holder`，等待：`pthread_cond_wait(&gil->switch_cond, &gil->switch_mutex)`。
    3. 释放 switch mutex：`pthread_mutext_unlock(&gil->switch_mutex)`。

注意，释放 GIL 线程不需要循环等待一个条件。
它调用 `pthread_cond_wait(&gil->switch_cond, &gil->switch_mutex)` 只是确保不会立刻重新获取 GIL。
如果发生了切换，这表明另一个线程取走了 GIL，再次争夺 GIL 就行。 

> *If you have any questions, comments or suggestions, feel free to contact me at victor@tenthousandmeters.com*
>
> 如果你有任何疑问、意见或建议，请随时通过 victor@tenthousandmeters.com 与我联系。

## 后记

**2021 年 10 月 7 日更新：** 将线程限制在单核上没有实际解决护航效应。
是的，它强制操作系统选择调度两线程中的一个，使得 I/O 密集型线程有更好的机会获得 GIL 用于 I/O 操作上，但如果 I/O 操作在阻塞中，它就无能为力了。
在这种情况下，I/O 密集型线程没有真正准备好受调度，所以操作系统会调度 CPU 密集型线程。

在回显服务器的例子中，实际上每个 `recv()` 都是阻塞的——服务器等待客户端读取响应并发送下一条消息。
限制线程到单核上没有帮助。
那我们看到了 RPS 提高了。
为什么呢？
因为对比实验有缺陷。
我在同一个机器上运行了客户端，和服务端线程在同一个内核上。
当服务端的 I/O 密集线程在 `recv()` 上阻塞时，这种操作迫使操作系统在服务端的 CPU 密集型线程和客户端线程中做出选择。
客户端线程更有可能被调度。
它发送下一条消息，且也阻塞在 `recv()` 上。
不过此刻，服务端的 I/O 密集型线程就准备好与 CPU 密集型线程竞争了。
事实上，是在同一个内核上运行客户端，才使得操作系统在 I/O 密集型线程和 CPU 密集型线程间做出的选择，即使 `recv()` 是阻塞的。

同样，你不需要修改 CPython 源码或弄乱 `ctypes` 来限制 Python 线程在特定的内核上运行。
在 Linux 中，`pthread_setaffinity_np()` 函数是在 [`sched_setaffinity()`](https://man7.org/linux/man-pages/man2/sched_setaffinity.2.html) 的系统调用上实现的，而且 `os` 标准模块将这个系统调用 [暴露](https://docs.python.org/3/library/os.html#os.sched_setaffinity) 给了 Python。
感谢 Carl Bordum Hansen 替我指出了这点。

同样还有 [`taskset`](https://man7.org/linux/man-pages/man1/taskset.1.html) 命令，可以让你完全不改变源码就设置某进程的 CPU 亲和力（*affinity*）。
只要像这样运行程序：

```Shell
$ taskset -c {cpu_list} python program.py
```

**2021 年 10 月 16 日的更新：** Sam Gross 最近 [宣布了](https://mail.python.org/archives/list/python-dev@python.org/thread/ABR2L6BENNA6UPSPKV474HCS4LWT26GY/) 移除 GIL 后的 CPython 分支。
你可以把该项目视作 Gilectomy 2.0：它用线程安全的替代机制替换了 GIL，但与 Gilectomy 不同，它不会让单线程代码变慢。
实际上，Gross 优化了解释器，使得在没有 GIL 的分支上，单线程性能比主线（*mainline*）的 CPython 3.9 更强。

这个项目看上去是最有希望从 CPython 移除 GIL 的尝试。
我确信 Gross 提出的一些想法会出现在上游里。
想要了解有关该项目更多的内容及其中的观点，请参考 [设计文档](https://docs.google.com/document/d/18CXhDb1ygxg-YXNBJNzfzZsDFosB5e6BfnXLlejd9l0/) 和 [GitHub 仓库](https://github.com/colesbury/nogil)。
LWN 上也有一份不错的 [总结](https://lwn.net/SubscriberLink/872869/0e62bba2db51ec7a/)。