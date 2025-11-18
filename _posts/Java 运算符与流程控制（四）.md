---
layout: post
title:  "Java 运算符与流程控制（四）：循环揭秘 (for-each 语法糖与 LinkedList 性能陷阱)"
date:    2025-11-21 10:00:00 +0800
categories: [Java, 核心技术, 底层原理]
tags: [流程控制, for, for-each, Iterator, 字节码, ConcurrentModificationException]
---

## 系列回顾

在前三篇中，我们深入剖析了 Java 的运算符（`i++` 字节码、`Integer` 缓存池）和分支结构（`switch` 的 `tableswitch`/`lookupswitch` 实现、`String` 匹配原理）。本篇是本系列的终章（上），我们将彻底解构 Java 的循环体系。

---

## 第九章：`while` 与 `do-while`

`while` 和 `do-while` 是最基本的循环结构，它们在字节码层面清晰地展示了“循环”的本质：**条件跳转**。

### 9.1 `while` (先测后执)

```java
int i = 0;
while (i < 3) {
    i++;
}
````

  * **字节码 (简化)**：
    ```bytecode
     0: iconst_0    // i = 0
     1: istore_1
     2: goto 8      // [关键] 无条件跳转到 8 (循环条件检查)
     5: iinc 1, 1   // i++ (循环体)
     8: iload_1     // 加载 i
     9: iconst_3    // 加载 3
    10: if_icmplt 5 // [关键] if (i < 3) a, goto 5 (回到循环体)
    13: return      // 循环结束
    ```
  * **分析**：`while` 循环在字节码中被编译为一个“入口检查”。`goto 8` 先跳到条件判断，如果 `if_icmplt` (if integer comparison less than) 成立，则 `goto 5` 执行循环体，执行完后再 `goto 8` 回到检查点。

### 9.2 `do-while` (先执后测)

```java
int i = 0;
do {
    i++;
} while (i < 3);
```

  * **字节码 (简化)**：
    ```bytecode
     0: iconst_0    // i = 0
     1: istore_1
     2: iinc 1, 1   // i++ (循环体，直接执行)
     5: iload_1     // 加载 i
     6: iconst_3    // 加载 3
     7: if_icmplt 2 // [关键] if (i < 3) goto 2 (回到循环体)
    10: return      // 循环结束
    ```
  * **分析**：`do-while` **没有入口跳转**，它直接执行循环体（指令 2），然后在**末尾**进行条件检查 `if_icmplt`。
  * **结论**：`do-while` 保证了循环体**至少执行一次**。

-----

## 第十章：`for` 循环的经典结构

`for(int i=0; i < 10; i++)` 是最经典的循环。

```java
for (int i = 0; i < 3; i++) {
    // 循环体
}
```

  * **字节码 (简化)**：

    ```bytecode
     0: iconst_0    // int i = 0 (初始化)
     1: istore_1
     2: goto 8      // [关键] 跳转到 8 (条件检查)
     5: ...         // 循环体
     6: iinc 1, 1   // i++ (更新)
     8: iload_1     // 加载 i
     9: iconst_3    // 加载 3
    10: if_icmplt 5 // if (i < 3) goto 5 (回到循环体)
    13: return
    ```

  * **分析**：`for` 循环在字节码层面被**完全重写 (re-written)** 了。

    1.  `int i = 0` (初始化) **只执行一次**。
    2.  `i < 3` (条件) 变成了 `while` 循环的**入口检查** (`if_icmplt`)。
    3.  `i++` (更新) 被放到了**循环体的末尾**。

  * **结论**：`for` 循环本质上就是 `while` 循环的一个“语法糖”全家桶。

  * **无限循环**：`for(;;)` 和 `while(true)` 在编译后的字节码**完全相同**（一个无条件的 `goto` 循环），在性能上没有任何区别。

-----

## 第十一章：`for-each` 循环的“谎言” (重中之重)

`for-each` (增强型 `for` 循环) 是 JDK 5 引入的语法糖，它极大地简化了集合和数组的遍历。

**`for-each` 根本不是一个新的循环结构，而是编译器帮你写的 `Iterator` 或 `for(i)` 循环！**

### 11.1 Case 1：`Iterable` (集合)

当你对 `List`, `Set` 等实现了 `Iterable` 接口的类使用 `for-each`：

```java
// 你写的代码：
List<String> list = ...;
for (String s : list) {
    System.out.println(s);
}

// 编译器“翻译”后的代码 (伪代码)：
List<String> list = ...;
Iterator<String> iterator = list.iterator(); // 1. 获取迭代器
while (iterator.hasNext()) {               // 2. 检查
    String s = iterator.next();            // 3. 取值
    System.out.println(s);
}
```

  * **结论**：对集合的 `for-each`，**本质上是 `Iterator` 迭代器**。

#### 史诗级陷阱：`ConcurrentModificationException`

**为什么在 `for-each` 中删除元素会崩溃？**

```java
List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"));
for (String s : list) {
    if ("b".equals(s)) {
        list.remove(s); // 100% 抛出 ConcurrentModificationException
    }
}
```

  * **源码分析**：

    1.  `for-each` 底层是 `iterator.next()`。
    2.  `ArrayList` 的 `Iterator` 内部有两个关键变量：`modCount`（列表的修改次数）和 `expectedModCount`（迭代器期望的修改次数）。
    3.  `iterator.next()` 在执行时，会先检查 `modCount == expectedModCount`。
    4.  当你调用 `list.remove(s)` 时，`ArrayList` 的 `modCount` **增加了**！
    5.  下次循环调用 `iterator.next()` 时，发现 `modCount` (4) 不等于 `expectedModCount` (3)，认为“迭代时发生了并发修改”，于是**主动抛出** `ConcurrentModificationException` (这是一种“快速失败”机制)。

  * **正解**：**必须使用迭代器自身的 `remove()` 方法**。

    ```java
    Iterator<String> it = list.iterator();
    while (it.hasNext()) {
        String s = it.next();
        if ("b".equals(s)) {
            it.remove(); // 正确！it.remove() 会同步更新 expectedModCount
        }
    }
    ```

### 11.2 Case 2：数组 (Array)

当你对**数组**使用 `for-each`：

```java
// 你写的代码：
int[] arr = {1, 2, 3};
for (int i : arr) {
    System.out.println(i);
}

// 编译器“翻译”后的代码：
int[] arr = {1, 2, 3};
for (int i = 0; i < arr.length; i++) { // 1. 变成了普通的 for-i 循环
    int element = arr[i];             // 2. 取值
    System.out.println(element);
}
```

  * **结论**：对数组的 `for-each`，**本质上是 `for(i)` 索引循环**。

### 11.3 性能对决：`LinkedList` 上的 `for(i)` vs `for-each`

理解了 `for-each` 的本质，就能看懂这个经典的性能陷阱。

```java
List<String> list = new LinkedList<>();
// ... 假设 list 中有 100,000 个元素

// 跑法 1：`for-each`
long start1 = System.nanoTime();
for (String s : list) {
    // ...
}
System.out.println("for-each: " + (System.nanoTime() - start1)); // 极快 (毫秒级)

// 跑法 2：`for(i)`
long start2 = System.nanoTime();
for (int i = 0; i < list.size(); i++) {
    list.get(i); // 灾难的根源
}
System.out.println("for(i): " + (System.nanoTime() - start2)); // 极慢 (分钟级)
```

  * **分析**：
      * **跑法 1 (`for-each`)**：`javac` 翻译为 `Iterator`。`LinkedList` 的迭代器通过内部 `next` 指针移动，**时间复杂度 O(N)**。
      * **跑法 2 (`for(i)`)**：`LinkedList` 不是数组！`list.get(i)` 必须从头（或尾）开始遍历链表 `i` 次。
      * `get(0)` 很快，`get(99999)` 极慢。
      * 总时间复杂度 = 1 + 2 + 3 + ... + N = O(N^2)。
  * **结论**：**在 `LinkedList` 上使用 `for(i)` 是灾难性的 O(N^2) 算法**。`for-each` (O(N)) 才是正确的选择。
  * （`ArrayList` 则相反，`for(i)` 和 `for-each` 性能相近，`for(i)` 甚至可能因省去迭代器对象创建而略快）。

-----

## 第十二章：跳转语句 (`break`, `continue`) 与标签

### 12.1 `break` 和 `continue`

  * `break`：**跳出**（终止）**当前**所在的循环或 `switch` 块。
  * `continue`：**跳过**（终止）**本次**循环，立即进入**下一次**循环（`for` 循环会先执行更新语句 `i++`）。

### 12.2 “降维打击”：标签 (Label)

`break` 和 `continue` 只能控制它们所在的**最内层**循环。如果你想从一个**嵌套循环**中直接跳出到**外层**怎么办？

```java
// 需求：在一个 10x10 的坐标系中，找到第一个值为 5 的坐标 (i, j)
int targetI = -1, targetJ = -1;

outerLoop: // 1. 定义一个标签
for (int i = 0; i < 10; i++) {
    for (int j = 0; j < 10; j++) {
        if (matrix[i][j] == 5) {
            targetI = i;
            targetJ = j;
            break outerLoop; // 2. 直接跳出到 "outerLoop" 标签
        }
    }
    // 如果只用 break，会跳到这里，外层循环会继续
}
// `break outerLoop` 会直接跳到这里
```

  * **`break <label>`**：跳出到标签所在的循环**之后**。
  * **`continue <label>`**：跳过内层循环，直接进入标签所在循环的**下一次**迭代。
  * **本质**：这是 Java 提供的“结构化 `goto`”。虽然不常用，但在处理深度嵌套时是唯一优雅的解决方案。

-----

## 系列终章（下）预告

本系列从 `i++` 的字节码出发，到 `switch` 的 `lookupswitch`，再到 `for-each` 的 `Iterator` 本质，我们已经解构了 Java 几乎所有的运算符和流程控制。

在最后的**终篇**中，我们将探讨一个特殊的“流程控制”—— **`try-catch-finally`**，并揭秘 `finally` 块中 `return` 语句的终极陷阱。

```
```
