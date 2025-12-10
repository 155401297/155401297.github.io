这是《Java 并发与多线程：从入门到入魔》系列的第二篇。

在上一篇中，我们构建了线程池来复用线程。但是，当这群“复用的线程”开始同时访问同一个变量时，噩梦就开始了。

你是否遇到过这样的诡异现象：你在一个线程里把开关 `flag` 设为了 `true`，但另一个正在死循环检测 `flag` 的线程却视而不见，依然跑得欢快？

这一篇，我们将深入 CPU 的三级缓存架构，揭开 **JMM (Java Memory Model)** 的面纱，并解释为什么 `volatile` 是 Java 并发中最轻量级、却也最容易被误用的关键字。

-----

## layout: post title:  "Java 并发与多线程（二）：JMM 内存模型与 volatile 的底层原理" date:   2025-12-01 10:00:00 +0800 categories: [Java, 并发编程, 底层原理] tags: [JMM, volatile, 可见性, 内存屏障, MESI协议]

## 系列前言

这是《Java 并发与多线程：从入门到入魔》系列的第二篇。
并发编程的三大核心问题是：**可见性 (Visibility)**、**原子性 (Atomicity)** 和 **有序性 (Ordering)**。
今天我们要攻克的是**可见性**和**有序性**。如果你不理解 JMM，你写的并发程序就如同在沙堆上建塔，随时可能因为 CPU 的一个优化指令而崩塌。

## 第一章：可见性的诡异现象

先来看一段经典的“死循环”代码：

```java
public class VisibilityDemo {
    // 没加 volatile
    private static boolean flag = true;

    public static void main(String[] args) throws InterruptedException {
        // 线程 A：死循环检测 flag
        new Thread(() -> {
            System.out.println("Thread A started.");
            while (flag) {
                // do nothing
            }
            System.out.println("Thread A finished.");
        }).start();

        Thread.sleep(1000); // 保证 A 先跑起来

        // 线程 B：修改 flag
        new Thread(() -> {
            System.out.println("Thread B tries to change flag.");
            flag = false;
            System.out.println("Thread B changed flag to false.");
        }).start();
    }
}
```

**运行结果**：
你以为线程 B 把 `flag` 改为 `false` 后，线程 A 会立刻跳出循环并打印 "Finished" 吗？
**错！** 线程 A 会**死循环运行下去**，程序永远不会结束。

为什么？明明 `flag` 是 `static` 变量，大家共用一份内存，为什么 A 看不到 B 的修改？

-----

## 第二章：一切源于 CPU 缓存 (The Hardware Reality)

要理解 Java 的行为，必须先看硬件。

### 2.1 CPU 哪怕太快了

CPU 的速度比内存（RAM）快 100 倍以上。如果 CPU 每执行一条指令都要去内存读写数据，那 CPU 99% 的时间都在“摸鱼”等待。
为了解决这个矛盾，现代计算机引入了 **CPU 缓存 (L1/L2/L3 Cache)**。

1.  **线程 A** 运行时，把 `flag` 变量从主内存读取到了自己的 **CPU 缓存**中。
2.  **线程 B** 运行时，也把 `flag` 读取到了自己的缓存中，并修改为 `false`，然后回写到主内存。
3.  **关键点**：线程 A 的缓存里，`flag` 依然是旧值 `true`。除非有人通知它，否则它永远不知道主内存里的值变了。

这就是**可见性问题**的物理根源。

### 2.2 缓存一致性协议 (MESI)

有人会问：“CPU 不是有 MESI 协议（缓存一致性协议）吗？B 修改了，应该会通知 A 失效啊？”

是的，但 MESI 协议有性能代价。为了极致的性能，CPU 往往引入了 **Store Buffer（写缓冲）** 和 **Invalidate Queue（失效队列）**，这导致了“短暂的”不一致。而在多线程的高速循环中，这个“短暂”可能就是无限长。

-----

## 第三章：JMM —— Java 的抽象层

Java 为了屏蔽不同操作系统和硬件的差异，定义了 **Java 内存模型 (JMM)**。

JMM 规定：

1.  **主内存 (Main Memory)**：所有变量都存在这里（类似于物理 RAM）。
2.  **工作内存 (Working Memory)**：每个线程有自己的独立空间（类似于 CPU Cache + 寄存器），存储了该线程用到的变量副本。
3.  **规则**：线程对变量的所有操作（读取、赋值）必须在**工作内存**中进行，不能直接读写主内存。

回到刚才的例子：线程 B 修改了 `flag`，仅仅是修改了自己的工作内存，还没来得及（或者不确定什么时候）同步回主内存；即使同步回去了，线程 A 也不确定什么时候会从主内存重新拉取。

-----

## 第四章：volatile 的两大神力

只要在 `flag` 前面加上 `volatile` 关键字，上面的程序就会立刻正常结束。
`volatile` 到底做了什么？

### 4.1 神力一：保证可见性 (Visibility)

当一个变量被声明为 `volatile` 时：

1.  **写操作**：JMM 会强制将该线程工作内存中的改变**立刻刷新回主内存**。
2.  **读操作**：JMM 会强制让该线程工作内存中的副本**失效**，必须从主内存重新读取。

**底层实现 (x86 架构)**：
JVM 会在生成的机器码中，给写操作指令加一个 `lock` 前缀（如 `lock addl`）。
这个 `lock` 指令会触发硬件层面的 **总线锁定** 或 **缓存锁定 (MESI)**，并清空 Store Buffer，确保数据立刻对其他 CPU 核心可见。

### 4.2 神力二：禁止指令重排序 (Ordering)

编译器和 CPU 为了优化性能，经常会打乱指令执行的顺序。
比如：

```java
int a = 1; // 1
boolean flag = true; // 2
```

CPU 可能会先执行 2，再执行 1，因为它们互不依赖。但在多线程环境下，这会导致灾难（比如双重检查单例模式 Double-Checked Locking 的失效）。

`volatile` 通过插入 **内存屏障 (Memory Barrier)** 来禁止重排序：

  * 在 `volatile` 写操作之前，插入 `StoreStore` 屏障。
  * 在 `volatile` 写操作之后，插入 `StoreLoad` 屏障。
  * ...

简单来说：**volatile 变量就像一道墙，墙上面的指令不能掉到墙下面，墙下面的也不能翻到墙上面。**

-----

## 第五章：volatile 不是万能药——原子性陷阱

这是面试中最最最常见的坑。

```java
public class AtomicityDemo {
    private volatile int count = 0;

    public void add() {
        count++; // 即使加了 volatile，这也是不安全的！
    }
}
```

如果你开启 10 个线程，每个线程调用 `add()` 10000 次，最终 `count` 的结果往往小于 100000。

**为什么？**
`count++` 这一行代码，在字节码层面其实是三步操作：

1.  `getstatic`：读取 count 的值。
2.  `iadd`：加 1。
3.  `putstatic`：写回 count。

`volatile` 只能保证第 1 步读取到的是最新值。
但如果线程 A 读到了 100，还没来得及写回，线程 B 也读到了 100。
A 加 1 变成 101 写回。
B 加 1 变成 101 写回。
**结果：两次自增，只加了 1。**

**结论**：`volatile` **不能**保证复合操作的原子性。要解决这个问题，必须使用 `synchronized` 或 `AtomicInteger`。

-----

## 总结与预告

`volatile` 是 Java 并发编程的轻量级武器：

1.  **可见性**：利用 CPU 的 `lock` 指令和缓存一致性协议，保证修改立刻被其他线程看到。
2.  **有序性**：利用内存屏障禁止指令重排序。
3.  **局限**：**无法保证原子性**（如 `i++`）。

既然 `volatile` 搞不定 `i++`，用 `synchronized` 加锁又太重（涉及内核态切换），有没有一种既能保证原子性，又像 `volatile` 一样轻量级的方案呢？

答案是有的。那就是 **CAS (Compare And Swap)**。

**下一篇预告**：
我们将剖析 Java 并发包 `java.util.concurrent` 的基石——**CAS 机制**与 **AQS (AbstractQueuedSynchronizer)**。为什么 `AtomicInteger` 比 `synchronized` 快？自旋锁（SpinLock）是空转浪费 CPU 吗？敬请期待《Java 并发与多线程（三）：CAS 无锁编程与 AQS 的独占之舞》。
