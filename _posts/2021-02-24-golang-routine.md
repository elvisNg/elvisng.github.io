---
layout: post
title: "Golang-Routine"
subtitle: 'G-P-M'
author: "Elvis"
header-style: text
tags:
  - golang
  - 技术分享
  - GPM模型

---

### Goroutine基础知识：

**并发:** 逻辑上具有处理多个同时性任务的能力。

**并行:**  物理上同一时刻执行多个并发任务。

通常所说的并发编程，也就是说它允许多个任务同时执行，但实际上并不一定在同一时刻被执行。在单核处理器上，通过多线程共享CPU时间片串行执行(并发非并行)。而并行则依赖于多核处理器等物理资源，让多个任务可以实现并行执行(并发且并行)。

多线程或多进程是并行的基本条件，但单线程也可以用协程(coroutine)做到并发。简单将Goroutine归纳为协程并不合适，因为它运行时会创建多个线程来执行并发任务，且任务单元可被调度到其它线程执行。这更像是多线程和协程的结合体，能最大限度提升执行效率，发挥多核处理器能力。

Go编写一个并发编程程序很简单，只需要在函数之前使用一个Go关键字就可以实现并发编程。

```go
func main() {    go func(){
        fmt.Println("Hello,World!")
    }()
}
```

Go调度器组成

进程：进程是系统进行资源分配的基本单位，有独立的内存空间。

线程：线程是 CPU 调度和分派的基本单位，线程依附于进程存在，每个线程会共享父进程的资源。

**协程：协程是一种用户态的轻量级线程**，协程的调度完全由用户控制，协程间切换只需要保存任务的上下文，没有内核的开销。

#### 线程上下文切换

由于中断处理，多任务处理，用户态切换等原因会导致 CPU 从一个线程切换到另一个线程，切换过程需要保存当前进程的状态并恢复另一个进程的状态。

**上下文切换的代价是高昂的**，因为在核心上交换线程会花费很多时间。上下文切换的延迟取决于不同的因素，大概在在 50 到 100 纳秒之间。考虑到硬件平均在每个核心上每纳秒执行 12 条指令，那么一次上下文切换可能会花费 600 到 1200 条指令的延迟时间。实际上，上下文切换占用了大量程序执行指令的时间。

如果存在**跨核上下文切换**（Cross-Core Context Switch），可能会导致 CPU 缓存失效（CPU 从缓存访问数据的成本大约 3 到 40 个时钟周期，从主存访问数据的成本大约 100 到 300 个时钟周期），这种场景的切换成本会更加昂贵。



### Golang 并发

Goroutine 非常轻量，主要体现在以下两个方面：

* 上下文切换代价小

* 内存占用少：线程栈空间通常是 2M，Goroutine 栈空间最小 2K；



### Go 调度器实现机制：GPM

> Go 调度器模型我们通常叫做**G-P-M 模型**，他包括 4 个重要结构，分别是**G、P、M、Sched**：



**G:Goroutine**，每个 Goroutine 对应一个 G 结构体，G 存储 Goroutine 的运行堆栈、状态以及任务函数，可重用。

G 并非执行体，每个 G 需要绑定到 P 才能被调度执行。

**P: Processor**，表示逻辑处理器，对 G 来说，P 相当于 CPU 核，G 只有绑定到 P 才能被调度。对 M 来说，P 提供了相关的执行环境(Context)，如内存分配状态(mcache)，任务队列(G)等。

**M: Machine**，OS 内核线程抽象，代表着真正执行计算的资源，在绑定有效的 P 后，进入 schedule 循环；而 schedule 循环的机制大致是从 Global 队列、P 的 Local 队列以及 wait 队列中获取。



**Sched：Go 调度器**，它维护有存储 M 和 G 的队列以及调度器的一些状态信息等。

调度器循环的机制大致是从各种队列、P 的本地队列中获取 G，切换到 G 的执行栈上并执行 G 的函数，调用 Goexit 做清理工作并回到 M，如此反复。

> 在Go的1.0版本实现中并没有P的概念，Go中的调度器直接将G分配到合适的M上运行。但这样带来了很多问题，例如，不同的G在不同的M上并发运行时可能都需向系统申请资源（如堆内存），由于资源是全局的，将会由于资源竞争造成很多系统性能损耗。
>

详细要了解P是怎么管理内存的可以看这篇：

[go内存分配](https://elvisng.github.io/2020/11/27/golang-memory/)



### G-P-M模型示意图:

![img](https://img2020.cnblogs.com/blog/982408/202011/982408-20201114161958685-1671778105.png)

Go 调度器中有两个不同的运行队列：**全局运行队列(GRQ)和本地运行队列(LRQ)。**

每个 P 都有一个 LRQ，用于管理分配给在 P 的上下文中执行的 Goroutines，这些 Goroutine 轮流被和 P 绑定的 M 进行上下文切换。GRQ 适用于尚未分配给 P 的 Goroutines。

**从上图可以看出，G 的数量可以远远大于 M 的数量，换句话说，Go 程序可以利用少量的内核级线程来支撑大量 Goroutine 的并发。多个 Goroutine 通过用户级别的P上下文切换来共享内核线程 M 的计算资源，对于操作系统来说就少了线程管理和上下文切换，之所以GO这么快的原因了。**

### 调度策略

**为了更加充分利用线程的计算资源，Go 调度器采取了以下几种调度策略**：

#### **任务窃取（work-stealing）**

我们知道，现实情况有的 Goroutine 运行的快，有的慢，那么势必肯定会带来的问题就是，忙的忙死，闲的闲死，Go 肯定不允许摸鱼的 P 存在，势必要充分利用好计算资源。

为了提高 Go 并行处理能力，调高整体处理效率，当每个 P 之间的 G 任务不均衡时，调度器允许从 GRQ，或者其他 P 的 LRQ 中获取 G 执行。

#### **减少阻塞**

如果正在执行的 Goroutine 阻塞了线程 M 怎么办？P 上 LRQ 中的 Goroutine 会获取不到调度么？

#### **在 Go 里面阻塞主要分为一下 4 种场景：**

##### 场景 1：

**由于原子、互斥量或通道操作调用导致 Goroutine 阻塞**，调度器将把当前阻塞的 Goroutine 切换出去，重新调度 LRQ 上的其他 Goroutine；

##### 场景 2：

**由于网络请求和 IO 操作导致 Goroutine 阻塞**，这种阻塞的情况下，我们的 G 和 M 又会怎么做呢？

Go 程序提供了**网络轮询器（NetPoller）**来处理网络请求和 IO 操作的问题，其后台通过 kqueue（MacOS），epoll（Linux）或 iocp（Windows）来实现 IO 多路复用。

通过使用 NetPoller 进行网络系统调用，调度器可以防止 Goroutine 在进行这些系统调用时阻塞 M。这可以让 M 执行 P 的 LRQ 中其他的 Goroutines，而不需要创建新的 M。有助于减少操作系统上的调度负载。

**下图展示它的工作原理**：G1 正在 M 上执行，还有 3 个 Goroutine 在 LRQ 上等待执行。网络轮询器空闲着，什么都没干。 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200515095321466.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1aWdhbg==,size_16,color_FFFFFF,t_70)

接下来，G1 想要进行网络系统调用，因此它被移动到网络轮询器并且处理异步网络系统调用。然后，M 可以从 LRQ 执行另外的 Goroutine。此时，G2 就被上下文切换到 M 上了。 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200515095356925.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1aWdhbg==,size_16,color_FFFFFF,t_70)

最后，异步网络系统调用由网络轮询器完成，G1 被移回到 P 的 LRQ 中。一旦 G1 可以在 M 上进行上下文切换，它负责的 Go 相关代码就可以再次执行。这里的最大优势是，执行网络系统调用不需要额外的 M。网络轮询器使用系统线程，它时刻处理一个有效的事件循环。 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200515095420784.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1aWdhbg==,size_16,color_FFFFFF,t_70)

这种调用方式看起来很复杂，值得庆幸的是，**Go 语言将该“复杂性”隐藏在 Runtime 中**：Go 开发者无需关注 socket 是否是 non-block 的，也无需亲自注册文件描述符的回调，只需在每个连接对应的 Goroutine 中以“block I/O”的方式对待 socket 处理即可，**实现了 goroutine-per-connection 简单的网络编程模式**（但是大量的 Goroutine 也会带来额外的问题，比如栈内存增加和调度器负担加重）。

用户层眼中看到的 Goroutine 中的“block socket”，实际上是通过 Go runtime 中的 netpoller 通过 Non-block socket + I/O 多路复用机制“模拟”出来的。Go 中的 net 库正是按照这方式实现的。

**场景 3：**当调用一些系统方法的时候，如果系统方法调用的时候发生阻塞，这种情况下，网络轮询器（NetPoller）无法使用，而进行系统调用的 Goroutine 将阻塞当前 M。

让我们来看看同步系统调用（如文件 I/O）会导致 M 阻塞的情况：G1 将进行同步系统调用以阻塞 M1。 ![在这里插入图片描述](https://img-blog.csdnimg.cn/2020051509550612.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1aWdhbg==,size_16,color_FFFFFF,t_70)

调度器介入后：识别出 G1 已导致 M1 阻塞，此时，调度器将 M1 与 P 分离，同时也将 G1 带走。然后调度器引入新的 M2 来服务 P。此时，可以从 LRQ 中选择 G2 并在 M2 上进行上下文切换。 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200515095526409.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1aWdhbg==,size_16,color_FFFFFF,t_70)

阻塞的系统调用完成后：G1 可以移回 LRQ 并再次由 P 执行。如果这种情况再次发生，M1 将被放在旁边以备将来重复使用。 ![在这里插入图片描述](https://img-blog.csdnimg.cn/20200515095550341.jpg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2h1aWdhbg==,size_16,color_FFFFFF,t_70)

**场景 4：**如果在 Goroutine 去执行一个 sleep 操作，导致 M 被阻塞了。

Go 程序后台有一个监控线程 sysmon，它监控那些长时间运行的 G 任务然后设置可以强占的标识符，别的 Goroutine 就可以抢先进来执行。

只要下次这个 Goroutine 进行函数调用，那么就会被强占，同时也会保护现场，然后重新放入 P 的本地队列里面等待下次执行。



