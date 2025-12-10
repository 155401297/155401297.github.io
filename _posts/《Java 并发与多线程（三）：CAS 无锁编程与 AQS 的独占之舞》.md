这是《Java 并发与多线程：从入门到入魔》系列的第三篇。

在上一篇中，我们用 `volatile` 解决了可见性问题，但它对 `i++` 这种复合操作的原子性无能为力。这就好比你给所有线程发了通知“数据变了”，但当它们同时冲上去修改数据时，依然会发生踩踏事故。

今天，我们将从“悲观锁”的思维定势中跳出来，看看 JDK 工程师是如何利用硬件底层的 **CAS (Compare And Swap)** 指令，实现“无锁胜有锁”的境界；以及 Doug Lea 是如何用一个简单的 `int` 变量和一个队列，构建起整个 Java 并发包的基石——**AQS**。

-----

## layout: post title:  "Java 并发与多线程（三）：CAS 无锁编程与 AQS 的独占之舞" date:   2025-12-02 10:00:00 +0800 categories: [Java, 并发编程, 底层原理] tags: [CAS, AQS, AtomicInteger, 自旋锁, Unsafe]

## 系列前言

这是《Java 并发与多线程：从入门到入魔》系列的第三篇。
如果说 `volatile` 是轻量级的防御，那么 `synchronized` 就是重型的装甲。但在高并发场景下，重型装甲的“穿戴”（线程切换）成本太高。于是，**CAS (Compare And Swap)** 应运而生。它是 `java.util.concurrent` (J.U.C) 包中最底层的原子能力。没有 CAS，就没有现在的 Java 并发包。

## 第一章：什么是 CAS？(乐观锁的极致)

我们在做数据库更新时，常有这样的逻辑：
`UPDATE account SET balance = 100 WHERE id = 1 AND balance = 90;`
这句话的意思是：“如果原本余额是 90（我预期的值），就把它改成 100；如果不是 90（被别人改过了），我就什么都不做。”

这就是 **CAS (比较并交换)** 的核心思想。它包含三个操作数：

1.  **V (Memory Value)**: 内存中的实际值。
2.  **E (Expected Value)**: 我预期的旧值。
3.  **N (New Value)**: 我想设置的新值。

**原子操作**：只有当 `V == E` 时，才把 `V` 更新为 `N`。否则，说明发生了冲突，更新失败（通常会重试）。

### 1.1 为什么它比 synchronized 快？

  * **Synchronized (悲观锁)**：认为冲突总会发生。所以不管有没有人抢，先加锁再说。线程会被挂起（Blocked），涉及**用户态到内核态的切换**，开销极大。
  * **CAS (乐观锁)**：认为冲突很少发生。先试着改一下，如果成功了就赚了；如果失败了，我就**自旋 (Spin)** 重试，线程始终在 CPU 上运行，没有上下文切换的开销。

-----

## 第二章：Unsafe 类与 CPU 指令

Java 无法直接操作内存，但 JDK 提供了一个后门——**`sun.misc.Unsafe`** 类。
`AtomicInteger.incrementAndGet()` 的底层就是靠它：

```java
// JDK 8 AtomicInteger 源码
public final int getAndAddInt(Object var1, long var2, int var4) {
    int var5;
    do {
        // 1. 获取当前内存中的值 (预期值 E)
        var5 = this.getIntVolatile(var1, var2);
        
        // 2. 只有当内存值还是 var5 时，才设置为 var5 + var4
        // compareAndSwapInt 是 Native 方法
    } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

    return var5;
}
```

**底层原理**：
`Unsafe` 的 native 方法最终调用的是 C++ 代码，进而映射到 CPU 的汇编指令 `cmpxchg`。

  * **多核 CPU**：在多核下，`cmpxchg` 指令前会加上 `lock` 前缀 (`lock cmpxchg`)。它会锁定总线或缓存行，确保这一步“比较+交换”的操作在硬件层面是**原子**的。

**结论**：CAS 不是软件层面的技巧，而是**硬件指令集的直接暴露**。

-----

## 第三章：CAS 的三大缺陷

世界上没有免费的午餐，CAS 也有副作用。

### 3.1 ABA 问题

假如这一秒内存里的值是 A，下一秒被线程 X 改成了 B，再下一秒又被线程 Y 改回了 A。
此时线程 Z 过来检查：预期是 A，内存也是 A。**CAS 成功了**。
但是，**现在的 A 已经不是原来的 A 了**。

  * **生活案例**：你倒了一杯水（A），离开了一会。别人喝了（B），又给你接满了水（A）。你回来看到水还是满的，喝了一口——但杯子其实已经脏了。
  * **解决**：**加版本号**。`AtomicStampedReference` 类在比较引用的同时，还会比较一个 `int stamp`（版本戳）。只有引用和版本号都对得上，才允许交换。

### 3.2 自旋时间过长 (CPU 飙升)

如果并发冲突极高，所有线程都在 `do-while` 循环里疯狂失败、重试。这会把 CPU 也就是吃满，却没干什么正事。

  * **解决**：如果要等待很久，不如直接挂起线程（使用 `LongAdder` 分段减少冲突，或者直接用 Lock）。

### 3.3 只能保证一个变量的原子性

CAS 无法同时锁住 `x` 和 `y` 两个变量。

  * **解决**：把多个变量封装成一个对象，用 `AtomicReference` 进行 CAS。

-----

## 第四章：AQS —— 并发包的“心脏”

CAS 只能解决原子更新的问题，但如果有 100 个线程抢资源，抢不到的线程总不能一直自旋烧 CPU 吧？它们需要排队、需要挂起、需要被唤醒。

为此，Java 并发之父 Doug Lea 设计了 **AbstractQueuedSynchronizer (AQS)**。
它是 `ReentrantLock`、`CountDownLatch`、`Semaphore` 等一切并发工具的**父类**。

### 4.1 AQS 的核心模型

AQS 的核心非常简单，就两样东西：

1.  **state (资源状态)**：一个 `volatile int` 变量。
      * 对于 `ReentrantLock`，state=0 表示无锁，state=1 表示有锁，state\>1 表示重入次数。
      * 对于 `CountDownLatch`，state 表示倒计时的数量。
2.  **CLH 队列 (等待队列)**：一个双向链表。
      * 抢不到锁的线程，会被封装成 `Node` 节点，扔进这个队列里排队睡觉（`LockSupport.park()`）。

### 4.2 独占锁的加锁流程 (以 ReentrantLock 为例)

当你调用 `lock.lock()` 时，AQS 内部发生了什么？

1.  **尝试抢锁 (Try)**：
    使用 CAS 尝试将 `state` 从 0 修改为 1。
      * 成功：我是当前线程，设置 `exclusiveOwnerThread = Thread.currentThread()`。拿下！
2.  **入队 (Enqueue)**：
    CAS 失败，说明锁被占了。将当前线程封装为 Node，通过 CAS 尾插法加入 CLH 队列。
3.  **阻塞 (Park)**：
    在队列里，调用 `LockSupport.park()` 将自己挂起，让出 CPU，等待被唤醒。

### 4.3 释放锁流程

当你调用 `lock.unlock()` 时：

1.  修改 `state` 减 1。
2.  如果 `state` 归零，说明锁彻底释放。
3.  **唤醒 (Unpark)**：
    找到队列头部的下一个节点（Head.next），调用 `LockSupport.unpark(thread)` 叫醒它：“老二，轮到你了，起来干活！”

-----

## 总结与预告

AQS 的美妙之处在于它是一个**模板方法模式**的集大成者：

  * 它写好了排队、阻塞、唤醒的所有脏活累活。
  * 子类（如 ReentrantLock）只需要实现 `tryAcquire`（怎么才算抢到锁）和 `tryRelease`（怎么才算释放锁）这两个简单的逻辑即可。

理解了 CAS 和 AQS，你就理解了 Java 锁的半壁江山。

但是，`ReentrantLock` 和 `synchronized` 到底选谁？`ReentrantLock` 的“公平锁”真的公平吗？`Condition` 又是什么神器？

**下一篇预告**：
我们将对比 Java 两大锁机制的巅峰对决。敬请期待《Java 并发与多线程（四）：Synchronized vs ReentrantLock —— 谁才是锁王？》。
