# Nodejs 消息队列

Nodejs 和浏览器中的消息循环可能不一样. 这里我们只讨论 Nodejs  Event Loop 的原理.

## Nodejs 总体架构



## Nodejs 线程模型

![](http://voidcanvas.com/wp-content/uploads/2018/02/nodejs-event-loop-workflow.png)

Nodejs 完全是单线程的. 从进程启动后, 由主线程加载我们的 js 文件(上图中 main.js), 然后进入消息循环. 可见对于 js 程序而言, 完整运行在单线程之中.

但并不是说 Node 进程只有一个线程. 正如 [Node.js event loop workflow & lifecycle in low level](http://voidcanvas.com/nodejs-event-loop/) 中所说:

> the fact is thread-pool is something in libUV library (used by node for third party asynchronous handling)
> ...
> However, ... like, file reading, making request to a different host, dns lookup etc., are handled by the thread-pool, which uses only 4 threads by default...

相关的源码在这里: [libuv/src/threadpool.c](https://github.com/libuv/libuv/blob/c73e73c8743cc664460a8ae173f034481d94323f/src/threadpool.c#L38)

在 libUV 这一层实际上是有个线程池辅助完成一些工作的. 

## 细说消息循环

再来看一下 JS 中的消息循环部分:

![](http://voidcanvas.com/wp-content/uploads/2018/02/nodejs-event-loop-phase.png)

Nodejs 将消息循环又细分为 6 个阶段(官方叫做 Phrase), 每个阶段都会有一个类似于队列的结构, 存储着该阶段需要处理的回调函数. 我们来看一下这 6 个 Phrase 的作用:

### Timer Phrase

这是消息循环的第一个阶段, 处理所有

我们来举个例子直观的感受一下.

这是一道经典的 FE 面试题: 

> 请问如下代码的输出:
> ```js
> // index.js
>
> setImmediate(() => console.log(2))
> setTimeout(() => console.log(1))
> 
> sleep(10)    // 休眠 10ms, 假设由他人提供
> ```
> 答案: 1 2

我们从原理的角度看看这道消息循环的基础问题.

1. 首先, Nodejs 启动, 初始化环境后加载我们的 JS 代码(index.js)
    
    执行 JS 代码是 v8 的工作. 它执行 setImmediate 后, 将回调参数放入了 Check Phrase 所在的队列. 


## Refs

[Node 定时器详解](http://www.ruanyifeng.com/blog/2018/02/node-event-loop.html)
[Node.js event loop workflow & lifecycle in low level](http://voidcanvas.com/nodejs-event-loop/)
[The Node.js Event Loop, Timers, and process.nextTick()](https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/)
[不同的event loop](https://cnodejs.org/topic/5a9108d78d6e16e56bb80882)
[libuv 源码](https://github.com/libuv/libuv/tree/master)