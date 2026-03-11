Java 后端开发核心面试题大全 (进阶与全栈架构版)

本文档涵盖了 Java 后端开发面试中最常被问到的高频考点，不仅包含基础理论，更深入底层源码、实际应用场景、分布式架构演进及中间件底层原理。

第一部分：Java 基础与进阶

1. 面向对象的三大特性是什么？

封装：把对象的属性私有化，同时提供可以被外界访问的属性的方法。隐藏底层逻辑，保证安全性。

继承：使用已存在的类的定义作为基础建立新类。子类可以复用父类的代码，并可以扩展新功能。

多态：程序中定义的引用变量所指向的具体类型和通过该引用变量发出的方法调用在编程时并不确定，而是在程序运行期间才确定。实现多态的三个条件：继承、重写、父类引用指向子类对象。

2. == 和 equals() 的区别？

==：对于基本数据类型，比较的是值；对于引用数据类型，比较的是内存地址（即判断两个对象是否为同一个对象）。

equals()：属于 Object 类的方法。默认情况下它的实现就是 ==。但通常诸如 String、Integer 等类都会重写此方法，用于比较对象的**内容（属性值）**是否相等。

追问：为什么重写 equals() 时必须重写 hashCode()？
如果只重写 equals，两个内容相同的对象 equals 返回 true，但它们的 hashCode 可能不同。当把它们放入散列表（如 HashMap）时，由于哈希值不同，会被放到不同的桶中，导致集合中存在“重复”对象，破坏了基于 Hash 的集合规则。

3. String、StringBuffer 和 StringBuilder 的区别？

String：不可变类（JDK 8 之前用 final char[]，JDK 9 后用 final byte[]）。每次修改都会生成新对象，频繁拼接会产生大量垃圾，性能差。

StringBuilder：可变字符序列，线程不安全，但在单线程下性能最高。

StringBuffer：可变字符序列，线程安全（核心方法加了 synchronized 关键字），性能略低于 StringBuilder。

4. Java 异常体系是怎样的？Error 和 Exception 的区别？

顶级父类是 Throwable，主要分为两大类：

Error（错误）：程序无法处理的错误，如 OutOfMemoryError、StackOverflowError。这类错误发生时，JVM 通常会选择终止线程。

Exception（异常）：程序本身可以处理的异常。

受检查异常（Checked Exception）：编译时必须处理（try-catch 或 throws），否则编译不通过，如 IOException、SQLException。

运行时异常（Runtime Exception / Unchecked Exception）：编译时不强制要求处理，运行时才可能暴露，如 NullPointerException、IndexOutOfBoundsException。

5. 反射的原理是什么？有哪些应用场景？

原理：在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性。这种动态获取信息以及动态调用对象方法的功能就是反射。核心类是 Class、Method、Field、Constructor。

应用场景：Spring IoC 的 Bean 实例化、AOP 动态代理、JDBC 加载驱动、自定义注解的解析等。

6. NIO、BIO、AIO 的区别？

BIO (Blocking I/O)：同步阻塞，服务器实现模式为一个连接一个线程。适用于连接数目比较小且固定的架构。

NIO (Non-blocking I/O)：同步非阻塞，服务器实现模式为一个请求一个线程。核心组件：Channel、Buffer、Selector。Selector 会轮询 Channel 上的事件。适用于连接数目多且连接比较短的架构（如聊天服务器、Netty 底层）。

AIO (Asynchronous I/O)：异步非阻塞，引入了异步通道的概念。发起读写操作后，直接返回，等 OS 完成读写后主动回调通知程序。适用于连接数目多且连接比较长（重操作）的架构。

第二部分：Java 集合框架

1. ArrayList 和 LinkedList 的底层实现与区别？

ArrayList：底层是动态数组。支持随机访问（时间复杂度 $O(1)$），但插入和删除非尾部元素需要移动数据（时间复杂度 $O(N)$）。

扩容机制：默认初始容量为 10（懒加载，首次 add 时初始化）。扩容为原来的 1.5 倍（oldCapacity + (oldCapacity >> 1)），并使用 Arrays.copyOf() 拷贝数据。

LinkedList：底层是双向链表。不支持高效的随机访问（$O(N)$），但插入和删除只需要修改指针（已知节点位置的前提下操作是 $O(1)$）。不连续分配内存，不存在扩容问题。

2. HashMap 的底层数据结构与扩容机制？（JDK 1.8）

数据结构：数组 + 链表 + 红黑树。

Put 流程：

计算 key 的 hash 值（高低16位异或 (h = key.hashCode()) ^ (h >>> 16)），再通过 (n - 1) & hash 计算数组下标。

如果该位置为空，直接插入。

发生冲突时，如果该节点是红黑树，则插入树中；如果是链表，尾插法插入。

插入后，如果链表长度大于等于 8 且数组长度大于等于 64，链表转化为红黑树。

扩容机制：当元素个数大于阈值（容量 $\times$ 负载因子 0.75）时，触发 resize()。容量翻倍（如 16 扩容到 32）。在 JDK 1.8 中，扩容时通过 e.hash & oldCap == 0 来判断元素保留在原位还是移动到“原位置 + 旧容量”的位置，避免了 JDK 1.7 头插法造成的死循环问题。

3. ConcurrentHashMap 的底层原理？(JDK 1.8)

JDK 1.7 使用的是 Segment 分段锁（ReentrantLock + HashEntry 数组）。

JDK 1.8 抛弃了 Segment，数据结构同 HashMap（数组 + 链表 + 红黑树）。

并发控制：使用 CAS + synchronized。当插入元素时，如果该桶（Bucket）为空，使用 CAS 进行无锁插入；如果发生了 Hash 冲突，只锁住该桶的头节点（使用 synchronized）。大大降低了锁的粒度。

协助扩容：当一个线程 put 时发现正在扩容（遇到 ForwardingNode），它会协助进行数据的搬迁，提高扩容效率。

4. LinkedHashMap 和 TreeMap 的应用场景？

LinkedHashMap：继承自 HashMap，内部额外维护了一个双向链表来记录插入顺序或者访问顺序。常用于实现 LRU（最近最少使用）缓存（重写 removeEldestEntry 方法）。

TreeMap：基于红黑树实现，可以根据 Key 按照自然顺序或者自定义 Comparator 排序。时间复杂度 $O(\log N)$。

第三部分：并发编程 (JUC)

1. 线程的生命周期与状态转换？

Java 线程有 6 种状态（定义在 Thread.State 中）：

NEW（新建）：创建了 Thread 对象但未调用 start()。

RUNNABLE（运行中）：包含了操作系统的 Ready 和 Running 状态。

BLOCKED（阻塞）：等待获取排他锁（如进入 synchronized 块）。

WAITING（无限期等待）：调用了不带超时的 wait()、join() 或 LockSupport.park()，需被其他线程唤醒（notify()/notifyAll()）。

TIMED_WAITING（限期等待）：调用了带超时的 sleep(long)、wait(long) 等。

TERMINATED（终止）：线程执行完毕或异常退出。

2. sleep() 和 wait() 的区别？

所属类：sleep() 属于 Thread 类；wait() 属于 Object 类。

释放锁：sleep() 不释放锁，抱着锁睡觉；wait() 会释放锁，让出 CPU 资源。

唤醒方式：sleep() 时间到了自动苏醒；wait() 必须在 synchronized 同步代码块中使用，需要等待 notify()/notifyAll() 唤醒（或设置超时时间）。

3. synchronized 的底层原理与锁升级？

原理：基于 JVM 内部的 Monitor（管程）实现。编译后会在同步块前后插入 monitorenter 和 monitorexit 字节码指令。

锁升级过程（JDK 1.6+ 优化）：

无锁：对象刚创建。

偏向锁：只有同一个线程访问，将线程 ID 记录在对象头的 Mark Word 中，无竞争，效率极高。

轻量级锁：发生轻微竞争（其他线程试图获取锁）。未获取到的线程通过 自旋（CAS） 等待，不阻塞。

重量级锁：自旋超过一定次数或并发量大，升级为重量级锁，向操作系统申请互斥锁，导致线程阻塞和上下文切换。

4. volatile 关键字的作用？

保证内存可见性：每次读取 volatile 变量都从主存读，写入时立即刷回主存，使得其他线程能立刻看到最新值（利用 CPU 的 MESI 缓存一致性协议）。

禁止指令重排序：通过插入**内存屏障（Memory Barrier）**防止编译器和 CPU 对指令进行重排（经典场景：单例模式的双重检查锁 DCL 中，防止对象半初始化）。

注意：volatile 不能保证原子性（如 i++ 依然非线程安全）。

5. ThreadLocal 的原理及内存泄漏问题？

原理：每个 Thread 对象内部维护了一个 ThreadLocalMap。调用 set() 时，以 ThreadLocal 实例为 key，存入当前线程的 Map 中，实现线程间数据隔离。

内存泄漏：ThreadLocalMap 的 Entry 中，Key（即 ThreadLocal）是弱引用，Value 是强引用。如果 ThreadLocal 外部没有强引用被 GC 回收（Key 变 null），而线程一直在运行（如线程池中的复用线程），就会形成强引用链 Thread -> ThreadLocalMap -> Entry -> Value，导致 Value 无法回收。

解决：使用完后务必在 finally 块中显式调用 threadLocal.remove()。

6. AQS (AbstractQueuedSynchronizer) 原理是什么？

核心思想：如果被请求的共享资源空闲，就把当前请求线程设置为有效的工作线程，并将共享资源设置为锁定状态；如果共享资源被占用，就需要一定的阻塞等待唤醒机制，AQS 用一个**虚拟双向队列（CLH 队列）**来实现。

状态标识：使用一个 volatile int state 变量表示同步状态。通过内置的 FIFO 队列来完成获取资源线程的排队工作。通过 CAS 操作更新 state。

实现类：ReentrantLock、CountDownLatch、Semaphore 等都是基于 AQS 实现的。

7. 线程池的核心参数与工作流程？

7大核心参数（ThreadPoolExecutor）：
corePoolSize（核心线程数）、maximumPoolSize（最大线程数）、keepAliveTime（空闲线程存活时间）、unit（时间单位）、workQueue（任务阻塞队列）、threadFactory（线程工厂）、handler（拒绝策略）。
工作流程：

提交任务，若当前线程数 < corePoolSize，立即创建新核心线程执行。

若运行的线程数 $\ge$ corePoolSize，将任务加入 workQueue 等待。

若队列已满，且线程数 < maximumPoolSize，创建非核心线程执行该任务。

若队列满且线程数已达 maximumPoolSize，触发拒绝策略（默认抛出异常、由调用线程执行、静默丢弃、丢弃最老的任务）。

第四部分：JVM 虚拟机

1. JVM 内存模型（运行时数据区）有哪些？

线程私有：

程序计数器：指示当前线程执行的代码行号，唯一不会报 OOM 的区域。

虚拟机栈：方法执行的内存模型，包含栈帧（局部变量表、操作数栈、动态链接、方法出口）。递归过深报 StackOverflowError。

本地方法栈：执行 Native 方法。

线程共享：

堆（Heap）：存放对象实例和数组，GC 的主要区域。报 OutOfMemoryError。

方法区（元空间 Metaspace）：存放类信息、常量池、静态变量、JIT 编译代码。JDK 1.8 从永久代移到本地内存的元空间。

2. 垃圾回收（GC）如何判断对象已死？

引用计数法：存在循环引用问题，Java 未采用。

可达性分析算法（GC Roots Tracing）：从一系列 GC Roots 对象作为起点向下搜索。如果一个对象到 GC Roots 没有任何引用链相连，说明其可被回收。

GC Roots 包括：虚拟机栈/本地方法栈引用的对象、方法区静态属性/常量引用的对象。

3. 常用的垃圾回收算法？

标记-清除（Mark-Sweep）：产生内存碎片。

复制算法（Copying）：内存分两半，每次用一半。存活对象拷贝到另一半，清空当前区。无碎片，利用率低（主要用于新生代）。

标记-整理（Mark-Compact）：存活对象向一端移动，清理边界外内存。无碎片（主要用于老年代）。

4. CMS 和 G1 垃圾收集器的区别？

CMS (Concurrent Mark Sweep)：以获取最短回收停顿时间为目标。分为初始标记（STW）、并发标记、重新标记（STW）、并发清除。

缺点：对 CPU 敏感，无法处理浮动垃圾，基于标记-清除算法会产生大量空间碎片。

G1 (Garbage-First)：面向服务端的收集器，目标是可预测的停顿时间模型。将堆划分为多个大小相等的独立区域（Region），新生代和老年代不再物理隔离。

G1 跟踪各个 Region 中垃圾堆积的“价值”大小，在后台维护一个优先列表，优先回收价值最大的 Region。整体基于“标记-整理”，局部基于“复制”，不会产生空间碎片。

5. 类加载过程与双亲委派模型？

过程：加载（Load） -> 链接（验证、准备、解析） -> 初始化（Initialize）。

双亲委派模型：当一个类加载器收到加载请求时，先将其委派给父加载器处理，逐级向上（App -> Ext -> Bootstrap）。如果父加载器无法加载，子加载器才尝试自己加载。

好处：避免类的重复加载，保护 Java 核心类库的安全（防止用户自定义 java.lang.String 替换系统类）。

破坏双亲委派：Tomcat 隔离不同 Web 应用、JDBC SPI 机制、OSGi 等。

第五部分：Spring 与 Spring Boot

1. 谈谈你对 Spring IoC 和 AOP 的理解？

IoC（控制反转 / 依赖注入 DI）：将对象的创建和管理权交给了 Spring 容器，降低了类之间的耦合度。底层通过反射机制 + 工厂模式读取 XML 或注解来实例化和装配 Bean。

AOP（面向切面编程）：在不修改原有业务代码的情况下，抽离出公共逻辑（如日志、事务控制、权限校验），将其增强到目标方法上。底层实现基于动态代理：

目标对象实现接口：使用 JDK 动态代理（基于反射）。

目标对象没有实现接口：使用 CGLIB（基于 ASM 字节码生成子类覆盖方法）。

2. Spring Bean 的生命周期是怎样的？

简单总结为 4 大阶段（包含众多扩展点）：

实例化 (Instantiation)：执行构造方法，创建 Bean 对象（在堆中开辟空间）。

属性赋值 (Populate)：依赖注入（DI），自动装配（如 @Autowired 属性）。

初始化 (Initialization)：执行 Aware 接口方法，执行 BeanPostProcessor 前置处理，执行 @PostConstruct 方法，执行 InitializingBean，执行定制的 init-method，最后执行 BeanPostProcessor 后置处理（AOP 代理通常在此处生成）。

销毁 (Destruction)：容器关闭时，执行 @PreDestroy，执行 DisposableBean，执行定制的 destroy-method。

3. Spring 怎么解决循环依赖？

Spring 通过 三级缓存 解决单例（Singleton）模式下的 setter 循环依赖问题：

一级缓存（singletonObjects）：存放完全初始化好的成品 Bean。

二级缓存（earlySingletonObjects）：存放实例化完成、但未注入属性的半成品 Bean。

三级缓存（singletonFactories）：存放 Bean 工厂对象（ObjectFactory），用于在需要 AOP 代理时提前生成并暴露代理对象的引用。

注意：构造器注入造成的循环依赖 Spring 无法解决，推荐使用 @Lazy 延迟加载或重构代码。

4. Spring 事务传播行为与失效场景？

传播行为（Propagation）：常用的是 REQUIRED（默认，没有事务就新建，有就加入）、REQUIRES_NEW（新建独立事务，外层事务挂起）、NESTED（嵌套事务）。

事务失效常见场景：

方法未被 public 修饰（Spring AOP 代理规则不支持）。

同一个类内部的非事务方法调用了事务方法（绕过了代理对象，直接通过 this 调用）。

异常被 try-catch 吃掉，没有向上抛出，Spring 无法感知。

抛出的异常类型不是 RuntimeException 或 Error（默认只回滚这类异常，除非显式配置 rollbackFor = Exception.class）。

MySQL 引擎使用了 MyISAM（不支持事务）。

5. Spring Boot 自动装配原理？

核心是 @SpringBootApplication 注解，它由三个核心注解组成：

@SpringBootConfiguration：标明是一个配置类。

@ComponentScan：包扫描。

@EnableAutoConfiguration：这是核心！它利用 @Import(AutoConfigurationImportSelector.class)。在启动时扫描 classpath 下所有的 META-INF/spring.factories (Spring Boot 3.x 后改为 META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports 文件)。

然后结合 @Conditional 系列条件注解（如 @ConditionalOnClass, @ConditionalOnMissingBean），根据当前环境 classpath 中是否存在相关依赖来决定是否向 IoC 容器中注入 Bean，实现了“开箱即用”。

第六部分：MySQL 数据库

1. 事务的四大特性（ACID）是怎么保证的？

原子性 (Atomicity)：通过 Undo Log（回滚日志） 保证。事务失败时，利用 Undo Log 逆向操作回滚。

持久性 (Durability)：通过 Redo Log（重做日志） 保证。WAL 机制（Write-Ahead Logging），修改数据时先写 Redo Log，宕机重启后可利用 Redo Log 恢复。

隔离性 (Isolation)：通过 锁机制（Record Lock/Gap Lock） 和 MVCC（多版本并发控制） 共同保证。

一致性 (Consistency)：是目的，由原子性、隔离性、持久性以及业务代码的合法性共同保证。

2. 事务隔离级别与解决的问题？MySQL 默认是哪个？

读未提交 (Read Uncommitted)：引发脏读（读到其他事务未提交的数据）。

读已提交 (Read Committed / RC)：解决脏读，但存在不可重复读（同一事务内多次读取同一数据结果不同）。Oracle 默认。

可重复读 (Repeatable Read / RR)：MySQL InnoDB 默认级别。解决不可重复读。在标准 SQL 中存在幻读问题，但 InnoDB 通过 MVCC 快照读 和 Gap Lock（间隙锁）+ Next-Key Lock 当前读，很大程度上避免了幻读。

串行化 (Serializable)：强制事务串行执行，完全解决所有并发问题，但性能极低。

3. 详细解释一下 MVCC（多版本并发控制）原理？

MVCC 主要在 RC 和 RR 级别下工作，实现“读写不阻塞”。核心机制：隐藏字段 + Undo Log 版本链 + Read View（读视图）。

隐藏字段：每行数据都有 trx_id（最后一次修改该行的事务 ID）和 roll_pointer（回滚指针，指向 Undo Log）。

Undo 版本链：每次更新都会将旧数据放到 Undo Log 中，通过 roll_pointer 连成链表。

Read View：事务执行查询时生成的“快照视图”（记录当前系统中活跃的未提交事务 ID 列表）。

可见性判断：

在 RC 级别下，每次 SELECT 都会生成全新的 Read View。

在 RR 级别下，只有第一次 SELECT 时生成 Read View，之后复用。根据当前行的 trx_id 和 Read View 的规则（对比 min_trx_id 和 max_trx_id），决定当前事务能看到版本链中的哪个历史快照。

4. 为什么 MySQL 索引底层用 B+ 树，不用 B 树或红黑树？

对比红黑树：树高太高，随着数据量增大，磁盘 IO 次数呈对数级增加。

对比 B 树：

B 树的非叶子节点也存储数据（data），而 B+ 树的非叶子节点只存储索引列。因此 B+ 树一个磁盘页（16KB）能容纳更多索引，树更“矮胖”（通常只有 3 层就能存千万级数据），磁盘 IO 更少。

B+ 树的叶子节点用双向链表相连，极大地提升了**范围查询（Range Query）**的效率，而 B 树需要进行树的中序遍历。

5. 什么是回表？什么是覆盖索引？什么是索引下推？

聚簇索引（通常主键）：叶子节点存放的是完整的一行数据。

二级索引（非聚簇索引）：叶子节点存放的是主键的值。

回表：走二级索引查询时，先查出主键，再根据主键去聚簇索引树查出所有字段的过程。性能损耗较大。

覆盖索引：如果你 SELECT 的字段刚好都在二级索引树里（如建立了联合索引 (a,b)，查询 SELECT a,b FROM t WHERE a = 1），可以直接从二级索引叶子节点拿到结果，不需要回表，极大提升性能。

索引下推 (ICP)：MySQL 5.6 引入。在执行 SELECT * FROM t WHERE name='张三' AND age=20（联合索引 name, age）时，如果根据 name 匹配后，在回表前，存储引擎层会先根据 age=20 过滤不符合条件的索引记录，减少回表的次数。

6. SQL 优化的常见思路？(Explain怎么看)

使用 EXPLAIN 查看执行计划，重点关注几个字段：

type：连接类型。从好到坏：system > const > eq_ref > ref > range > index > ALL。如果是 ALL（全表扫描）或 index，必须优化。

possible_keys / key：可能用到的索引 / 实际用到的索引。

Extra：

Using index：好，使用了覆盖索引。

Using index condition：使用了索引下推。

Using filesort / Using temporary：差，发生了外部文件排序或使用了临时表，必须想办法优化（通过建立合适的联合索引来利用树的有序性）。

常见优化手段：最左前缀法则匹配联合索引、避免 SELECT *、避免索引列参与计算/函数、LIKE 查询避免前导 %。

第七部分：Redis 缓存与中间件

1. Redis 为什么这么快？

纯内存操作：数据存放在内存中，读写速度极快。

核心单线程架构：避免了多线程频繁上下文切换带来的开销，也避免了各种并发锁的竞争问题（Redis 6.0 后引入多线程仅处理网络 IO 处理，命令执行仍是单线程）。

IO 多路复用模型：基于 epoll 机制，一个线程同时监听多个 Socket，实现非阻塞的事件驱动网络通信。

高效的数据结构：如 SDS（动态字符串，防溢出获取长度O(1)）、SkipList（跳表，实现 Zset 的高效范围查询）、ZipList/QuickList 等极限压榨内存空间。

2. Redis 持久化机制 RDB 和 AOF？

RDB（快照）：在指定时间间隔把内存中的数据生成快照保存到磁盘（dump.rdb）。

优点：文件紧凑，全量恢复速度极快。

缺点：会丢失最后一次快照后的数据；bgsave 开启子进程 fork() 时如果内存大，会短暂阻塞主线程。

AOF（追加日志）：将每次执行的写命令追加记录到日志文件（appendonly.aof）中。

优点：数据安全性高，最多丢失 1 秒数据（默认 everysec 策略）。

缺点：文件体积大，命令重放恢复速度慢。Redis 提供 AOF 重写（bgrewriteaof）来压缩体积。

生产建议：Redis 4.0 后开启 RDB + AOF 混合持久化。

3. 什么是缓存雪崩、穿透、击穿？解决方案？

缓存穿透：查询数据库和缓存中都不存在的数据，请求直接打到 DB（常被恶意攻击）。

解决：1. 缓存空值/空对象（设短 TTL）；2. 布隆过滤器（Bloom Filter） 前置拦截。

缓存击穿：某个高热点 Key 刚好到期，大量并发请求同时瞬间涌入查 DB重建缓存。

解决：1. 热点数据不设置过期时间（逻辑过期）；2. 使用互斥锁（如 setnx），让拿到锁的唯一线程去查 DB，其他线程休眠等待。

缓存雪崩：大量 Key 在同一时间失效，或 Redis 集群大面积宕机。

解决：1. 随机化 Key 的 TTL，避免集体过期；2. Redis 高可用（主从+哨兵或 Cluster）；3. 依赖服务熔断/降级机制保护底层 DB。

4. Redis 分布式锁怎么实现？Redisson 解决了什么问题？

基础命令：SET resource_name my_random_id NX PX 30000（如果不存在则设置，并加 30 秒过期时间，防止死锁）。释放时必须用 Lua 脚本检查 my_random_id 匹配后才 DEL（防止误删别人的锁）。

业务超时问题：如果业务执行耗时 40 秒，而锁 30 秒自动过期了，会导致并发安全问题。

Redisson 的 WatchDog（看门狗）：底层利用 Netty 的时间轮机制。只要抢到锁的线程业务还没执行完，看门狗会每隔 internalLockLeaseTime / 3（默认 10 秒）去执行一次 Lua 脚本给锁自动续期。彻底解决业务未完而锁超时的痛点。

5. Redis 的双写一致性如何保证？

旁路缓存模式 (Cache Aside)：先更新数据库，再删除缓存。

为什么不先删缓存？ 如果 A 线程删缓存 -> B 线程读缓存未命中去读 DB 拿到旧值 -> A 线程更新 DB -> B 线程把旧值写回缓存。导致长久的脏数据。

先更新DB后删缓存的极小概率问题：如果删缓存失败怎么办？引入 MQ（消息队列）重试机制 或通过 Canal 监听 MySQL Binlog 异步精准删除缓存，实现最终一致性。

第八部分：计算机网络

1. TCP 三次握手和四次挥手的详细过程？

三次握手（建立连接）：

客户端 -> 服务端：SYN=1, seq=x。客户端进入 SYN_SENT。

服务端 -> 客户端：SYN=1, ACK=1, ack=x+1, seq=y。服务端进入 SYN_RCVD。

客户端 -> 服务端：ACK=1, ack=y+1, seq=x+1。双方进入 ESTABLISHED。

为什么不是两次？ 为了防止已失效的连接请求报文段突然又传送到了服务端，产生脏连接浪费资源。

四次挥手（断开连接）：

主动方 -> 被动方：FIN=1, seq=u。主动方进入 FIN-WAIT-1。

被动方 -> 主动方：ACK=1, ack=u+1。被动方进入 CLOSE-WAIT。主动方收到后进入 FIN-WAIT-2。（此时属于半关闭状态，被动方还可以发数据）。

被动方 -> 主动方：发完数据后，FIN=1, ACK=1, seq=w, ack=u+1。被动方进入 LAST-ACK。

主动方 -> 被动方：ACK=1, ack=w+1。主动方进入 TIME-WAIT。等待 2MSL 后彻底关闭。

为什么要有 TIME-WAIT (2MSL)？ 1. 保证被动方能收到最后一个 ACK（若没收到，被动方会重发 FIN）；2. 防止“已失效的连接请求报文段”出现在本连接中。

2. HTTP 和 HTTPS 的区别？HTTPS 工作流程？

区别：HTTP 是明文传输（端口 80）；HTTPS 在 HTTP 下层加入了 SSL/TLS 协议（端口 443），提供数据加密、完整性校验和身份认证。

HTTPS 流程（RSA 为例）：

客户端发起 HTTPS 请求，连接服务器 443 端口。

服务器发送数字证书（包含公钥、CA 签名等）给客户端。

客户端验证证书合法性，生成一个随机的对称加密密钥（Session Key）。

客户端用服务器的公钥对 Session Key 进行非对称加密，发给服务器。

服务器用自己的私钥解密出 Session Key。

之后双方通过这个 Session Key 进行对称加密的快速业务数据传输。

第九部分：消息队列 (MQ)

1. 为什么要使用 MQ？(三大核心场景)

解耦：将强依赖的 RPC 调用改为基于消息的发布订阅，上下游系统独立演进。

异步：将主干流程中非核心、耗时的操作（如发短信、记录日志）抽取到异步消息中执行，缩短接口响应时间。

削峰填谷：面对突发的高并发流量，请求先积压在 MQ 中，下游消费者根据自己的处理能力匀速拉取，保护下游不被冲垮。

2. 如何保证消息不丢失？(以 RabbitMQ/Kafka 为例)

贯穿消息的三个生命周期：

生产者到 MQ：

RabbitMQ：开启 publisher confirm 确认机制。

Kafka：设置 acks=all，必须所有副本同步成功才返回 ACK。失败则利用本地重试。

MQ 自身丢失：

开启消息和队列的持久化（刷盘）。Kafka 可以配置多副本（Replica）机制，配合 Leader 选举保证集群高可用。

MQ 到消费者：

关闭自动 ACK，改为业务处理（如扣减库存、落库）完全成功后，再调用 API 手动 ACK。如果在处理过程中抛出异常或系统宕机，消息会重新投递。

3. 如何保证消费的幂等性？（防止消息重复消费）

消息重复是不可避免的（网络抖动重发），必须在消费端做业务幂等。

数据库唯一索引：如流水表使用订单号防重插入。

Redis 防重 Token/分布式锁：消费前用 setnx(messageId) 占坑，处理完设为已处理状态。

状态机乐观锁：UPDATE table SET status = '已支付' WHERE id = 1 AND status = '待支付'。

4. 如何保证消息的顺序性？

RabbitMQ：拆分多个 Queue，每个 Queue 只对应一个 Consumer 线程；或者在单 Consumer 内部按内存队列（如 Hash 取模）进行排队消费。

Kafka：将需要保证顺序的消息（如同一个订单的创建、支付、发货）发送到同一个 Topic 的同一个 Partition。Partition 内部是有序的。消费者端也是每个 Partition 被一个固定的线程按序拉取。

第十部分：分布式系统与微服务

1. CAP 定理与 BASE 理论？

CAP：一致性 (Consistency)、可用性 (Availability)、分区容错性 (Partition tolerance)。在一个分布式系统中，最多只能同时满足其中两项。因为网络分区 P 是常态，通常需要在 CP（强一致，如 Zookeeper）和 AP（高可用，如 Eureka, Nacos）之间做权衡。

BASE：Basically Available（基本可用）、Soft state（软状态）、Eventually consistent（最终一致性）。是针对 AP 系统的一种妥协，允许短暂的数据不一致，依靠补偿机制保证业务的最终一致。

2. 分布式事务的常见解决方案？

2PC（两阶段提交） / Seata AT 模式：

第一阶段：TM（事务管理器）通知各个 RM（资源管理器）执行本地 SQL 并写回滚日志，但不提交。

第二阶段：如果所有 RM 都成功，TM 通知全局 Commit；如果有失败，通知全局 Rollback（利用回滚日志撤销）。强一致性，但持锁时间长，性能差。

TCC (Try-Confirm-Cancel)：业务层面的两阶段。Try 预留资源，Confirm 确认执行，Cancel 释放资源。适用于资金转账等严谨场景。

可靠消息最终一致性（MQ 本地消息表）：将分布式事务拆分为本地事务和发送异步 MQ 消息。适用于非实时性要求极高的场景（如下单后发积分）。

3. 分布式 ID 的生成方案？

UUID：太长，无序，作为 MySQL 聚簇索引会导致页分裂和极高的插入性能损耗。

数据库自增 / 步长分段：利用 MySQL 的 auto_increment，依靠批量号段分配（如美团 Leaf）提升性能。

雪花算法 (Snowflake)：64 bit 整型。包含 41 位时间戳 + 10 位机器 ID + 12 位序列号。性能极高、趋势递增、不依赖数据库。需要解决时钟回拨问题。

4. 微服务核心组件都有哪些（Spring Cloud Alibaba）？

注册中心：Nacos。负责服务的注册与发现（AP/CP 可切换）。

配置中心：Nacos。支持配置的动态刷新。

RPC / 远程调用：OpenFeign / Dubbo。底层基于 Ribbon 负载均衡。

服务网关：Spring Cloud Gateway。负责统一鉴权、路由转发、限流。

熔断限流：Sentinel。提供基于并发线程数或 QPS 的限流，以及基于慢调用比例和异常比例的熔断降级策略，保护主业务不被雪崩拖垮。
