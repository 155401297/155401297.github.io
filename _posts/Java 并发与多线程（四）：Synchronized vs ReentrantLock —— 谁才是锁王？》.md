这是《Java 并发与多线程：从入门到入魔》系列的第四篇。

在上一篇中，我们解析了并发包的基石 CAS 和 AQS。今天，我们将通过一场“擂台赛”，对比 Java 历史上最著名的两把锁：**`synchronized`** 和 **`ReentrantLock`**。

曾经（JDK 1.5 以前），`synchronized` 被称为“重量级锁”，性能被 `ReentrantLock` 吊打。但经过 JDK 1.6 的“锁升级”革命后，战局发生了逆转。现在的面试官如果问你“为什么 ReentrantLock 比 synchronized 快”，你要小心了——因为这句话的前提可能已经不存在了。

-----

## layout: post title:  "Java 并发与多线程（四）：Synchronized vs ReentrantLock —— 谁才是锁王？" date:   2025-12-03 10:00:00 +0800 categories: [Java, 并发编程, 源码分析] tags: [Synchronized, ReentrantLock, 锁升级, 偏向锁, Condition]

## 系列前言

这是《Java 并发与多线程：从入门到入魔》系列的第四篇。锁，是解决线程安全最直接的方式。Java 提供了两种加锁模式：一种是 JVM 原生的 `synchronized` 关键字，另一种是 JDK API 层面的 `ReentrantLock`。它们都能保证线程安全，都支持可重入，但底层实现和适用场景却大相径庭。

## 第一章：Synchronized 的逆袭——锁升级之路

在 JDK 1.6 之前，`synchronized` 确实很慢。因为它直接调用操作系统的 Mutex Lock，每次加锁都需要从用户态切换到内核态，开销巨大。

但 JDK 1.6 引入了**锁升级 (Lock Escalation)** 机制，让 `synchronized` 变得“能屈能伸”。

### 1.1 对象头里的秘密 (Mark Word)

Java 对象在堆内存中包含一个**对象头 (Object Header)**，其中的 **Mark Word** 记录了锁的状态。

锁的状态会随着竞争的激烈程度，从低到高依次升级：

1.  **偏向锁 (Biased Lock)**：

      * **场景**：只有一个线程在访问同步块。
      * **原理**：JVM 认为“锁通常是被同一个线程获取的”。第一次获取锁时，直接把当前线程 ID 贴在 Mark Word 上。下次这个线程再来，对比一下 ID，如果是自己，**完全不需要同步操作**，连 CAS 都不用。这是极速模式。

2.  **轻量级锁 (Lightweight Lock)**：

      * **场景**：有两个线程交替执行，没有发生真正的并发争抢。
      * **原理**：当第二个线程尝试获取偏向锁失败时，锁升级为轻量级锁。线程会在自己的栈帧中创建一个 Lock Record，用 **CAS** 试图将对象的 Mark Word 替换为指向 Lock Record 的指针。如果成功，就拿到了锁。

3.  **重量级锁 (Heavyweight Lock)**：

      * **场景**：发生了激烈的并发竞争（如 CAS 自旋失败多次）。
      * **原理**：锁膨胀为重量级锁。此时 Mark Word 指向 **Monitor (监视器)** 对象。竞争失败的线程会被真的**挂起 (Blocked)**，进入操作系统层面的等待队列。

**结论**：在低竞争环境下，`synchronized` 几乎没有额外开销；只有在极高竞争下，它才会变成真正的“重量级锁”。

-----

## 第二章：ReentrantLock 的特异功能

既然 `synchronized` 变快了，还要 `ReentrantLock` 干嘛？
因为 `ReentrantLock` 提供了 `synchronized` 做不到的**高级功能**。它是一把“瑞士军刀”。

### 2.1 响应中断 (`lockInterruptibly`)

`synchronized` 只有两种结局：要么拿到锁，要么死等。如果发生死锁，你连把线程 kill 掉的机会都没有（它不吃中断信号）。

`ReentrantLock.lockInterruptibly()` 允许在等待锁的过程中**响应中断**。这对于避免死锁和编写健壮的并发程序至关重要。

### 2.2 尝试非阻塞获取 (`tryLock`)

```java
if (lock.tryLock()) {
    try {
        // 拿到锁了，干活
    } finally {
        lock.unlock();
    }
} else {
    // 没拿到，我不等了，去干别的事
}
```

这在处理“如果被锁了就降级服务”的场景下非常有用。

### 2.3 超时获取 (`tryLock(time, unit)`)

“我就等你 5 秒，5 秒没拿到我就走了。”这是解决死锁和防止线程饥饿的神器。

-----

## 第三章：公平锁的代价 (Fairness)

`synchronized` 只能是非公平锁。
`ReentrantLock` 默认是非公平锁，但可以构造为公平锁：`new ReentrantLock(true)`。

### 3.1 什么是公平？

  * **公平锁**：严格按照“先来后到”排队。队列里的第一个线程先拿锁。
  * **非公平锁**：新来的线程可以“插队”。如果新线程刚到，正好锁被释放了，它就直接抢走，而不用去队列尾部排队。

### 3.2 为什么默认是非公平？

**为了性能。**

假设线程 A 释放锁，唤醒线程 B。线程 B 从被唤醒到真正开始运行，需要漫长的上下文切换时间。
在这段时间里，线程 C 来了。如果是非公平模式，C 可以利用这段“时间差”，立刻拿到锁，执行完，然后释放。
等 B 真正醒来时，C 已经干完活了。

**实测数据**：非公平锁的吞吐量通常是公平锁的 **5-10 倍**。除非你的业务逻辑强制要求顺序性，否则**永远不要使用公平锁**。

-----

## 第四章：精确打击——Condition

`synchronized` 配合 `Object.wait()` 和 `notify()` 只能实现简单的等待/通知。
最大的痛点是：`notify()` 是**随机**唤醒一个等待线程，你无法指定唤醒谁。

`ReentrantLock` 配合 **`Condition`** 接口，可以实现**分组唤醒**。

**经典案例：生产者-消费者**
我们需要两个队列：满队列（生产者等）、空队列（消费者等）。

```java
ReentrantLock lock = new ReentrantLock();
Condition notFull  = lock.newCondition(); // 队列不满（生产者等这个信号）
Condition notEmpty = lock.newCondition(); // 队列不空（消费者等这个信号）

// 生产者
void put(Object x) {
    lock.lock();
    try {
        while (count == items.length) 
            notFull.await(); // 满了，生产者在 notFull 房间休息
        enqueue(x);
        notEmpty.signal(); // 此时一定有货了，只唤醒“消费者”
    } finally { lock.unlock(); }
}
```

使用 `Condition`，我们可以避免“虚假唤醒”，也不需要唤醒所有线程（`notifyAll`）造成 CPU 惊群效应。

-----

## 第五章：终极对决 (Summary)

面试官问：**我该选谁？**

**原则一：默认使用 `synchronized`**

  * **简洁**：不需要手动 `unlock`，代码更少，不会因为忘记释放锁导致死锁。
  * **优化**：JVM 团队持续在优化它（偏向锁、逃逸分析锁消除）。
  * **监控**：在 Thread Dump 中能更清晰地看到锁定信息。

**原则二：以下情况必选 `ReentrantLock`**

1.  你需要**超时等待** (`tryLock`)。
2.  你需要**中断等待** (`lockInterruptibly`)。
3.  你需要**公平性**（虽然极少见）。
4.  你需要**多路条件通知** (`Condition`)。

**总结**：`synchronized` 是自动挡轿车，好开、省心，够用；`ReentrantLock` 是手动挡赛车，功能强大，操作复杂，但能应对极端路况。

-----

## 预告

锁解决了并发安全问题，但有一种更隐蔽的“内存泄漏”杀手，它不加锁，却把数据绑定在线程身上。如果你在 Tomcat 或线程池中使用它而忘记清理，你的内存就会一点点被蚕食殆尽。

**下一篇预告**：
我们将深入 **ThreadLocal** 的源码，解密它内部那个神秘的 `ThreadLocalMap`，以及为什么它的 Key 是弱引用 (WeakReference)？敬请期待《Java 并发与多线程（五）：ThreadLocal 的内存泄漏陷阱与弱引用之谜》。
