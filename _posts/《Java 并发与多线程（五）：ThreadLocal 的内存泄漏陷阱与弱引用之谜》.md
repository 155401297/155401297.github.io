这是《Java 并发与多线程：从入门到入魔》系列的第五篇。

在并发编程的世界里，我们要么通过锁（Synchronized/Lock）来控制对共享变量的访问，要么通过 CAS（Atomic\*）来原子性地更新变量。但还有一种反其道而行之的思路：**如果数据完全不共享，是不是就不需要同步了？**

这就是 **ThreadLocal** 的核心思想。它是 Java 中最隐秘的角落，被广泛用于 Spring 的事务管理和 Web 框架的 Session 管理。但它也是一把双刃剑：用好了是神器，用不好就是 **内存泄漏 (Memory Leak)** 的剧毒来源。

今天，我们将剖析 ThreadLocal 的内部解剖图，解释为什么它的 Key 必须是**弱引用**，以及为什么在 Tomcat 线程池中使用它时要格外小心。

-----

## layout: post title:  "Java 并发与多线程（五）：ThreadLocal 的内存泄漏陷阱与弱引用之谜" date:   2025-12-04 10:00:00 +0800 categories: [Java, 并发编程, 源码分析] tags: [ThreadLocal, 内存泄漏, 弱引用, ThreadLocalMap, 开放寻址法]

## 系列前言

这是《Java 并发与多线程：从入门到入魔》系列的第五篇。我们常说“线程安全”，通常是指多线程访问同一个对象。而 `ThreadLocal` 提供了一种**线程封闭 (Thread Confinement)** 的机制，将变量绑定到当前线程上，实现“人手一份，互不干扰”。

## 第一章：颠覆认知的存储结构

很多初学者（甚至老手）认为 `ThreadLocal` 内部维护了一个 `ConcurrentHashMap<Thread, Object>`，把线程作为 Key，值作为 Value。

**大错特错！**

JDK 的设计恰恰相反：**数据是放在 Thread 对象里面的**，`ThreadLocal` 仅仅是一个**访问入口（Key）**。

打开 `Thread` 类的源码：

```java
public class Thread implements Runnable {
    // ...
    // 每个线程自己维护了一个 Map
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```

当你调用 `threadLocal.set("value")` 时，实际发生的是：

```java
// ThreadLocal.java
public void set(T value) {
    Thread t = Thread.currentThread(); // 1. 获取当前线程
    ThreadLocalMap map = getMap(t);    // 2. 拿出线程肚子里的 Map
    if (map != null)
        map.set(this, value);          // 3. 以【ThreadLocal实例自己】为 Key 存入
    else
        createMap(t, value);
}
```

**结论**：数据存储在 `Thread` 对象的堆内存中。这就解释了为什么不同线程取不到对方的数据——因为数据压根就不在一个对象里。

-----

## 第二章：ThreadLocalMap —— 被阉割的 HashMap

`Thread` 中持有的这个 `ThreadLocalMap` 是一个定制化的哈希表。它没有实现 `Map` 接口，在解决哈希冲突时，它没有使用 `HashMap` 的**链地址法**（拉链法），而是使用了**开放寻址法 (Open Addressing)** 中的**线性探测 (Linear Probing)**。

### 2.1 为什么用线性探测？

  * **链地址法**：冲突了就挂个链表（或红黑树）。
  * **线性探测**：冲突了就往后找下一个空位（Slot）。

因为通常一个线程持有的 ThreadLocal 变量不会太多，哈希冲突概率低。线性探测能节省 Entry 对象的内存开销（不用存 `next` 指针），且对 CPU 缓存更友好（数组连续内存）。

-----

## 第三章：弱引用的玄机 (The Mystery of WeakReference)

这是面试中最硬核的部分。`ThreadLocalMap` 的 Entry 定义如下：

```java
// Entry 继承了 WeakReference，Key 是弱引用！
static class Entry extends WeakReference<ThreadLocal<?>> {
    Object value; // Value 是强引用

    Entry(ThreadLocal<?> k, Object v) {
        super(k); // Key 被传递给弱引用构造器
        value = v;
    }
}
```

### 3.1 什么是弱引用？

  * **强引用 (Strong)**：`Object o = new Object()`。只要引用还在，GC 永远不会回收。
  * **弱引用 (Weak)**：只要发生 GC，不管内存够不够，**一定会被回收**。

### 3.2 为什么要设计成弱引用？

**为了防止 ThreadLocal 对象本身的内存泄漏。**

试想：如果 Key 是强引用。
当你的业务代码 `tl = null`（试图丢弃这个 ThreadLocal 对象）时，由于 `ThreadLocalMap`（在线程里）还强引用着它，它永远无法被 GC。
除非线程结束，否则这个 `ThreadLocal` 对象会一直“僵尸”在内存里。

**变成弱引用后**：
当你 `tl = null`，下一次 GC 时，`ThreadLocalMap` 里的 Key 就会被回收（变成 null）。

-----

## 第四章：真正的杀手 —— Value 的内存泄漏

虽然 Key 被回收了，但**真正的噩梦**才刚刚开始。

### 4.1 引用链分析

当 Key 被 GC 回收后，Entry 变成了 `null -> Value`。
此时的引用链是：
`Current Thread` (强) -\> `ThreadLocalMap` (强) -\> `Entry` (强) -\> `Value` (强)。

**关键点**：只要线程不透（比如使用了**线程池**），这个线程就会一直活着。那么 `Value` 对象（可能是个很大的 User 对象）就永远有一根强引用链牵着它，**无法被 GC**。

这就是 **ThreadLocal 内存泄漏**的根源：**Key 没了，但 Value 还在，且无法访问（因为 Key 丢了），也没人能清理它。**

### 4.2 JDK 的补救措施

JDK 的作者 Doug Lea 早就预判了这一点。他在 `get()`, `set()`, `remove()` 方法中埋下了“排雷”逻辑：

当你调用这些方法时，如果遇到 Key 为 `null` 的 Entry（被称为 **Stale Entry / 脏条目**），它会顺手把对应的 Value 也置为 `null`，从而断开引用链，帮助 GC。

**但是**：这是一种\*\*“懒清理”\*\*。如果你在用完后，**不再调用** `get/set`，或者长时间不调用，这些脏 Value 就会一直堆积，最终导致 **OOM (Out Of Memory)**。

-----

## 第五章：最佳实践 —— 谁污染，谁治理

在工程实践中（特别是配合线程池使用时），必须遵守\*\*“先 set，后 try，finally remove”\*\*的范式：

```java
// 1. 定义
private static final ThreadLocal<User> currentUser = new ThreadLocal<>();

// 2. 使用
try {
    currentUser.set(user);
    // 执行业务逻辑...
} finally {
    // 3. 必须清理！(防止内存泄漏 + 线程池复用导致的数据串读)
    currentUser.remove(); 
}
```

**为什么 `remove` 如此重要？**

1.  **防泄漏**：主动断开 Value 的强引用，不仅依靠 GC 的被动清理。
2.  **防串台**：线程池中的线程是复用的。如果线程 A 用完没清理，线程 B 拿到了同一个线程，可能会读到线程 A 留下的脏数据（比如用户 Session），导致严重的安全事故。

-----

## 总结与预告

ThreadLocal 是 Java 并发中一把精致的手术刀：

1.  **存储**：数据存在 `Thread` 自己的 `ThreadLocalMap` 里，天然线程隔离。
2.  **寻址**：使用线性探测法解决哈希冲突。
3.  **弱引用**：Key 是弱引用，为了让 ThreadLocal 对象能被回收。
4.  **泄漏**：Value 是强引用，必须配合 `finally { remove() }` 才能根治内存泄漏。

我们已经掌握了多线程的存储、控制和协作。但是，在现代微服务和高并发系统中，我们往往需要更复杂的异步编排：**“做完 A，把结果交给 B，同时并发做 C，等 B 和 C 都做完了，再做 D”**。

用 `Thread` 和 `Future` 写这种逻辑会把人写疯。JDK 8 引入了一个真正的异步编程神器，实现了 Promise 模式和函数式流式处理。

**下一篇预告**：
我们将探索 Java 异步编程的巅峰——**CompletableFuture**。它是如何用“观察者模式 + 驱动栈”实现异步回调链的？它是如何完美替代 `Future` 的？敬请期待《Java 并发与多线程（六）：CompletableFuture 的异步编排与函数式流水线》。
