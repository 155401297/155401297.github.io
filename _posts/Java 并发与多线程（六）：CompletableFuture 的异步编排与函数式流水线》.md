这是《Java 并发与多线程：从入门到入魔》系列的第六篇。

在之前的章节中，我们讨论的线程协作大多是“同步”的：线程 A 等待线程 B 释放锁，或者等待 CountDownLatch 倒数归零。但在微服务和高吞吐量的系统中，我们更追求**异步**：线程 A 发起请求后不应该傻等，而是应该去干别的事，等结果回来了自动触发回调。

JDK 5 的 `Future` 做了一半（能异步执行，但获取结果必须阻塞），而 JDK 8 的 `CompletableFuture` 真正完成了拼图。它借鉴了函数式编程的思想，让我们可以像搭积木一样编排复杂的异步流水线。

今天，我们将深入剖析它的“驱动栈”原理，以及为什么你在生产环境绝不能直接裸用它。

-----

## layout: post title:  "Java 并发与多线程（六）：CompletableFuture 的异步编排与函数式流水线" date:   2025-12-05 10:00:00 +0800 categories: [Java, 并发编程, 异步编程] tags: [CompletableFuture, 异步回调, 函数式编程, ForkJoinPool, 编排]

## 系列前言

这是《Java 并发与多线程：从入门到入魔》系列的第六篇。如果说线程池是工厂里的工人，那么 `CompletableFuture` 就是自动化流水线的传送带。它允许我们将多个异步任务串行、并行、组合，而无需编写复杂的回调嵌套（Callback Hell）。

## 第一章：`Future` 的遗憾与 `CompletableFuture` 的诞生

在 JDK 5 时代，`Future` 是异步的标准。

```java
Future<String> future = executor.submit(() -> doSomething());
// ... 干点别的 ...
String result = future.get(); // 阻塞！这里会卡住主线程
```

**痛点**：

1.  **阻塞获取**：调用 `get()` 会导致当前线程挂起，违背了异步的初衷。
2.  **无法编排**：如果你想“等 A 任务做完，把结果传给 B 任务做，同时 C 任务也在做，最后把 B 和 C 的结果汇总”，用 `Future` 写出来的代码会全是 `if-else` 和嵌套，极其丑陋。

JDK 8 引入的 `CompletableFuture` (CF) 实现了 `CompletionStage` 接口，它代表一个“阶段”。
核心思想是：**不要主动去 get()，而是注册一个回调，告诉它“做完了之后该干嘛”**。

-----

## 第二章：从零开始构建流水线

CF 的 API 多达 50 几个，初学者容易晕。其实只需要记住三个核心维度：**创建、转换、组合**。

### 2.1 创建异步任务 (Zero Dependency)

通常我们使用静态工厂方法：

```java
// 1. 无返回值
CompletableFuture<Void> runFuture = CompletableFuture.runAsync(() -> {
    System.out.println("Running asynchronously");
});

// 2. 有返回值 (最常用)
CompletableFuture<String> supplyFuture = CompletableFuture.supplyAsync(() -> {
    return "Hello";
});
```

### 2.2 任务的流式转换 (One Dependency)

当上一个任务完成时，自动触发下一个。这类似于 Stream 流的 `map`。

```java
CompletableFuture.supplyAsync(() -> "Hello")
    .thenApply(s -> s + " World")       // 1. 接收上一部的结果，转换
    .thenAccept(s -> System.out.println(s)) // 2. 消费结果，无返回值
    .thenRun(() -> System.out.println("Done")); // 3. 不关心结果，只运行代码
```

  * **`thenApply`**: 有入参，有返回值 (Function)。
  * **`thenAccept`**: 有入参，无返回值 (Consumer)。
  * **`thenRun`**: 无入参，无返回值 (Runnable)。

### 2.3 任务的组合 (Two Dependencies)

这是 CF 最强大的地方：处理两个异步任务的关系。

#### AND 关系 (Both)

“等 A **和** B 都做完，再做 C。”

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> 10);
CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> 20);

// 合并结果
CompletableFuture<Integer> result = future1.thenCombine(future2, (r1, r2) -> {
    return r1 + r2; // 30
});
```

#### OR 关系 (Either)

“A **或** B 只要有一个做完，就立刻做 C。”（通常用于高可用备份，谁快用谁）。

```java
future1.acceptEither(future2, result -> {
    System.out.println("Fastest result: " + result);
});
```

-----

## 第三章：底层原理 —— 驱动栈 (Completion Stack)

为什么 CF 可以一环扣一环地触发？

**内部结构**：
每个 `CompletableFuture` 对象内部持有一个 `result` 字段和一个 `stack`（堆栈）。

```java
volatile Object result;       // 存储结果或异常
volatile Completion stack;    // 依赖此任务的后续任务链表
```

**工作流程**：

1.  当你调用 `futureA.thenApply(func)` 时，CF 会生成一个新的 `futureB`。
2.  它会将这个操作封装成一个 `Completion` 对象，**压入** `futureA` 的 `stack` 中。
3.  当 `futureA` 执行完毕（调用 `completeValue`），它会检查 `stack`。
4.  它发现栈里有个任务（计算 B），于是**触发**这个任务执行。如果指定了线程池，就扔给线程池；否则可能由当前线程继续执行。

这是一种标准的**观察者模式**，结合了**栈**结构来实现回调链的深度优先触发。

-----

## 第四章：异常处理的艺术

在同步代码中我们用 `try-catch`。在异步流水线中，异常会**顺着链路向下传递**。

```java
CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("Boom!");
}).thenApply(s -> {
    return s + " World"; // 不会执行，直接跳过
}).exceptionally(ex -> {
    System.out.println("Error: " + ex.getMessage()); // 捕获异常
    return "Default Value"; // 异常恢复，返回兜底值
});
```

  * **`exceptionally`**: 相当于 `catch`。
  * **`handle`**: 相当于 `finally`，无论成功失败都会执行，能拿到 result 和 exception。

-----

## 第五章：生产环境的隐形杀手 —— 线程池

这是使用 CF 最容易踩的坑。

### 5.1 默认线程池的陷阱

如果你在使用 `supplyAsync` 时**不指定线程池**：

```java
CompletableFuture.supplyAsync(() -> { ... });
```

它默认使用的是 **`ForkJoinPool.commonPool()`**。

  * **问题**：`ForkJoinPool` 的核心线程数默认为 **CPU 核心数 - 1**。
  * **后果**：如果你的任务是 **IO 密集型**（比如查数据库、调 HTTP），线程会长时间阻塞。只要几个慢请求，就能把整个系统的 CommonPool 占满。导致系统内其他所有依赖 CommonPool 的异步任务（包括 Stream 并行流）全部卡死。

### 5.2 正确姿势

**永远显式指定自定义的线程池！**

```java
// 自定义 IO 密集型线程池
ExecutorService ioPool = new ThreadPoolExecutor(
    10, 20, 60, TimeUnit.SECONDS, 
    new ArrayBlockingQueue<>(1000), 
    new ThreadFactoryBuilder().setNameFormat("io-pool-%d").build()
);

CompletableFuture.supplyAsync(() -> {
    return queryDb();
}, ioPool); // <--- 传入 executor
```

这样实现了**资源隔离**：数据库慢了，只会把 `ioPool` 打满，不会拖垮负责计算的 CPU 线程池或系统的公共池。

-----

## 总结与预告

`CompletableFuture` 是 Java 异步编程的里程碑：

1.  **非阻塞**：彻底告别 `future.get()`。
2.  **编排能力**：支持 AND、OR、串行等多种组合逻辑。
3.  **注意点**：务必使用自定义线程池隔离 IO 任务；熟练使用 `exceptionally` 处理异常链。

至此，我们已经掌握了线程池、锁、ThreadLocal 和异步编排。但在某些极高性能的场景下（如秒杀系统、高频交易），即使是线程池和锁也显得太慢了。我们需要一种**无锁、无垃圾回收、缓存行填充**的极致队列。

**下一篇预告**：
我们将探索号称“单机最快队列”的架构设计。它是 Log4j2 异步日志高性能的秘密，也是 LMAX 交易所的核心。敬请期待《Java 并发与多线程（七）：Disruptor 的环形缓冲区与伪共享的黑魔法》。
