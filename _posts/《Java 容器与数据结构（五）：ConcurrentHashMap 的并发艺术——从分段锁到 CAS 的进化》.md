这是《Java 容器与数据结构：从入门到入魔》系列的第五篇。

在上一篇中，我们彻底搞懂了 `HashMap` 的底层原理。但所有的这些优化（位运算、红黑树）在多线程环境下都是“空中楼阁”——因为 `HashMap` 在并发写时会发生数据覆盖，扩容时甚至可能死循环（JDK 1.7）或丢失数据。

如果说 `HashMap` 是单兵作战的特种兵，那么 `ConcurrentHashMap` 就是协同作战的现代化军队。它是 Doug Lea 大神留给 Java 世界的瑰宝。今天，我们将见证它是如何从 JDK 1.7 的“分段锁”进化为 JDK 1.8 的“CAS + synchronized”原子级并发控制的。

-----

## layout: post title:  "Java 容器与数据结构（五）：ConcurrentHashMap 的并发艺术——从分段锁到 CAS 的进化" date:   2025-11-29 10:00:00 +0800 categories: [Java, 并发编程, 源码分析] tags: [ConcurrentHashMap, CAS, Synchronized, 分段锁, 扩容迁移]

## 系列前言

这是《Java 容器与数据结构：从入门到入魔》系列的第五篇。并发编程是 Java 的深水区，而 `ConcurrentHashMap` (下文简称 CHM) 则是深水区中的“定海神针”。面试官问 CHM，往往不是想知道怎么用，而是想考察你对 Java 内存模型 (JMM)、锁机制（CAS/ReentrantLock/Synchronized）以及高并发算法的综合理解。

## 第一章：时代的眼泪——JDK 1.7 的分段锁 (Segment)

要理解 JDK 1.8 的伟大，必须先看看 JDK 1.7 是如何解决并发问题的。

### 1.1 核心思想：化整为零

传统的 `Hashtable` 也就是简单粗暴地在所有方法上加 `synchronized`，这意味着一把**全局大锁**锁住了整个 Map。当线程 A 在写数据时，线程 B 连读数据都被阻塞。

JDK 1.7 的 CHM 引入了 **Segment (分段)** 的概念。
它将一个大的 Map 切分成 16 个（默认）小的 `Segment`。每个 `Segment` 本质上就是一个小型的 `Hashtable`（继承自 `ReentrantLock`）。

  * **写操作**：根据 Key 的 Hash 值映射到某个 `Segment`，只需要锁住这个特定的 `Segment`。其他 15 个 `Segment` 依然可以被其他线程并发访问。
  * **并发度**：理论上最高支持 16 个线程并发写。

### 1.2 局限性

虽然比 `Hashtable` 强，但分段锁依然有硬伤：

1.  **并发度受限**：默认 16 段，也就是最多 16 个并发。想要扩容 `concurrencyLevel` 很难（因为 Segment 数组一旦创建就不可变）。
2.  **内存开销**：每个 Segment 都要维护自己的 HashEntry 数组、锁对象等，内存碎片化严重。

-----

## 第二章：JDK 1.8 的革命——抛弃分段锁

JDK 1.8 完全重写了 CHM。**Segment 消失了**。
现在的 CHM 看起来和 `HashMap` 非常像：底层依然是 `Node` 数组 + 链表 + 红黑树。

那么，它是如何保证线程安全的？答案是：**CAS + synchronized 的细粒度控制**。

### 2.1 "无锁"的 CAS (Compare And Swap)

当一个新的 Key 插入时，如果对应的数组桶位（Bucket）是**空**的，CHM **不会加锁**。

```java
// JDK 1.8 putVal 源码简析
for (Node<K,V>[] tab = table;;) {
    Node<K,V> f; int n, i, fh;
    // 1. 如果位置是空的 (null)
    if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
        // 2. 直接尝试 CAS 插入
        if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
            break; // 插入成功，退出
        // 插入失败？说明刚才有别的线程抢先插入了，进入下一轮循环 (自旋)
    }
    // ...
}
```

**解析**：
JDK 利用 `Unsafe.compareAndSwapObject` 直接操作内存。

  * **乐观锁策略**：假设没有冲突，直接尝试写入。
  * **如果失败**：说明有并发干扰，自旋重试。
  * **优势**：对于从无到有的插入，完全没有锁的上下文切换开销。

### 2.2 锁住“头节点”

如果桶位**不为空**（发生了哈希碰撞），CAS 搞不定了，这时候必须加锁。
但 JDK 1.8 没有用 `ReentrantLock`，而是用了 **`synchronized`**，并且**只锁住链表/红黑树的头节点**。

```java
// ... 接上文
else {
    synchronized (f) { // f 就是当前桶位的头节点
        if (tabAt(tab, i) == f) { // 双重检查 (Double Check)
            // 执行链表追加或红黑树插入操作
        }
    }
}
```

**为什么回归 synchronized？**
很多人认为 `synchronized` 慢，那是老黄历了。

1.  **JVM 优化**：JDK 6 以后引入了偏向锁、轻量级锁、锁消除等机制，在低竞争下性能极佳。
2.  **内存节省**：`ReentrantLock` 每个节点都需要继承 AQS 或持有一个 Lock 对象，内存开销大。而 `synchronized` 直接利用对象头的 Mark Word，无需额外内存。
3.  **粒度极细**：锁的粒度不再是 Segment，而是 **HashEntry 的头节点**。数组长度是多少（比如 1024），并发度理论上就是多少。

-----

## 第三章：多线程协作扩容 (Transfer)

这是 CHM 最最最复杂、也是最精彩的设计：**多线程并发扩容**。

当 `HashMap` 扩容时，单线程搬运数据，整个世界都静止了。
当 `CHM` 扩容时，它会喊：“**兄弟们，别光看着，来帮忙！**”

### 3.1 ForwardingNode (转发节点)

在扩容过程中，如果某个桶位的数据已经被迁移到了新数组，原数组的这个位置会被放置一个特殊的节点：**`ForwardingNode`** (hash 值为 -1，MOVED)。

### 3.2 协作机制

当线程 A 正在扩容时，线程 B 想要 `put` 数据：

1.  线程 B 计算 Hash，找到对应的桶。
2.  发现桶里放的是 `ForwardingNode`。
3.  线程 B 意识到：“哦，正在扩容，而且这个位置已经搬完了，但我不能干等。”
4.  线程 B 调用 `helpTransfer()` 方法，**加入扩容大军**，帮助线程 A 迁移其他还没搬运的桶位。

这种机制极大地利用了多核 CPU 的算力，让扩容速度随着线程数的增加而加快，而不是导致阻塞。

-----

## 第四章：计数器的黑科技——LongAdder

`CHM.size()` 是怎么实现的？
如果你以为只是简单的 `AtomicLong` 自增，那就太小看 Doug Lea 了。

在高并发下，`AtomicLong` 会因为大量线程同时 CAS 修改同一个变量（Cache Line 伪共享）而导致 CPU 缓存频繁失效，性能急剧下降。

CHM 采用了类似 `LongAdder` 的机制：**`baseCount` + `CounterCell` 数组**。

  * **低并发**：直接 CAS 修改 `baseCount`。
  * **高并发**：如果 `baseCount` 修改失败，线程会随机映射到 `CounterCell` 数组的某个槽位进行 CAS 操作。
  * **求和**：`size()` = `baseCount` + 所有 `CounterCell` 的值。

这本质上也是**分段锁**思想在计数器领域的应用——**分散热点**。

-----

## 总结与预告

`ConcurrentHashMap` 在 JDK 1.8 的进化展示了并发编程的至高境界：

1.  **极简**：抛弃 Segment，回归数组结构。
2.  **极细**：使用 CAS 处理空节点，`synchronized` 锁头节点，将锁粒度降到最低。
3.  **协作**：引入 `ForwardingNode`，让写线程参与扩容，变阻塞为并发。
4.  **分散**：利用 `CounterCell` 解决全局计数器的热点问题。

读懂 CHM 1.8，你就读懂了 Java 并发包的一半智慧。

至此，我们的**Map 三部曲**（HashMap -\> LinkedHashMap/TreeMap -\> ConcurrentHashMap）暂告一段落（TreeMap 等留待后续番外篇）。

**下一篇预告**：
我们将离开“存储”，进入“线程”的世界。线程池（ThreadPoolExecutor）是如何复用线程的？如果不小心设置了错误的参数，会导致怎样的生产事故？敬请期待《Java 并发与多线程（一）：线程池的七个参数与拒绝策略的血泪史》。
