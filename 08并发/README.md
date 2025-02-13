1. [为什么我们需要并发？](#1-wei-shen-me-wo-men-xu-yao-bing-fa)
2. [什么是竞争条件？举一个例子来描述？](#2-shen-me-shi-jing-zheng-tiao-jian-ju-yi-ge-li-zi-lai-miao-shu)
3. [什么是死锁？编写一些代码是由死锁导致的；](#3-shen-me-shi-si-suo-bian-xie-yi-xie-dai-ma-shi-you-si-suo-dao-zhi-de)
4. [什么是进程饥饿？如果需要，重新回顾一下定义](#4-shen-me-shi-jin-cheng-jie-ru-guo-xu-yao-zhong-xin-hui-gu-yi-xia-ding-yi)
5. [什么是Wait Free算法？](#6-shen-me-shi-wait-free-suan-fa)
6. [为什么测试多线程或者并发比较困难？](#9-wei-shen-me-ce-shi-duo-xian-cheng-huo-zhe-bing-fa-bi-jiao-kun-nan)

## 1 为什么我们需要并发？

首先，并发(`concurrency`)和并行 (`parallelism`)是两个不同的概念

- 并发是逻辑上同时多个任务执行；
- 并行是物理上同时多个任务执行；

对于单核CPU处理器，无论如何设计，都是并行执行的代码，而并发可以通过CPU分配时间片来完成。在计算机早期的时候，所有机器在同一个时刻只运行一个程序，也叫单任务处理器。在后来随着个人电脑（PC）的兴起，我们对计算机有了更高的要求，比如在浏览网页的时候，也可以通过音乐播放器播放音乐，而后台的电子邮件程序不停的接受邮件。这样我们设计出多任务处理器，将CPU运行划分为一个个时间片，不同运行程序拥有不同的时间片，由于时间片划分非常短，以致我们觉察不出来程序运行有任何中断，这种设计是并发。通过设计出虚拟内存的机制，使每个进程仿佛独立拥有自己的内存，但是这样的导致不同进程之间进行无法共享变量。比如在使用文档编辑器的时候，你希望程序（比如 `MS Word`)既可以接受你的键盘的输入，也可以对你的输入进行单词检查，这样就设计出更线程(`thread`)，每个进程包含了大量的线程，每个线程拥有自己的线程栈，但是可以共享进程的堆栈( `heap` )，达到不同线程之间通信的功能，而CPU时间片调度也转换为线程为单位。
通过并发，我们实现了计算机多任务处理的能力，而且对于计算密集的任务，多线程设计可以争取更多的CPU计算时间片，加快任务运行速度。对于 `Web` 服务器，每一个请求都需要创建新的线程来响应请求。

## 2 什么是竞争条件？举一个例子来描述？

竞争条件发生在多个线程访问同一个共享数据的时候，它们都尝试去修改这个数据。由于线程调度算法可以在任何时候修改它们的执行的顺序，所以你不会知道哪一个线程有优先执行，这就会导致最终的结果由线程调度算法决定。
问题往往发生在对共享数据执行`Read-Modify-Write`的操作，由于`CPU`执行的粒度比代码执行的程序更小，如果在执行`Write`之前，线程被调度算法切换出去了，那么就会导致最终的结果和预期不一致。

```go
if x == 0 {
    x = x + 1
}
```

执行的过程如何首先读取 `x` 的值是否为`0`，如果满足条件，则修改它加1，然后在写回到内存那种。中间的过程通常借助寄存器完成计算，如果在执行中间过程中，线程被切换出去，那么导致另外的线程取得的`x`值任然为内存，导致最终的结果不一致。

## 3 什么是死锁？编写一些代码是由死锁导致的；

由于并发执行线程或者进程设计不合理，导致它们互相等待对象释放资源，从而导致程序无法进行下去。假设线程 `A` 锁住了资源 `x`，并且等待资源 `y`
的释放；然而线程 `B` 目前锁住了资源 `y`, 并且等待资源 `x` 的释放。这样带来所有程序无法继续往下进行，导致死锁 (`dead lock`)

```go
var x int
var y int
var xlocker sync.Mutex
var ylocker sync.Mutex

func main(){
    var wg sync.WaitGroup
    wg.Add(2)
    go func(){
        xlocker.Lock()
        x++
        // do something for x
        xlocker.Unlock()

        ylocker.Lock()
        y++
        // do something for y
        ylocker.Unlock()
        wg.Done()
    }()

    go func(){
        ylocker.Lock()
        // do something for y
        y++
        ylocker.Unlock()

        xlocker.Lock()
        // do something for x
        x++
        xlocker.Unlock()
        wg.Done()
    }
    wg.Wait()
}
```

解决死锁的方法有：

- 对资源的进行排序，每次获取锁按照训练进行；
- 增大锁的粒度，每次锁住全部所需的全部资源。

## 4 什么是进程饥饿？如果需要，重新回顾一下定义

在一个动态系统中，资源请求与释放是经常性发生的进程行为．对于每类系统资源，操作系统需要确定一个分配策略，当多个进程同时申请某类资源时，由分配策略确定资源分配给进程的次序。 资源分配策略可能是公平的(fair)，能保证请求者在有限的时间内获得所需资源；资源分配策略也可能是不公平的(unfair)，即不能保证等待时间上界的存在。 在后一种情况下，即使系统没有发生死锁，某些进程也可能会长时间等待．当等待时间给进程推进和响应带来明显影响时，称发生了进程饥饿(starvation)，当饥饿到一定程度的进程所赋予的任务即使完成也不再具有实际意义时称该进程被饿死(starve to death)。

## 6 什么是`Wait Free`算法？

在并发编程中，对共享数据的读写保护非常重要，通常的手段就是同步。同步分为两种

- 阻塞型同步(`Blocking Synchronization`)
- 非阻塞型同步(`Non-blocking Synchronization`)

常见的阻塞同步的方法当某个线程到达某个临界区间，如果另外一个线程持有访问该共享数据的锁，那么这个线程将会阻塞，直到另一个线程释放该锁。常见的同步原语有 `mutex`, `semaphore`。但是这种方法存在`deadlock`, `livelock` 和 优先级反转 (`priority inversion`)等问题。

比较流行的 `Non-blocking Synchronization`方法有

- Wait Free
`Wait-free` 是指任意线程的任何操作都可以在有限步之内结束，而不用关心其它线程的执行速度。 `Wait-free` 是基于 `per-thread` 的，可以认为是 `starvation-free` 的。非常遗憾的是实际情况并非如此，采用 `Wait-free` 的程序并不能保证 `starvation-free`，同时内存消耗也随线程数量而线性增长。目前只有极少数的非阻塞算法实现了这一点。

- Lock-free
`Lock-Free` 是指能够确保执行它的所有线程中至少有一个能够继续往下执行。由于每个线程不是 `starvation-free` 的，即有些线程可能会被任意地延迟，然而在每一步都至少有一个线程能够往下执行，因此系统作为一个整体是在持续执行的，可以认为是 `system-wide` 的。所有 `Wait-free` 的算法都是 `Lock-Free` 的。

- Obstruction-free

Obstruction-free 是指在任何时间点，一个孤立运行线程的每一个操作可以在有限步之内结束。只要没有竞争，线程就可以持续运行。一旦共享数据被修改，Obstruction-free 要求中止已经完成的部分操作，并进行回滚。 所有 Lock-Free 的算法都是 Obstruction-free 的。

一般采用原子级的 read-modify-write 原语来实现 Lock-Free 算法，其中 LL 和 SC 是 Lock-Free 理论研究领域的理想原语，但实现这些原语需要 CPU 指令的支持，非常遗憾的是目前没有任何 CPU 直接实现了 SC 原语。根据此理论，业界在原子操作的基础上提出了著名的 CAS（Compare - And - Swap）操作来实现 Lock-Free 算法，Intel 实现了一条类似该操作的指令：cmpxchg8。

## 9 为什么测试多线程或者并发比较困难？
*todo*
