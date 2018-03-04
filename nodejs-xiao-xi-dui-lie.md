# Nodejs 消息队列

Nodejs 和浏览器中的消息循环可能不一样. 这里我们只讨论 Nodejs Event Loop 的原理.

源码参照的当下最新的 Nodejs v9.7.1, 对应 libUV 是 v1.x.

## Nodejs 总体架构



## Nodejs 线程模型

![](http://voidcanvas.com/wp-content/uploads/2018/02/nodejs-event-loop-workflow.png)

Nodejs 完全是单线程的. 从进程启动后, 由主线程加载我们的 js 文件(上图中 main.js), 然后进入消息循环. 可见对于 js 程序而言, 完整运行在单线程之中.

但并不是说 Node 进程只有一个线程. 正如 [Node.js event loop workflow & lifecycle in low level](http://voidcanvas.com/nodejs-event-loop/) 中所说:

> the fact is thread-pool is something in libUV library (used by node for third party asynchronous handling)
> ...
> However, ... like, file reading, making request to a different host, dns lookup etc., are handled by the thread-pool, which uses only 4 threads by default...

相关的源码在这里: [libuv/src/threadpool.c#38](https://github.com/libuv/libuv/blob/c73e73c8743cc664460a8ae173f034481d94323f/src/threadpool.c#L38)

在 libUV 这一层实际上是有个线程池辅助完成一些工作的.

## 细说消息循环

再来看一下 JS 中的消息循环部分:

![](http://voidcanvas.com/wp-content/uploads/2018/02/nodejs-event-loop-phase.png)

Nodejs 将消息循环又细分为 6 个阶段(官方叫做 Phase), 每个阶段都会有一个类似于队列的结构, 存储着该阶段需要处理的回调函数. 我们来看一下这 6 个 Phase 的作用:

### Timer Phase

这是消息循环的第一个阶段, 用一个 for 循环处理所有 `setTimeout` 和 `setInterval` 的回调. 代码在这里: [src/unix/timer.c#150](https://github.com/libuv/libuv/blob/9ed3ed5fcb3f19eccd3d29848ae2ff0cfd577de9/src/unix/timer.c#L150)

这些回调被保存在一个最小堆(min heap) 中. 这样引擎只需要每次判断头元素, 如果符合条件就拿出来执行, 直到遇到一个不符合条件或者队列空了, 才结束 Timer Phase.

Timer Phase 中判断某个回调是否符合条件的方法也很简单. 消息循环每次进入 Timer Phase 的时候都会保存一下当时的系统时间, 然后只要看上述最小堆中的回调函数设置的启动时间是否超过进入 Timer Phase 时保存的时间, 如果超过就拿出来执行.

此外, Nodejs 为了防止某个 Phase 任务太多, 导致后续的 Phase 发生饥饿的现象, 所以消息循环的每一个迭代(iterate) 中, 每个 Phase 执行回调都有个最大数量. 如果超过数量的话也会强行结束当前 Phase 而进入下一个 Phase. 这一条规则适用于消息循环中的每一个 Phase, 后面也就不再单独说了.

### Pending I/O Callback Phase

这一阶段是执行你的 `fs.read`, socket 等 IO 操作的回调函数, 同时也包括各种 error 的回调. 源码在这里: [src/unix/core.c#763](https://github.com/libuv/libuv/blob/9ed3ed5fcb3f19eccd3d29848ae2ff0cfd577de9/src/unix/core.c#L763)

### Idle, Prepare Phase

据说是内部使用, 所以我们也不在这里过多讨论.

### Poll Phase

这是整个消息循环中最重要的一个 Phase, 作用是等待异步请求和数据(原文: _accepts new incoming connections (new socket establishment etc) and data (file read etc)_). 

说它最重要是因为它支撑了整个消息循环机制. Poll Phase 首先会执行 `watch_queue` 队列中的 IO 请求, 一旦 `watch_queue` 队列空, 则整个消息循环就会进入 sleep (在不同的平台上使用不同的技术. 比如 Linux 下使用 epoll, Win 下则是 IOCP, OS X 是 kqueue), 从而等待被内核事件唤醒. 源码在这里: [src/unix/linux-core.c#188](https://github.com/libuv/libuv/blob/9ed3ed5fcb3f19eccd3d29848ae2ff0cfd577de9/src/unix/linux-core.c#L188)

当然 Poll Phase 不能一直等下去. 它有着精妙的设计. 简单来说, 

    1. 它首先会判断后面的 Check Phase 以及 Close Phase 是否还有等待处理的回调. 如果有, 则不等待, 直接进入下一个 Phase. 
    2. 如果没有其他回调等待执行, 它会给 epoll 这样的方法设置一个 timeout. 可以猜一下, 这个 timeout 设置为多少合适呢? 答案就是 Timer Phase 中最近要执行的回调启动时间到现在的差值, 假设这个差值是 detal. 因为 Poll Phase 后面没有等待执行的回调了. 所以这里最多等待 delta 时长, 如果期间有事件唤醒了消息循环, 那么就继续下一个 Phase 的工作; 如果期间什么都没发生, 那么到了 timeout 后, 消息循环依然要进入后面的 Phase, 让下一个迭代的 Timer Phase 也能够得到执行.

Nodejs 就是通过 Poll Phase, 对 IO 事件的等待和内核异步事件的到达来驱动整个消息循环的.

### Check Phase

接下来是 Check Phase. 这个阶段只处理 `setImmediate` 的回调函数. 

那么为什么这里要有专门一个处理 `setImmediate` 的 Phase 呢? 简单来说, 是因为 Poll Phase 阶段可能设置一些回调, 希望在 Poll Phase 后运行. 所以在 Poll Phase 后面增加了这个 Check Phase. 

### Close Callbacks Phase

专门处理一些 close 类型的回调. 比如 `socket.on('close', ...)`. 用于资源清理.


## FAQ

### `setTimeout(..., 0)` vs. `setImmediate` 到底谁快?

我们来举个例子直观的感受一下.

这是一道经典的 FE 面试题:

> 请问如下代码的输出:
> ```js
> // index.js
>
> setImmediate(() => console.log(2))
> setTimeout(() => console.log(1))
>
> sleep(10) // 休眠 10ms, 假设由他人提供
> ```
> 答案: 1 2

我们从原理的角度看看这道消息循环的基础问题.

1. 首先, Nodejs 启动, 初始化环境后加载我们的 JS 代码(index.js)
执行 JS 代码是 v8 的工作. 它执行 setImmediate 后, 将回调参数放入了 Check Phase 所在的队列.

### `setTimeout(..., 0)` 是否可以代替 `setImmediate` 呢?

从性能角度讲, `setTimeout` 的处理是在 Timer Phase, 其中 min heap 保存了 timer 的回调, 因此每执行一个回调的同时都会涉及到堆调整. 而 `setImmediate` 仅仅是清空一个队列. 效率自然会高很多. 

再从执行时机上讲. `setTimeout(..., 0)` 和 `setImmediate` 完全属于两个 Phase. 请参考 _'`setTimeout(..., 0)` vs. `setImmediate` 到底谁快?'_ 中的讨论.


## Refs

[Node 定时器详解](http://www.ruanyifeng.com/blog/2018/02/node-event-loop.html)
[Node.js event loop workflow & lifecycle in low level](http://voidcanvas.com/nodejs-event-loop/)
[The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
[不同的event loop](https://cnodejs.org/topic/5a9108d78d6e16e56bb80882)
[libuv 源码](https://github.com/libuv/libuv/tree/master)