这是《Java 并发与多线程：从入门到入魔》系列的开篇之作。

在之前的“容器系列”中，我们探讨了数据的存储。现在，我们要让这些数据“动”起来。
线程池（ThreadPool）是 Java 并发编程的基础设施。你可能背过它的七个参数，但你是否真正理解这七个参数在源码层面是如何互动的？为什么阿里巴巴开发手册严禁使用 `Executors` 去创建线程池？`ctl` 变量是如何用一个 `int` 同时存下线程状态和数量的？

这一篇，我们将从源码的位运算开始，彻底拆解 `ThreadPoolExecutor`。

-----

## layout: post title:  "Java 并发与多线程（一）：线程池的七个参数与拒绝策略的血泪史" date:   2025-11-30 10:00:00 +0800 categories: [Java, 并发编程, 源码分析] tags: [线程池, ThreadPoolExecutor, OOM, 拒绝策略, 位运算]

## 系列前言

欢迎来到\*\*《Java 并发与多线程：从入门到入魔》\*\*系列。
如果说数据结构是程序的“肉体”，那线程就是程序的“灵魂”。在这个系列中，我们将深入 JVM 的并发机制，从线程池、锁机制（AQS）、JMM 内存模型，一直讲到无锁并发（Disruptor）和协程（Loom）。

## 第一章：不要重复造轮子——为什么需要线程池？

在 Java 中，创建一个线程 (`new Thread()`) 是一个**昂贵**的操作。

1.  **JVM 层面**：需要分配栈内存（默认 1MB）。
2.  **OS 层面**：需要调用操作系统内核 API (`pthread_create`)，涉及到用户态到内核态的切换。

如果你来一个请求就 `new` 一个线程，处理完就销毁，高并发下 CPU 会把大量时间浪费在“创建”和“销毁”上，而不是真正的业务逻辑。

**线程池的核心思想**：**复用**。
就像共享单车一样，用完不扔，还回去给下一个人用。

## 第二章：构造函数的七个参数 (The Magnificent Seven)

打开 `ThreadPoolExecutor` 的源码，最全的构造函数包含 7 个参数。这是面试必考题，但能讲清楚逻辑的人很少。

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler)
```

我们用一个\*\*“银行网点”\*\*的例子来类比：

1.  **`corePoolSize` (核心线程数)**：

      * **银行的常驻柜台**。哪怕没人办业务，这些窗口也开着，工作人员坐在那里待命。
      * *注：除非设置了 `allowCoreThreadTimeOut`，否则核心线程即使空闲也不会被回收。*

2.  **`workQueue` (任务队列)**：

      * **银行的等候区 (椅子)**。当常驻柜台满了，新来的客户（任务）就先坐到椅子上排队。
      * *关键点：队列的类型（有界/无界）直接决定了线程池的抗压能力。*

3.  **`maximumPoolSize` (最大线程数)**：

      * **银行的最大窗口数**（常驻 + 临时）。
      * 当**等候区坐满了**，大堂经理就会把暂时关闭的临时窗口打开，调更多人来加班。

4.  **`keepAliveTime` & `unit` (存活时间)**：

      * **临时工的解雇时间**。
      * 如果临时窗口空闲了超过这个时间，临时工就会被解雇（线程销毁），直到线程数恢复到 `corePoolSize`。

5.  **`threadFactory` (线程工厂)**：

      * **工牌打印机**。用于创建线程时给线程起名字（如 `order-pool-thread-1`）。
      * *强烈建议自定义，方便出问题时看日志排查。*

6.  **`handler` (拒绝策略)**：

      * **保镖**。当窗口全开、椅子全满，还有人要挤进来时，保镖会出面把人“扔出去”。

-----

## 第三章：源码级的执行流程

当调用 `execute(Runnable command)` 时，流程如下（一定要背下来）：

1.  **Check Core**: 如果当前运行的线程数 \< `corePoolSize`，**立即创建新线程**运行这个任务（即使有其他核心线程在空闲，也会创建新的，为了尽快预热）。
2.  **Check Queue**: 如果核心线程满了，尝试把任务**放入 `workQueue`**。
3.  **Check Max**: 如果队列也满了（offer 失败），判断当前线程数是否 \< `maximumPoolSize`。
      * 如果是，**创建非核心线程**运行这个任务。
4.  **Reject**: 如果非核心线程也满了，调用 `handler.rejectedExecution` 触发拒绝策略。

> **反直觉陷阱**：
> 很多人以为流程是：核心满 -\> 开临时 -\> 临时满 -\> 进队列。
> **错！** 实际上是：**核心满 -\> 进队列 -\> 队列满 -\> 开临时**。
> 这是一个“懒惰”的策略，尽量不创建新线程，除非队列实在装不下了。

-----

## 第四章：位运算的魔法——`ctl` 变量

在 `ThreadPoolExecutor` 中，有一个极其牛逼的变量：

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
```

Doug Lea 大神为了节省空间，用**一个 int (32位)** 同时存储了两个状态：

1.  **runState (线程池状态)**：占高 3 位。
2.  **workerCount (工作线程数量)**：占低 29 位。

**状态定义**：

  * **RUNNING**:  `111...` (负数, -536870912) -\> 接受新任务，处理队列。
  * **SHUTDOWN**: `000...` -\> 不接新任务，但处理队列里剩余的。
  * **STOP**:     `001...` -\> 不接新，不处理队列，中断正在运行的。
  * **TIDYING**:  `010...` -\> 所有任务已终止，准备调用 `terminated()`。
  * **TERMINATED**: `011...` -\> 彻底死透。

**源码片段**：

```java
// 获取运行状态 (通过位与运算屏蔽低 29 位)
private static int runStateOf(int c)     { return c & ~CAPACITY; }

// 获取线程数量 (通过位与运算屏蔽高 3 位)
private static int workerCountOf(int c)  { return c & CAPACITY; }
```

这种设计使得对“状态”和“数量”的修改可以变成一次 CAS 原子操作，避免了维护两个原子变量带来的并发一致性问题。

-----

## 第五章：生产事故之源——Executors

面试必问：**为什么阿里巴巴 Java 开发手册禁止使用 `Executors` 创建线程池？**

### 5.1 `newFixedThreadPool` 的坑

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>()); // ！！！
}
```

  * **问题**：它使用了 `LinkedBlockingQueue`，且没有指定容量。
  * **默认容量**：`Integer.MAX_VALUE` (约 21 亿)。
  * **后果**：如果任务处理不过来，队列会无限堆积，直到耗尽堆内存，导致 **OOM (Out Of Memory)**。

### 5.2 `newCachedThreadPool` 的坑

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, // ！！！
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

  * **问题**：最大线程数是 `Integer.MAX_VALUE`。
  * **后果**：如果你瞬间并发 10 万个请求，它会瞬间创建 10 万个线程（因为 `SynchronousQueue` 存不下任务）。这会直接把 CPU 或者是操作系统（文件句柄/线程数限制）打爆。

**正确姿势**：
永远**手动 `new ThreadPoolExecutor(...)`**，并显式指定有界队列的长度（如 `ArrayBlockingQueue(1000)`）。

-----

## 第六章：拒绝策略——最后的底线

当队列满且线程满时，必须拒绝。JDK 提供了 4 种内置策略：

1.  **AbortPolicy (默认)**：

      * 直接抛出 `RejectedExecutionException` 异常。
      * *评价*：简单粗暴，调用者必须捕获异常，否则业务崩溃。

2.  **CallerRunsPolicy (调用者运行)**：

      * 谁提交的任务，谁自己去执行（比如主线程）。
      * *评价*：这是个**自带“背压” (Backpressure)** 的策略。如果主线程被迫去执行任务，它就没空提交新任务了，从而降低了提交速度，给线程池喘息的机会。**推荐用于不能丢数据的关键业务。**

3.  **DiscardPolicy**：

      * 默默丢弃，不抛异常。
      * *评价*：极其危险，除非是日志收集这种无关紧要的任务。

4.  **DiscardOldestPolicy**：

      * 丢弃队列里**最老**的一个任务，尝试把新任务塞进去。
      * *评价*：这是在“喜新厌旧”，但在某些场景（如最新的传感器数据比老数据重要）下有用。

-----

## 总结与预告

线程池是 Java 并发编程的基石，但也充满了陷阱。

1.  **参数配合**：核心 -\> 队列 -\> 最大 -\> 拒绝。
2.  **禁止 Executors**：避免无界队列导致的 OOM。
3.  **拒绝策略**：善用 `CallerRunsPolicy` 进行流控。

但是，线程池解决了“线程复用”的问题，却没有解决“多线程修改共享变量”带来的安全问题。当多个线程同时修改 `count++` 时，为什么结果总是不对？`volatile` 关键字到底保证了什么？

**下一篇预告**：
我们将深入 CPU 硬件层面，探讨 **Java 内存模型 (JMM)**。为什么会有可见性问题？CPU 缓存一致性协议 (MESI) 是什么？敬请期待《Java 并发与多线程（二）：JMM 内存模型与 volatile 的底层原理》。
