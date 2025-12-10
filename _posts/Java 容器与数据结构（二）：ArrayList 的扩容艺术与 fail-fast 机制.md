这是《Java 容器与数据结构：从入门到入魔》系列的第二篇。

在上一篇中，我们彻底剖析了 Java 数组的内存模型。今天，我们在此基础上构建 Java 中最常用的容器——`ArrayList`。你可能觉得它很简单，不就是封装了一个数组吗？

但你是否思考过：为什么它的默认扩容是 1.5 倍而不是 2 倍？为什么 `elementData` 被设计为 `transient`？`subList` 到底隐藏着什么内存泄漏的风险？`ConcurrentModificationException` 真的是为了并发安全吗？

让我们通过源码（基于 JDK 8+）和字节码，揭开它的面纱。

-----

## layout: post title:  "Java 容器与数据结构（二）：ArrayList 的扩容艺术与 fail-fast 机制" date:   2025-11-26 10:00:00 +0800 categories: [Java, 容器源码, 性能优化] tags: [ArrayList, 扩容机制, Fail-Fast, 内存泄漏, 源码分析]

## 系列前言

这是《Java 容器与数据结构：从入门到入魔》系列的第二篇。我们正在一步步搭建 Java 数据结构的知识大厦。上一篇我们讲了地基——**数组**，今天我们来讲讲建在地基上的第一层楼——**ArrayList**。

## 第一章：核心架构与序列化的秘密

`ArrayList` 的本质是一个**动态数组**。它解决了数组长度不可变的痛点，代价是引入了扩容和拷贝的开销。

### 1.1 核心字段

打开源码，映入眼帘的是这两个关键字段：

```java
// 存储元素的数组缓冲区
transient Object[] elementData; 

// 列表的大小（包含的元素数量）
private int size;
```

注意到了吗？`elementData` 被声明为 **`transient`**。
我们在学习 Java 基础时知道，`transient` 关键字意味着该字段**不会被默认序列化**。

**问题来了：** 如果 `elementData` 不被序列化，那我们存进去的数据岂不是在网络传输或落盘时都丢失了？

### 1.2 序列化的“性能洁癖”

`ArrayList` 并没有丢失数据，而是实现了自定义的 `writeObject` 和 `readObject` 方法。

**原因分析**：
假设我们 `new ArrayList<>(100)`，但只 `add` 了一个元素。

  * `elementData.length` = 100 (容量)
  * `size` = 1 (实际大小)

如果我们使用默认的序列化机制，Java 会傻乎乎地把整个 `elementData` 数组（包含 99 个 null）全部序列化。这不仅浪费时间，还浪费磁盘空间/带宽。

**源码中的智慧**：
`ArrayList` 在 `writeObject` 中，**只遍历并序列化了前 `size` 个有效元素**。

```java
private void writeObject(java.io.ObjectOutputStream s) throws java.io.IOException {
    // ... 前置逻辑 ...
    s.defaultWriteObject(); // 写入非 transient 字段 (如 size)

    // 只写入实际存在的元素
    for (int i = 0; i < size; i++) {
        s.writeObject(elementData[i]);
    }
}
```

-----

## 第二章：扩容机制 (The Art of Expansion)

扩容是 `ArrayList` 的灵魂，也是面试的重灾区。

### 2.1 懒加载 (Lazy Initialization)

在 JDK 7 以前，`new ArrayList()` 会直接创建一个长度为 10 的数组。
但在 JDK 8 之后，官方做了一个优化：

```java
// JDK 8+
private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

public ArrayList() {
    // 构造时仅仅赋予一个空数组，不占用内存
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
}
```

只有当你**第一次**调用 `add()` 时，扩容逻辑才会触发，将数组初始化为默认容量 **10**。这是一种典型的**懒加载**策略，极大减少了创建大量空列表时的内存浪费。

### 2.2 1.5 倍的玄学公式

当数组满时，`grow()` 方法会被调用。核心代码如下：

```java
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    // 新容量 = 旧容量 + (旧容量 / 2)
    // 右移一位相当于除以 2
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    
    // ... 处理溢出等边界情况 ...
    
    // 真正的扩容操作：申请新数组 + 拷贝旧数据
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

**为什么是 1.5 倍（`old + old >> 1`）？**

  * **对比 2 倍扩容**：如果采用 2 倍扩容（如 C++ 的 `std::vector` 的某些实现），虽然扩容次数更少，但可能会导致内存碎片的浪费更加严重。
  * **对比 1 倍扩容**：如果每次只扩容一点点（比如 +10），那么 `copy` 的频率太高，导致添加操作的时间复杂度退化为 O(N)。
  * **1.5 倍的平衡**：这是一个经验值。它试图在“扩容次数”（CPU 开销）和“内存浪费”（空间开销）之间找到平衡点。此外，1.5 倍扩容使得旧的内存块在释放后，更有可能被后续的内存分配重用（这涉及到内存分配器的底层原理，如 k-allocator）。

> **字节码细节**：使用 `>> 1` 位运算代替 `/ 2` 除法运算，在早期 CPU 上有极其微小的性能优势，但更多是 C/C++ 程序员遗留下来的“肌肉记忆”和代码风格。

-----

## 第三章：Fail-Fast 机制的真相

你在遍历 List 时遇到过 `ConcurrentModificationException` (CME) 吗？

### 3.1 什么是 Fail-Fast？

Fail-Fast（快速失败）不是一种修复机制，而是一种**错误检测机制**。它的哲学是：“如果我发现现在的状态不对劲，与其继续执行导致不可预知的结果，不如立刻崩溃并报错。”

### 3.2 幕后黑手：`modCount`

`ArrayList` 父类 `AbstractList` 中有一个字段：

```java
protected transient int modCount = 0;
```

每当你执行 `add`, `remove`, `clear` 等**修改结构**的操作时，`modCount` 都会自增。

### 3.3 触发原理

当你使用 `foreach` (增强 for 循环) 遍历列表时，编译器实际上生成了一个 `Iterator`。在迭代器初始化的瞬间，它会记录当前的 `modCount`：

```java
private class Itr implements Iterator<E> {
    int expectedModCount = modCount; // 记录快照

    public E next() {
        checkForComodification(); // 每次获取元素前，先检查
        // ...
    }
    
    final void checkForComodification() {
        // 如果现在的 modCount 和我记录的不一样，说明有人动过列表！
        if (modCount != expectedModCount)
            throw new ConcurrentModificationException();
    }
}
```

这就是为什么下面的代码会抛出异常：

```java
for (String s : list) {
    if ("target".equals(s)) {
        list.remove(s); // 危险！直接调用 list 的 remove，修改了 modCount
    }
}
```

**正解**：使用 `Iterator` 自己的 `remove()` 方法。它会在删除元素后，**手动同步** `expectedModCount = modCount`，从而骗过下一次检查。

-----

## 第四章：隐藏的内存泄漏——SubList

`ArrayList` 提供了一个看起来很方便的方法：`subList(int fromIndex, int toIndex)`。

```java
List<String> masterList = new ArrayList<>(10000);
// ... 填充 10000 个数据 ...
List<String> sub = masterList.subList(0, 10); // 只取前 10 个
```

### 4.1 视图 (View) 而非拷贝

很多人误以为 `subList` 生成了一个新的 List。**错！**

`SubList` 是 `ArrayList` 的一个内部类，它**没有自己的数组**，而是直接持有**原数组的引用** (`parent.elementData`)，只是记录了 `offset` 和 `size`。

### 4.2 内存泄漏场景

如果你有一个巨大的 `masterList` (比如 100MB)，你调用 `subList` 取了其中一小部分并返回给前端，然后丢弃了 `masterList`。

```java
public List<Data> getData() {
    ArrayList<Data> hugeData = loadFromDb(); // 100MB
    return hugeData.subList(0, 5); // 返回一个 SubList
}
```

**后果**：虽然你以为 `hugeData` 可以被 GC 回收了，但因为返回的 `SubList` 内部强引用了 `hugeData` 的底层数组，导致**这 100MB 内存永远无法被释放**。

**最佳实践**：

```java
// 显式创建新列表，切断与旧数组的联系
return new ArrayList<>(hugeData.subList(0, 5));
```

-----

## 总结与预告

`ArrayList` 的设计充满了工程上的权衡：

1.  **空间换时间**：通过 1.5 倍扩容预留空间，将添加元素的平均复杂度降为 O(1)。
2.  **序列化优化**：通过 `transient` 和自定义逻辑，避免传输无效的 null 值。
3.  **Fail-Fast**：通过 `modCount` 版本号机制，尽早暴露并发修改的风险。

但是，`ArrayList` 在**头部插入**或**中间删除**时，依然需要 `System.arraycopy` 搬运大量数据，效率极低。

有没有一种结构，插入删除极快，但查询却很慢呢？

**下一篇预告**：
我们将剖析链表结构的代表——`LinkedList`，并对比它与 `ArrayList` 在现代 CPU 缓存架构下的真实性能差异（Spoiler: 链表在现代计算机上可能比你想象的要慢）。敬请期待《Java 容器与数据结构（三）：LinkedList 与 CPU 缓存的爱恨情仇》。
