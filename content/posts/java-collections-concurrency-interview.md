---
title: "Java 集合与并发面试 16 问：从 HashMap 扩容到 CAS 与 ABA"
date: 2026-11-04T09:00:00+08:00
draft: false
description: "整理 Java 集合与并发面试常见问题，讲清 HashMap 两倍扩容、ConcurrentHashMap、synchronized 锁升级、volatile、CAS 与 ABA，并结合仓位计算场景讨论线程池和任务积压。"
categories: ["Android"]
tags: ["Java", "Android", "集合", "并发", "面试"]
---

上一篇整理了 [Kotlin 与协程面试题](https://visiongem.github.io/Blog/posts/kotlin-coroutines-interview/)，这一篇接着聊 Java 集合与并发。

这部分面试经常从一个熟悉的名字开始：HashMap、synchronized、volatile。你回答完定义，面试官接着问：HashMap 为什么按两倍扩容？给 Map 加上 volatile 就线程安全了吗？CAS 发现值没变，能不能证明中间没有人动过它？

下面整理了 16 道常见问题，每题包含原理、示例或使用场景，以及一段可以用自己的话复述的答案。

**版本约定：** HashMap、ConcurrentHashMap 的源码细节以 **OpenJDK 8** 为主要讲解对象；Java 内存模型与 API 语义另附官方资料。HotSpot 的锁实现会随 JDK 版本变化，Android 使用的 ART 也有自己的实现，不能把某个 JVM 的对象头和锁升级细节当成 Java 语言规范。示例省略常规 import。

![Java 集合与并发面试 16 问封面](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/java_interview_0.png)

## 1. ArrayList 和 LinkedList 有什么区别？链表增删一定更快吗？

`ArrayList` 用数组保存元素引用，`LinkedList` 使用双向链表。区别要落到具体操作上：

| 操作 | ArrayList | LinkedList |
|---|---|---|
| 按下标读取 | O(1) | O(n) |
| 尾部追加 | 均摊 O(1) | O(1) |
| 按下标增删 | O(n) | O(n) |
| 已定位增删 | 移动元素 | 修改链接 |

表中的增删以列表中间位置为例。`list.remove(index)` 在链表中也要先找到节点。只有已经通过迭代器走到目标位置等情况下，才能省掉这段查找。参见 [ArrayList API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/ArrayList.html) 和 [LinkedList API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/LinkedList.html)。

ArrayList 的扩容会分配新数组并复制引用；LinkedList 则要为元素维护节点和前后指针。对于仓位列表这种遍历多、按位置访问多的场景，我会先用 ArrayList，再根据测量结果调整。

如果提前知道大约有 1,500 个元素，可以指定初始容量，减少扩容复制。但初始容量不等于元素数量，`new ArrayList<>(1500).size()` 仍然是 0。

**面试时可以这样说：**

> ArrayList 适合随机访问和连续遍历，尾部追加是均摊常数时间。LinkedList 的增删优势需要已经定位节点，按下标操作仍然要遍历，不能直接说“增删多就用链表”。

## 2. HashMap 的底层结构是什么？put 和 get 怎么执行？

OpenJDK 8 的 HashMap 使用**数组、链表和红黑树**。数组的每个位置称为一个桶，发生碰撞时，多个节点会落到同一个桶。

定位桶之前，先对 key 的哈希值做扰动。对于非 null 的 key，可以理解成：

```java
int h = key.hashCode();
int hash = h ^ (h >>> 16);
int index = (capacity - 1) & hash;
```

扰动让高位信息也参与低位计算；数组容量是 2 的幂，定位时使用按位与。null key 的内部 hash 按 0 处理。

一次 `put` 的主要过程是：

1. 数组尚未初始化时，先初始化数组。
2. 定位桶。空桶直接放入节点。
3. 非空桶中查找相同 key：先比较保存的 hash，再比较引用或 `equals()`；找到就替换 value。
4. 非空桶中没找到相同 key，则追加新节点，并按条件检查树化。

无论新节点放进空桶还是非空桶，只要新增了映射，最后都会增加 size，并检查是否超过扩容阈值。

`get` 使用相同的定位方式，再到对应桶里查找。结构和流程可对照 [OpenJDK 8 HashMap 源码](https://raw.githubusercontent.com/openjdk/jdk8u/master/jdk/src/share/classes/java/util/HashMap.java)。

HashMap 允许一个 null key 和多个 null value，不保证遍历顺序。哈希分布合理、key 的哈希与比较操作成本固定时，查找平均为 O(1)，连续插入的均摊成本为 O(1)；单次插入若触发扩容，就需要迁移节点，不能说“任何情况下都是 O(1)”。参见 [HashMap API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/HashMap.html)。

**面试时可以这样说：**

> HashMap 先用扰动后的 hash 定位数组桶，再在链表或红黑树中查找 key。put 遇到相同 key 会替换 value，新增映射才会增加 size。它不保证顺序，也不提供并发读写保护。

## 3. HashMap 为什么按两倍扩容？扩容时会重新调用 hashCode 吗？

两倍扩容和 **2 的幂容量、位运算定位、桶拆分**配合在一起。

假设容量从 16 变成 32，定位掩码会从二进制 `01111` 变成 `11111`，只增加了一个参与计算的位。原来落在桶 `j` 的节点，新位置只有两种可能：

```text
(hash & oldCapacity) == 0
    → 留在原位置 j

(hash & oldCapacity) != 0
    → 移到 j + oldCapacity
```

拿两个已经扰动过的 hash 举例：

| hash | 容量 16 时 | 容量 32 时 |
|---|---|---|
| 5，即 00101 | 桶 5 | 桶 5 |
| 21，即 10101 | 桶 5 | 桶 21 |

因此扩容时可以把一个旧桶拆成两组，不需要对每个 key 重新调用 `hashCode()`，会复用节点中保存的 hash。对应实现见 [HashMap 的 resize 方法](https://github.com/openjdk/jdk8u/blob/master/jdk/src/share/classes/java/util/HashMap.java)。

倍增也能避免容量每次只增加一点所带来的频繁迁移，但扩容本身仍然要分配数组、遍历并迁移节点，有时间和内存成本。

默认负载因子为 0.75。通常在新增映射后，size 超过容量与负载因子的乘积时触发扩容。例如容量 16、阈值 12，加入第 13 个不同的 key 时会超过阈值。碰撞过多时，还有下面一题提到的提前扩容路径。参见 [HashMap 容量与负载因子说明](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/HashMap.html)。

预计保存 1,500 个映射时，按默认负载因子估算需要至少 2,000 个桶，再向上取 2 的幂，可以使用 `new HashMap<>(2048)`。数组初始化后的扩容阈值为 1,536，逐条 put 这 1,500 个不同的 key 不会超过阈值；容量也已超过 64，长链表会进入树化处理，不会因为容量不足而再次扩容。预分配能减少扩容，但不能消除哈希碰撞带来的查找成本。

**面试时可以这样说：**

> 两倍扩容能保持容量为 2 的幂。新旧掩码只差一位，所以节点只会留在原桶，或移动到原下标加旧容量的位置。迁移复用已有 hash，仍然需要遍历节点，预估容量可以减少这部分开销。

![HashMap 从容量 16 扩到 32 时的桶拆分示意](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/java_interview_1.png)

## 4. 链表有 8 个节点就一定树化吗？为什么又有 6 和 64？

OpenJDK 8 中有三个常见常量：

- `TREEIFY_THRESHOLD = 8`：树化相关阈值。
- `UNTREEIFY_THRESHOLD = 6`：扩容拆分树桶时，判断是否退回链表的阈值。
- `MIN_TREEIFY_CAPACITY = 64`：允许树化的最小数组容量。

**只背“8 变树、6 变链表”会漏掉触发条件。**

以普通 `put` 的链表追加路径为例：已有 8 个节点，再插入第 9 个不同 key 时，会调用树化检查。如果数组容量小于 64，这次会优先扩容；达到容量条件后才转成树。其他插入 API 的判断位置要看各自实现。

扩容拆分树桶时，某一侧节点数不超过 6，会退回链表。普通删除的退化判断还涉及树形结构，不能把 6 当成所有路径统一执行的计数规则。具体条件见 [OpenJDK 8 的 putVal、treeifyBin 和 TreeNode 实现](https://raw.githubusercontent.com/openjdk/jdk8u/master/jdk/src/share/classes/java/util/HashMap.java)。

这些阈值体现了空间与查找成本的折中：树节点比普通节点占用更多空间，哈希分布良好时，长链表又很少出现，因此短链表没必要一开始就树化。8 和 6 是实现中的取舍，不是数学上唯一正确的数字；64 则让小容量时优先通过扩容分散碰撞。

红黑树也有适用条件：hash 相同、又无法通过比较关系确定方向的 key，查找可能需要探索多个分支，因此不能保证任意 key 下都严格为 O(log n)。

**面试时可以这样说：**

> 树化既看桶内节点数量，也看数组容量。容量不足 64 时优先扩容；普通 put 路径是在已有 8 个节点后再追加时触发检查。6 主要用于扩容拆分后的退化判断，回答时要说明版本和执行路径。

## 5. equals 和 hashCode 为什么要一起重写？key 能修改吗？

两者的约定是：**equals 相等的对象，hashCode 必须相等；hashCode 相等的对象，equals 可以不相等。** 参见 [Object 的 hashCode 与 equals 契约](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/Object.html#hashCode())。

如果只重写 equals，两个业务上相等的对象可能因为 hash 不同落到不同桶，导致查不到已有映射，或保存出本来不应重复的 key。

另一个坑是把参与 equals、hashCode 的可变字段当作 key 的一部分：

```java
final class PositionKey {
    int id;

    PositionKey(int id) {
        this.id = id;
    }

    @Override
    public int hashCode() {
        return id;
    }

    @Override
    public boolean equals(Object other) {
        return other instanceof PositionKey
                && id == ((PositionKey) other).id;
    }
}
```

```java
PositionKey key = new PositionKey(1);
Map<PositionKey, String> map = new HashMap<>();
map.put(key, "仓位 A");

key.id = 2;
System.out.println(map.get(key)); // null
```

插入时保存的是旧 hash，查询时却使用修改后的 hash。即使还是同一个对象，也可能查不到。业务中可以使用不可变的仓位 ID，或让 key 中参与比较的字段保持不变。

**面试时可以这样说：**

> hashCode 决定候选位置，equals 判断逻辑相等，二者必须遵守同一个业务定义。作为 key 保存期间，参与比较和哈希计算的字段应当稳定，否则可能出现对象还在 Map 中却查不到的情况。

## 6. HashMap 为什么线程不安全？换成线程安全集合就够了吗？

多个线程没有同步地读写 HashMap，可能出现更新丢失、读到不一致数据等问题。不要把原因只归结为某个旧版本的扩容成环：**即使没有触发扩容，无同步的并发更新也不受支持。**

迭代时可能出现的 `ConcurrentModificationException` 是尽力检测错误的机制，不能作为并发正确性的保障；同一个线程在增强 for 循环中直接修改集合，也可能触发它。参见 [HashMap 的同步与 fail-fast 说明](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/HashMap.html)。

需要多线程访问时，可以按需求选择：

| 需求 | 可考虑的方案 |
|---|---|
| 单线程维护 | 普通集合与线程约束 |
| 按 key 并发读写 | 并发 Map（见第 7 题） |
| 多步整体互斥 | 同一把锁保护全流程 |
| 遍历远多于修改 | 写时复制列表 |

`Collections.synchronizedMap` 用同步包装器保护访问，但遍历集合视图时仍需要在返回的 Map 上加锁，且所有访问都应经过包装器。参见 [synchronizedMap 的使用要求](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/Collections.html#synchronizedMap(java.util.Map))。

`CopyOnWriteArrayList` 的修改通常涉及复制底层数组，迭代器读取创建时的数组快照。它可以用于变动少的监听器列表；仓位价格高频更新时，要考虑复制和内存开销。快照也只保存元素引用，不会把可变元素深拷贝成不可变对象。参见 [CopyOnWriteArrayList API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/CopyOnWriteArrayList.html)。

**面试时可以这样说：**

> 线程安全集合保护的是它承诺的操作范围。多个调用组成的业务流程，以及集合中可变对象的字段，仍然需要单独设计同步。读多写少、按 key 更新、整体事务，对应的方案会不同。

## 7. ConcurrentHashMap 怎么保证并发安全？还用分段锁吗？

面试中常见的 Segment 分段锁，是 JDK 7 时代的实现。**讲 OpenJDK 8 时，应当讲 CAS、桶级同步和并发扩容。**

以普通 put 路径为例：空桶尝试用 CAS 安装节点；非空桶的更新涉及 synchronized 保护，并在加锁后重新检查桶头；遇到迁移标记时，可以协助扩容。get 通常不需要获取更新时使用的锁。参见 [OpenJDK 8 ConcurrentHashMap 源码](https://raw.githubusercontent.com/openjdk/jdk8u/master/jdk/src/share/classes/java/util/concurrent/ConcurrentHashMap.java)。

不过，即使多个线程共享同一个 ConcurrentHashMap，下面这种计数仍然可能丢更新：

```java
ConcurrentHashMap<String, Integer> counts =
        new ConcurrentHashMap<>();

// 两个线程可能同时读到 0，再分别写入 1
counts.put("BTC", counts.getOrDefault("BTC", 0) + 1);
```

可以改成该 key 上的原子合并：

```java
counts.merge("BTC", 1, Integer::sum);
```

`putIfAbsent`、`compute`、`merge` 解决的是相应的原子操作需求，不提供多个 key 的整体事务。计算回调应保持短小，不在其中做网络请求等耗时工作。

此外，ConcurrentHashMap 不允许 null key 或 null value。并发更新期间，size 和遍历结果不构成整个 Map 的一致性快照；把一个可变 Position 放进去，也不会自动保护它后续的字段修改。参见 [ConcurrentHashMap API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ConcurrentHashMap.html)。

**面试时可以这样说：**

> JDK 8 的 ConcurrentHashMap 通过 CAS 和桶级同步等机制协调更新，读取通常不用获取更新锁。单次操作安全不代表 get 加 put 的组合原子，计数可以用 merge，多字段或多 key 的一致性要另外设计。

## 8. 原子性、可见性、有序性分别是什么？happens-before 又是什么？

可以用三种故障理解：

- **原子性问题：** 两个线程都执行加一，结果只增加一次。
- **可见性问题：** 一个线程已经写了停止标记，另一个线程缺少同步，不能保证按预期观察到更新。
- **有序性问题：** 一个线程观察到了“准备完成”，却没有可靠地观察到之前准备的数据。

Java 内存模型，也就是 JMM，规定线程之间允许观察到什么结果。它不等于 JVM 堆、栈的内存区域划分，也不能直接理解成“每个线程把主内存复制一份”。

happens-before 用来建立跨线程的可见性与顺序保证。常见规则包括：同一把锁的解锁先行发生于后续加锁；volatile 写先行发生于后续对该变量的读；启动线程在调用 `t.start()` 前完成的操作，对线程 t 启动后的代码可见。这些关系还能传递。参见 [JLS 17.4.5](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html#jls-17.4.5)。

“代码看起来先执行”或“我让另一个线程 sleep 一会儿”都不能替代同步关系。

**面试时可以这样说：**

> 原子性关注操作会不会被并发交错破坏，可见性关注其他线程能看到什么，有序性关注允许观察到的执行顺序。分析代码时，我会找锁、volatile、线程启动等建立的 happens-before 关系，而不是靠延时猜执行顺序。

## 9. synchronized 锁的是什么？加在方法上就线程安全了吗？

每个 Java 对象都关联一个监视器。synchronized 根据写法选择要获取的监视器：

| 写法 | 锁对象 |
|---|---|
| 实例同步方法 | 当前实例 this |
| 静态同步方法 | 声明类的 Class 对象 |
| synchronized(lock) | lock 指向的对象 |

同一时刻只有一个线程能持有同一个监视器；持有者可以重入。正常退出或异常退出同步区域时，都会释放对应的锁。参见 [JLS 的同步语义](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html#jls-17.1)。

下面这样可以保护计数：

```java
final class Counter {
    private final Object lock = new Object();
    private int value;

    void increment() {
        synchronized (lock) {
            value++;
        }
    }

    int get() {
        synchronized (lock) {
            return value;
        }
    }
}
```

关键是读写都遵守同一套同步规则。两个线程分别锁住不同实例，再修改同一份 static 状态，彼此不会互斥；只锁写方法、读方法随意访问，也不构成完整的线程安全设计。

同步块保护期间，其他线程仍然可以执行没有获取这把锁的代码。规范并不会禁止它们访问同一个对象。参见 [JLS synchronized 语句说明](https://docs.oracle.com/javase/specs/jls/se25/html/jls-14.html#jls-14.19)。

**面试时可以这样说：**

> synchronized 提供互斥，也通过同一监视器的解锁与加锁建立可见性。判断线程安全要看共享状态的全部访问路径是否使用同一把锁，以及锁是否覆盖了整个需要原子执行的业务操作。

## 10. synchronized 的锁升级怎么回答？偏向锁还适用于所有版本吗？

传统面试资料经常给出这条路径：

```text
无锁 → 偏向锁 → 轻量级锁 → 重量级锁
```

这可以帮助理解**旧版 HotSpot 的部分优化策略**，但不能当成每次加锁必走的固定流程。

偏向锁试图降低同一线程反复进入同步区域的成本；轻量级方案处理竞争较少的情况；竞争加剧或满足其他条件时，可能膨胀为监视器锁。自旋会占用 CPU，阻塞等待则涉及线程调度，具体策略由实现决定。

版本边界要说清楚：**JDK 15 已默认关闭并弃用偏向锁相关选项。** 因而“当前 JVM 都从偏向锁开始升级”不成立。参见 [JDK 15 发布说明](https://www.oracle.com/java/technologies/javase/15-relnote-issues.html)。

同样，不要把“锁只能升级不能降级”当成跨版本的规范保证；运行时可能回收空闲的膨胀监视器，这和某次持锁期间的竞争处理又是不同问题。可对照 [HotSpot 8 的监视器膨胀与回收实现](https://raw.githubusercontent.com/openjdk/jdk8u/master/hotspot/src/share/vm/runtime/synchronizer.cpp)。

Android 面试里如果追问到对象头、锁字或膨胀过程，我会先确认是在讨论 HotSpot 还是 ART，以及具体版本。语言层面需要保证的是互斥、重入与同步语义，内部编码不是通用答案。

**面试时可以这样说：**

> 锁升级是运行时为不同竞争程度选择实现方式的优化。经典四态流程需要限定旧版 HotSpot，现代 JDK 和 ART 要看各自实现。业务代码应先控制临界区长度、锁竞争和锁顺序，再考虑具体运行时的优化细节。

## 11. volatile 有什么作用？volatile int 的 count++ 安全吗？

volatile 为变量访问提供可见性与相应的顺序约束。比如一次性发布数据：

```java
final class PublishedResult {
    private int result;
    private volatile boolean ready;

    // 约定只有一个写线程，只发布一次
    void publish() {
        result = 42;
        ready = true;
    }

    Integer readIfReady() {
        return ready ? result : null;
    }
}
```

对象已安全交给两个线程、且没有其他写入时，读线程观察到 ready 为 true 后，可以看到此前的 result 写入。这依赖 volatile 的同步关系，不要求 result 也声明 volatile。参见 [JLS volatile 字段](https://docs.oracle.com/javase/specs/jls/se25/html/jls-8.html#jls-8.3.1.4)。

但 `count++` 包含读取、计算、写回。两个线程可能都读到 10，再都写回 11。**volatile 不会把这整个过程变成原子操作。** 可以使用 synchronized，或 `AtomicInteger.incrementAndGet()`。

另一个常见误区是：

```java
volatile Map<String, Integer> prices = new HashMap<>();
```

volatile 修饰的是 prices 引用，对引用的发布有作用；多个线程继续对同一个 HashMap 执行 put，仍然不安全。若采用“构建完整快照后替换引用”的设计，发布后就不能继续修改快照及其中需要保持稳定的可变数据。

快照不可变也不代表更新流程原子。如果多个写线程都基于旧快照计算新值，再分别赋给 prices，仍可能互相覆盖。此时需要单写线程、锁，或 AtomicReference 的 CAS 更新来协调。

实现层面，JVM 会根据目标平台使用相应的内存访问指令与屏障约束。把它背成“禁止所有指令重排”或“每次都绕过 CPU 缓存”都不准确。

**面试时可以这样说：**

> volatile 适合状态标记和满足约束的安全发布，保证的是相关读写的可见性与顺序语义。count++、先判断再更新、多字段一致性都需要额外的原子操作或锁，volatile 引用也不会让对象内部自动线程安全。

![volatile count++ 的读取、计算与写回交错造成丢失更新](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/java_interview_2.jpeg)

## 12. 双重检查单例为什么需要 volatile？

双重检查锁，也就是 DCL，常见写法如下：

```java
final class Client {
    private static volatile Client instance;

    private Client() {}

    static Client getInstance() {
        Client current = instance;
        if (current == null) {
            synchronized (Client.class) {
                current = instance;
                if (current == null) {
                    current = new Client();
                    instance = current;
                }
            }
        }
        return current;
    }
}
```

外层检查让已初始化的调用避开加锁；内层检查防止多个线程先后进入同步块时重复创建。

volatile 负责安全发布：快速路径在锁外读取 instance，需要通过 volatile 读写建立可见性，才能可靠地观察构造阶段完成的初始化。只在创建时加锁、锁外用普通字段读取，缺少这条保证。依据是 [JLS 的 happens-before 规则](https://docs.oracle.com/javase/specs/jls/se25/html/jls-17.html#jls-17.4.5)。

面试时经常用“分配内存、初始化、赋引用发生重排”来解释，它是一种理解风险的简化模型；更准确的表述是，读取线程需要有安全发布保证。

这段代码只解决实例的创建和发布。Client 内部如果还有可变集合或计数，后续并发访问仍要另外保护。

**面试时可以这样说：**

> DCL 的两次检查避免重复创建，volatile 则为锁外快速读取提供安全发布保证。单例创建安全，不等于单例内的所有业务操作都线程安全。

## 13. CAS 是什么？有什么问题？AtomicInteger 就一定比锁快吗？

CAS 通常指 Compare-And-Swap（比较并交换），Java 原子类通过 `compareAndSet` 等方法提供相应操作。它把“比较当前值是否等于预期值，如果相等则更新”作为一个原子操作完成；引用类型比较的是引用身份，不调用 `equals()`。

例如，用 CAS 循环实现加一：

```java
static int increment(AtomicInteger counter) {
    while (true) {
        int previous = counter.get();
        int next = previous + 1;
        if (counter.compareAndSet(previous, next)) {
            return next;
        }
    }
}
```

真实项目直接用 `incrementAndGet()` 即可。CAS 原子操作由 JVM 和平台原子指令等机制支持，不是 Java 层两个普通语句拼出来的检查与赋值。API 的内存语义见 [AtomicInteger 文档](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/atomic/AtomicInteger.html)。

CAS 常见问题有：

- **竞争时反复失败。** 重试增加 CPU 和缓存一致性开销，单个线程也可能长时间抢不到更新机会。
- **只检查当前值。** 无法直接识别 A 变 B 再变 A 的历史，也就是 ABA。
- **原子范围有限。** 两个独立原子变量分别更新，不等于它们构成一个原子事务。

多个字段必须一起变化时，可以用一把锁保护，也可以把它们放进不可变状态对象，用 `AtomicReference` 替换整个状态。如果新状态依赖旧状态，应使用 CAS 循环等原子更新方式，避免先 get 再 set 丢失更新；读线程也应先读取一次引用，再从同一份快照取出各字段。后一种方案仍需处理重试和对象分配成本。参见 [AtomicReference API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/atomic/AtomicReference.html)。

还要注意，`updateAndGet` 等更新函数在竞争下可能重复执行，因此函数应无副作用，不能把发请求、扣款等动作写进去。

统计高并发计数时可以考虑 LongAdder，用分散更新降低争用；但并发更新期间，`sum()` 不是原子快照，不适合据此完成精确的“检查额度再扣减”。参见 [LongAdder API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/atomic/LongAdder.html)。

**面试时可以这样说：**

> CAS 用原子比较更新实现乐观重试，适合较小的状态更新。竞争高时重试也有成本，还要考虑 ABA 和多字段一致性。选择原子类还是锁，要看状态结构、竞争程度和业务语义，不能只比较名字。

## 14. ABA 问题是什么？加版本号就能解决吗？

假设线程 T1 读到引用 A，准备稍后把它更新为 C。在它暂停期间，T2 把引用改成 B，又改回同一个 A。

```text
T1：读到 A ────────────────── CAS(A, C)
T2：       A → B → A
```

普通引用 CAS 比较的是当前引用是否仍为 A，它可能成功。但如果算法还依赖“这个节点没有被移除、复用，或关联关系没有变化”，之前的判断就可能失效。

**是否有问题，取决于业务是否关心中间变化。** 一个只关心当前数值的条件更新，不一定需要识别整段历史。

需要识别变化时，可以把引用和版本戳一起原子更新：

```java
Object a = new Object();
Object b = new Object();
Object c = new Object();
AtomicStampedReference<Object> ref =
        new AtomicStampedReference<>(a, 0);

// 模拟 T1 一次性读取引用与版本戳
int[] stampHolder = new int[1];
Object observed = ref.get(stampHolder);
int observedStamp = stampHolder[0];

// 模拟 T2 在这段间隙完成 A → B → A
ref.compareAndSet(a, b, 0, 1);
ref.compareAndSet(b, a, 1, 2);

// 当前虽然又是 a，但版本戳已从 0 变成 2
boolean success = ref.compareAndSet(
        observed, c, observedStamp, observedStamp + 1);
System.out.println(success); // false
```

这里用顺序代码模拟线程交错，便于观察结果。`get(int[])` 一次读取引用和戳；分别调用 getReference 与 getStamp，则可能读到两次更新之间不匹配的组合。参见 [AtomicStampedReference API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/atomic/AtomicStampedReference.html)。

版本方案要求所有相关更新都遵守同一个协议，并避免版本在观察窗口内复用或回绕；它也不会自动发现引用对象内部的随意修改。[AtomicMarkableReference](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/atomic/AtomicMarkableReference.html) 只携带布尔标记，适合标记状态，不能当成记录任意次数变化的递增版本号。

Java 的 GC 处理对象内存回收，但程序仍可能把同一个对象引用移出再放回，因此不能说“有 GC 就不会有 ABA”。

**面试时可以这样说：**

> ABA 是当前值恢复了，但中间状态已经变化。算法关心这段历史时，可以把版本戳和引用作为整体比较更新，或者用锁保护完整操作。版本号需要覆盖所有相关更新，布尔标记和 GC 都不能通用地消除 ABA。

![普通引用 CAS 与携带版本戳的 CAS 在 ABA 场景中的对比](https://cdn.jsdelivr.net/gh/visiongem/BlogImages@master/android/2026/java_interview_3.jpeg))

## 15. synchronized 和 ReentrantLock 怎么选？

synchronized 是语言提供的内置锁，ReentrantLock 是显式锁。两者都提供可重入的互斥和相应的内存可见性语义。区别主要在能力与使用方式：

| 能力 | 内置锁 | 显式锁 |
|---|---|---|
| 释放 | 自动 | 手动 |
| 限时获取 | 不支持 | 支持 |
| 中断获取 | 不支持 | 支持 |
| 公平配置 | 不支持 | 可选 |
| 等待条件 | 单一等待集 | 多个条件队列 |

ReentrantLock 用 `tryLock` 尝试或限时获取锁，用 `lockInterruptibly` 可中断地等待获取锁，并通过 `newCondition()` 创建多个条件队列。synchronized 则通过对象的 `wait`、`notify`、`notifyAll` 配合条件等待。

使用 ReentrantLock 时，在成功获取锁后进入 try，并在 finally 中释放：

```java
lock.lock();
try {
    updateState();
} finally {
    lock.unlock();
}
```

如果用 tryLock，只有成功获取后才能 unlock。公平锁也不等于线程调度绝对公平；无参数 `tryLock()` 不遵守公平排队设置。参见 [ReentrantLock API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/locks/ReentrantLock.html)。

一个常见追问是 wait 与 sleep：wait 需要持有对应监视器，等待时释放这个监视器；sleep 不会释放已经持有的监视器。wait 返回前需要重新获取锁，且应放在检查条件的 while 循环中，处理虚假唤醒与条件再次变化。参见 [Object.wait 文档](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/lang/Object.html#wait())。

不管选哪一种锁，多把锁获取顺序不一致都可能造成死锁。可以统一锁顺序，并缩短持锁时间，避免在锁内执行不受控的外部调用。

**面试时可以这样说：**

> 简单临界区可以用 synchronized，需要限时获取、可中断等待或多个条件队列时考虑 ReentrantLock。性能要测量，正确性上则要保证释放锁、统一锁顺序和控制临界区范围。

## 16. 1,500 个仓位，每个 Item 开一个线程算数据，有什么问题？用单线程池呢？

如果这里指的是平台线程，一条仓位开一个线程会带来线程栈内存、创建与调度开销。CPU 核心数量有限，增加线程不会同比增加计算能力。

即使换成线程池，也要追问：输入每秒多少条？一批计算多久？任务能否合并？等待队列有多长？

例如，**假设**每秒收到 10 轮行情，每轮为 1,500 个仓位各提交一个任务，就会产生每秒 15,000 个任务。如果处理速度跟不上，旧任务会越积越多，页面显示的结果也越来越滞后。

面试里如果听到“single”，先确认是 `newSingleThreadExecutor()`，还是其他库的 Single 类型。若指单线程池，要分两种情况：

- **所有仓位共享一个单线程池：** 任务串行执行，重任务会阻塞后续任务；默认队列无界，持续提交可能造成积压。
- **每个 Item 各建一个单线程池：** 线程总数仍可能随 Item 数量增长，也很难统一管理生命周期。

`newFixedThreadPool` 同样使用无界队列。固定线程数控制了执行量，却没有同时限制等待任务总量。参见 [Executors API](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/Executors.html)。

面试时可以按下面的顺序给出方案：

**先减少任务数量。** 几次乘加运算可以按批计算；相同交易品种共享行情解析和公共计算。只有展示快照允许覆盖时，才合并旧任务、保留最新输入；成交、持仓增减等业务增量必须按协议处理。

**再控制并行度与队列。** 重计算放到复用的后台执行器，结合设备能力和实测耗时设置并行度。有界队列满后，要明确是合并、拒绝、暂停生产还是走其他处理路径，不能只加一个队列容量就算完成。

**最后控制结果提交。** 用请求代次或状态版本拒绝过期结果，合并 UI 刷新；页面离开后取消无用任务，计算循环也要配合中断或取消检查。UI 更新回到主线程，线程池由明确的所有者负责关闭。

线程池原理也常跟着追问。在执行器正常运行、线程能够成功创建的通常路径中：

```text
工作线程数不足 corePoolSize
    → 尝试创建线程处理任务
否则尝试进入队列
    → 入队失败，再尝试增加线程至 maximumPoolSize
仍然无法接受
    → 执行拒绝策略
```

因此，无界队列通常会让 maximumPoolSize 失去扩展线程数的作用。`CallerRunsPolicy` 会让未关闭执行器的提交者执行被拒绝任务；如果提交者是 Android 主线程，重计算就可能回到主线程造成卡顿。参见 [ThreadPoolExecutor 的队列与拒绝策略](https://docs.oracle.com/en/java/javase/25/docs/api/java.base/java/util/concurrent/ThreadPoolExecutor.html)。

**面试时可以这样说：**

> 这个场景要同时控制计算量、线程数和待处理任务数。我会先批量计算与复用结果，再按负载设置有界执行方案，明确过载策略，并在提交 UI 前检查结果版本。单线程能简化部分并发问题，但不能解决吞吐不足和任务积压。

---

复习时可以把这几题连起来：仓位用什么 key 保存，谁负责修改集合，一次更新涉及几个字段，读线程如何拿到完整状态，旧计算为什么不能覆盖新行情。把这些约束讲清楚，HashMap、锁、volatile 和 CAS 才能对应到具体代码。

---

🌈关注我吖~❤️

公众号：**妮K妮K妮**
