# java并发

## java线程的创建方式

通过直接new一个线程 Thread或者通过线程池管理多个线程

还可以通过创建一个任务 runable或者callable交给线程来执行

runable可以直接传递给线程执行   没有返回值 不能抛出异常  callable需要通过futureTask包装成runable来执行   有返回值 可以抛出异常

因为futureTask本身实现了runable和 future

## JMM是什么

JMM 是 Java 内存模型，用于规定多线程下共享变量的访问规则，保证原子性、可见性和有序性。

它定义了主内存和线程工作内存的交互规则，解决了 CPU 缓存、指令重排导致的线程安全问题。

Java 中通过 volatile、synchronized、原子类等机制体现 JMM 规范。

## happens-before 规则

- 程序顺序规则
- 同一线程内，前面的操作 happens-before 后面的操作
- volatile 规则
- 对一个 volatile 变量的写\
  happens-before\
  后续对这个变量的读
- 锁规则
- unlock happens-before lock

Thread A: synchronized(lock){ x = 1; } // unlock

Thread B: synchronized(lock){ print(x); } // lockB 一定能看到 A 对 x 的修改

- 线程启动规则
- `Thread.start()` happens-before 线程内操作
- 线程终止规则
- 线程内操作 happens-before `Thread.join()`

## `volatile` 关键字的底层实现原理是什么？JMM 是通过什么机制，来保证它的可见性和有序性的？

volatile 的底层实现是 JVM 在读写 volatile 变量时插入内存屏障，并利用 CPU 的缓存一致性协议来实现的( volatile 写 → 会生成带 `lock` 前缀的指令（如 `lock addl`）  )。\
写 volatile 会强制刷新并使其他 CPU 缓存行失效，保证可见性；同时通过禁止特定指令重排序（内存屏障），建立 happens-before 关系，保证有序性，但不保证复合操作的原子性。

## CAS（Compare-And-Swap）原理是什么？ 它在 Java 中如何实现的？

CAS（Compare-And-Swap）是一个原子操作，通过比较内存值和预期值来决定是否更新，它依赖 CPU 的原子指令 保证不可分割性。在 Java 中，CAS 由 `Unsafe` 类封装，通过 CPU 指令实现，`AtomicInteger`、`AtomicReference` 等类就是在自旋循环中利用 CAS 实现线程安全的无锁原子更新。

CAS 本质是 CPU 提供的“原子比较交换指令”，JVM 只是调用者。

## `synchronized` 在 JVM 中是如何实现的？它、经历了哪些锁优化阶段？

实现入口：

- 同步代码块在字节码层通过 `monitorenter/monitorexit` 指令实现。
- 同步方法通过方法标志位 `ACC_SYNCHRONIZED` 实现。

对象头相关（重点）：

- HotSpot 对象头核心包含 `Mark Word` 与 `Klass Pointer`（数组对象还有长度）。
- `Mark Word` 会复用存储哈希、GC 年龄、锁标志位，以及在不同锁状态下的线程/指针信息。
- 锁优化本质是 `Mark Word` 表达语义与指向目标的切换。

锁优化阶段（JDK 1.6+ 历史路径）：

1. 无锁：对象头处于普通状态。
2. 偏向锁：无竞争且同线程反复进入时，在对象头记录偏向线程，重入几乎零开销。
3. 轻量级锁：出现竞争后，线程在栈帧建立 `Lock Record`，CAS 尝试把对象头替换为该记录指针；失败先自旋。
4. 重量级锁：竞争持续则锁膨胀为 `ObjectMonitor`，对象头指向 Monitor，未获取锁线程进入 `EntryList` 阻塞等待。

性能解释：

- 低/中竞争场景优先走偏向与轻量级路径，减少内核态阻塞与线程切换开销。
- 高竞争场景最终进入重量级锁，保证正确性，但吞吐与延迟会退化。

版本补充：

- 偏向锁在较新 JDK 已逐步退出默认路径（JDK 15 默认关闭，JDK 18 移除）。

## 什么是偏向锁？它解决的是什么问题？

偏向锁是 JVM 对 synchronized 的一种优化，适用于几乎没有竞争且同一线程反复获取锁的场景。它通过在对象头中记录线程 ID，使得同一线程再次进入同步块时无需任何同步操作，从而几乎消除了加锁开销。当出现其他线程竞争时，偏向锁会被撤销并升级为轻量级或重量级锁。

## synchronized为什么是非公平

synchronized 在轻量级阶段通过 CAS 抢占锁，没有排队机制；\
在重量级阶段虽然有 EntryList，但并不是严格 FIFO。

JVM 设计它为非公平锁，是为了减少线程上下文切换，提高吞吐量。\
公平锁虽然避免饥饿，但会降低整体性能。

## volatile 和 synchronized 的本质区别是什么？各自适用于什么场景？

volatile 通过内存屏障和缓存一致性协议保证变量的可见性和有序性，但不提供互斥和原子性；synchronized 基于 JVM Monitor 机制实现，既保证可见性和有序性，又通过互斥保证复合操作的原子性，适用于需要保护临界区的场景

## synchronized 和 ReentrantLock  区别

synchronized   是 JVM 提供的内置锁    reentrantLock是基于AQS实现的的锁，他可中断，显示释放锁，可以控制是否公平，以及多个条件队列。在性能上其实两者差距不大，reentrantLock功能更多，适合更复杂的并发控制场景

<br />

## 线程池参数调优思路

IO密集型:线程大量时间在等待 cpu占用时间低  线程数 ≈ CPU核数 × (1 + 等待时间 / 计算时间)

而对于cpu密集型  线程会一直占用cpu 线程多了会增加上下文切换所以  线程数 ≈ CPU 核心数

理论最佳实践 线程数 = CPU 核心数 + 1   因为：

- 可能存在极短暂阻塞
- 多一个线程填补空隙

QPS 高说明：

- 同时在途请求很多
- 每个请求都要占用线程
- 如果线程数太少，会形成排队
- 排队会增加响应时间
- 最终导致超时、雪崩

所以： IO 型 + 高 QPS → 需要足够多线程覆盖阻塞时间

不过也不是线程数越多越好，需要找到系统瓶颈

在 IO 型系统中，瓶颈通常不在 CPU，而在：

- DB 连接池
- Redis 连接池
- 下游 RPC
- 磁盘
- 网络带宽

如果你线程池开得过大：

- DB 连接打满
- 下游被压垮
- 系统整体 RT 飙升

根据  Little's Law：   系统中平均在途请求数 = 吞吐率(qps) × 平均响应时间  (rt)

线程池核心线程数覆盖这个数就行

maximumPoolSize

- 应对突发流量
- 不能无限扩
- 受内存和上下文切换限制

考虑因素：

- 单线程栈大小（默认 1M）
- 线程过多会导致频繁调度
- 上下文切换成本

所以：

最大线程数受内存和系统调度能力限制

在io型和高qps情况 阻塞队列不能太大 会导致请求堆积 rt增加  队列太小又会频繁触发拒绝策略

## Tomcat / Dubbo / Netty 各自线程池设计差异

### tomcat:

Tomcat 主要线程：

&#x20;

（1）Acceptor 线程

- 接收 TCP 连接
- 把 socket 交给 worker

***

（2）Worker 线程池（核心）

每个请求 → 一个线程处理完整生命周期

缺点:

     阻塞严重

&#x20;

          DB / IO 一阻塞 → 线程被占住

     线程数 = 并发上限

        线程池满 → 请求排队

    上下文切换成本高

<br />

### Dubbo

IO线程 和 业务线程分离

1）IO线程（Netty）

&#x20;

负责：网络收发 + 编解码

特点：

- &#x20;不能阻塞&#x20;
- &#x20;CPU密集但轻量&#x20;

***

（2）业务线程池（Dubbo ThreadPool）

IO线程 → 丢给业务线程池 → 执行业务逻辑

Dubbo模型

&#x20;

```
Client
  ↓
Netty IO线程（接收请求）
  ↓
Dispatcher
  ↓
业务线程池（fixed/cached）
  ↓
Service执行
```

### Netty 线程池设计（Reactor模型核心）

核心思想：事件驱动 + 非阻塞

    不是“来一个请求开一个线程”
而是“事件来了才处理”

&#x20;

<br />

两类核心线程池

&#x20;

   （1）BossGroup

          负责：连接建立（accept）

   （2）WorkerGroup

          负责：IO读写 + pipeline处理

<br />

Reactor模型

&#x20;

     Netty = 多路复用 + 事件循环：

```
Selector
  ↓
事件触发（read/write/connect）
  ↓
EventLoop线程处理

## 线上有个批量处理接口，请求进来后会把 5000 条数据拆成多个任务扔进线程池执行。最近高峰期出现：1. 接口 RT 很高，偶发超时；2. 机器 CPU 不满，但线程数很多；3. 日志里偶尔出现任务堆积，甚至 RejectedExecutionException；4. 下游数据库连接池也经常被打满。请你回答：1. 你会怎么判断问题到底在线程池、任务模型，还是下游资源？2. 线程池参数 corePoolSize、maxPoolSize、queueCapacity、拒绝策略你会怎么设计？为什么？3. 为什么“线程数很多但 CPU 不高”仍然可能很慢？4. 如果必须保护下游数据库，你会在线程池之外再加什么机制？

- 排查思路（怎么看）：按同一时间窗对齐 `QPS/p95-p99/超时率` 与线程池指标 `activeCount/poolSize/queueSize/rejectCount/任务等待时长/执行时长`，再对齐下游指标（DB 连接池等待、慢 SQL、RPC 超时、Redis 慢命令），先判断是“流量突增打满”还是“下游变慢传导”。  
- 定位规则（看到什么代表什么）：  
  - `activeCount≈max && queueSize≈满 && rejectCount↑` -> 容量打满；  
  - `queueSize↑但activeCount不高` -> 线程阻塞/死锁/外部依赖卡住；  
  - `QPS平稳但执行时长↑` -> 下游退化而非入口流量问题；  
  - `线程数很多但CPU不高` -> 大量线程在 WAITING/BLOCKED，常见是 IO、锁、连接池等待。  
- 处置策略（怎么解决）：  
  - 流量打满：先限流+降级止血，再按压测结果扩容 `maxPoolSize` 与实例数；  
  - 下游退化：给 DB/RPC 设置超时、熔断、重试预算，优先修慢 SQL 与连接池耗尽；  
  - 阻塞争用：抓线程栈定位热点锁/等待点，拆小临界区或改异步；  
  - 突刺流量：预热线程池、入口令牌桶平滑流量，避免瞬时打满。  
- 参数设计：`corePoolSize` 按稳定并发定，`maxPoolSize` 按突发容忍和下游承载上限定；`queueCapacity` 用有界队列，避免“拒绝变少但尾延迟暴涨”；`keepAliveTime` 用于回收突发后冗余线程。  
- 拒绝策略：`CallerRunsPolicy` 适合需要反压的链路，但会拖慢调用方；核心链路常用“快速失败+降级+告警”，非核心链路可丢弃或延迟重试。  
- 保护下游：在线程池外增加限流、信号量、舱壁隔离，必要时用 MQ 削峰；线程池并发上限必须与 DB 连接池、下游 RPC 并发上限联动配置。  
```

<br />

EventLoop特点（重点）

&#x20;

✔ 一个线程绑定多个 Channel

- 一个线程处理 N 个连接

✔ 线程永不阻塞

- IO必须是非阻塞

✔ 单线程串行处理事件

- 避免锁竞争

EventLoop 工作流程

1. select() 监听事件
2. IO事件触发（read/write）
3. 执行 ChannelPipeline
4. 执行任务队列
5. 循环

Netty 为什么不会阻塞 IO线程（重点）

EventLoop → IO处理\
&#x20;↓\
业务线程池（异步执行）

EventLoop 并不是在“处理请求”，而是在做

1. IO事件检测（select）
2. socket读写
3. pipeline快速处理
4. 投递任务

### 总结

Tomcat 是“一请求一线程的阻塞式线程池模型”，Dubbo 是“IO线程与业务线程池分离的RPC线程模型”，Netty 是“基于Reactor事件驱动的少线程EventLoop模型（IO复用 + 事件循环）”。

<br />

<br />

## ConcurrentHashMap

### ConcurrentHashMap 1.8 的核心实现原理是什么？

JDK 1.8 的 `ConcurrentHashMap` 底层可以概括为：`Node[] + 链表 + 红黑树 + CAS + synchronized`。

它的核心思路是：

- 读操作基本无锁，依赖 `volatile` 保证可见性
- 写操作优先走 `CAS`，只有发生哈希冲突时才锁桶头节点
- 链表过长时会树化，避免极端冲突下退化成长链表
- 扩容时支持多个线程一起协同迁移

所以和 JDK 1.7 相比，JDK 1.8 的并发粒度更细，结构也更统一。

### ConcurrentHashMap 为什么不允许 null key 和 null value？

`ConcurrentHashMap` 禁止 `null key` 和 `null value`，核心原因是为了避免并发语义歧义。

如果允许 `null`，那么 `get(key)` 返回 `null` 时无法区分：

- 这个 key 根本不存在
- 这个 key 存在，但 value 恰好是 `null`

在并发场景下，这会让很多判断逻辑变得不可靠，所以 `ConcurrentHashMap` 直接禁止存储 `null`。

### ConcurrentHashMap 的 get() 为什么基本不加锁？

JDK 1.8 的 `ConcurrentHashMap` 之所以能做到读操作基本无锁，关键在于可见性设计：

- `table` 是 `volatile`
- `Node` 的 `val` 和 `next` 也是 `volatile`
- 节点在插入时是“先构造完成，再发布出去”

因此 `get()` 时通常只需要：

1. 计算 hash
2. 定位桶
3. 判断桶头、链表或红黑树
4. 返回结果

整个过程一般不需要加锁，也能读到正确数据。

### ConcurrentHashMap 1.8 的 putVal() 流程是什么？

可以按这条主线理解：

1. 先判空，`key` 和 `value` 都不允许为 `null`
2. 对 `key.hashCode()` 做 `spread()` 扰动，减少哈希冲突
3. 如果 `table` 还没初始化，就先走 `initTable()`
4. 用 `(n - 1) & hash` 定位桶下标
5. 如果桶为空，优先通过 `CAS` 直接插入新节点，不加锁
6. 如果桶头是 `MOVED`，说明当前正在扩容，当前线程会进入 `helpTransfer()` 协助迁移
7. 如果桶不为空且不是 `MOVED`，就 `synchronized(f)` 锁住桶头节点
8. 加锁后再次确认 `tabAt(tab, i) == f`，防止加锁前桶头已经变化
9. 如果是普通链表，就遍历链表，找到相同 key 则覆盖旧值，没找到则尾插新节点
10. 如果是 `TreeBin`，就按红黑树逻辑插入或更新
11. 链表节点数达到阈值后尝试树化，但容量较小时会优先扩容
12. 如果本次是新增节点，最后通过 `addCount(1L, binCount)` 更新计数并检查是否需要扩容

面试回答可以概括为：`ConcurrentHashMap 1.8` 的 `putVal()` 是“空桶 CAS，冲突后锁桶头，遇到扩容就协助迁移，新增后再统一计数和扩容判断”。

### ConcurrentHashMap 为什么锁桶头节点就够了？

因为 `ConcurrentHashMap` 在桶内操作时，所有结构性修改都是从桶头开始进入的：

- 链表插入、遍历都从桶头出发
- 红黑树操作也以桶头对应的结构为入口
- 同一个桶内的修改必须串行完成

所以只要锁住当前桶头节点，就能把这个桶的结构修改串行化，而不需要锁整张表。

### sizeCtl 和 initTable()/addCount() 是怎么配合控制初始化与扩容的？

<br />

`sizeCtl` 不是单纯的扩容阈值，而是一个复用状态变量。

- `sizeCtl = 0`：table 还没初始化，初始化时使用默认容量
- `sizeCtl > 0`：正常状态，表示下一次扩容阈值，通常是 `capacity * 0.75`
- `sizeCtl = -1`：有线程正在初始化 table
- `sizeCtl < -1`：当前正在扩容，而且这个负值里还编码了扩容标识和参与扩容线程数

`initTable()` 负责“从无到有”的初始化：

1. 如果发现 `table == null` 或长度为 0，就尝试初始化
2. 线程先看 `sizeCtl`
3. 如果 `sizeCtl < 0`，说明别的线程正在初始化，当前线程先自旋/让步等待
4. 如果 `sizeCtl >= 0`，当前线程通过 `CAS` 把 `sizeCtl` 改成 `-1`
5. 抢到初始化资格的线程真正创建 `table`
6. 创建完成后把 `sizeCtl` 设为新的扩容阈值，例如容量 16 时阈值为 12

`addCount()` 负责“从小到大”的扩容控制：

1. 在新增元素后更新总数，低竞争时更新 `baseCount`，高竞争时分散到 `CounterCell`
2. 汇总当前元素个数 `s`
3. 如果 `s` 还没到 `sizeCtl`，说明暂时不需要扩容
4. 如果 `s >= sizeCtl`，就尝试发起扩容
5. 第一个成功修改扩容状态的线程负责启动 `transfer()`
6. 其他线程如果发现已经在扩容，就可以通过 `helpTransfer()` 参与搬迁
7. 扩容结束后，把 `table` 指向新表，并把 `sizeCtl` 更新成新容量对应的阈值

一句话总结：`initTable()` 让 `sizeCtl` 从“未初始化/初始化中”切到“正常阈值”，`addCount()` 让它从“正常阈值”切到“扩容中”，扩容完成后再切回“新阈值”。

### transfer() 和 ForwardingNode 是怎么让 ConcurrentHashMap 安全协同扩容的？

`ConcurrentHashMap 1.8` 的扩容不是单线程搬家，而是多个线程一起迁移旧表到新表。

核心角色有四个：

- `table`：旧表
- `nextTable`：新表
- `transfer()`：负责搬迁数据
- `ForwardingNode`：旧桶迁移完成后的占位标记，`hash = MOVED`

工作过程可以这样理解：

1. 当 `addCount()` 判断需要扩容后，会创建 `nextTable`
2. 多个线程通过共享的 `transferIndex` 分段认领桶区间，避免重复搬同一批桶
3. 每个线程逐桶处理：
   - 空桶：直接放一个 `ForwardingNode`，表示这个桶已经处理过
   - 链表桶：按 `(hash & oldCap)` 拆成两组，分别放到新表的原索引和 `原索引 + oldCap`
   - 红黑树桶：也是按“原位 / 原位 + oldCap”拆分迁移，必要时可能退化为链表
4. 每迁完一个桶，就把旧桶位置替换为 `ForwardingNode`
5. 其他线程在 `put/get` 时如果看到桶头 `hash == MOVED`，就知道这个桶已经迁移完成，不能再在旧桶上操作，而要去新表查找或进入 `helpTransfer()`
6. 当所有桶都迁移完成后，最后收尾的线程会执行：
   - `table = nextTable`
   - `nextTable = null`
   - `sizeCtl = 新阈值`

`ForwardingNode` 的作用有两个：

- 标记旧桶已经迁移，防止重复搬迁
- 作为跳转路标，告诉其他线程去新表继续操作

所以可以把这段机制概括成：`transfer()` 负责“搬家”，`ForwardingNode` 负责“立路标”，两者一起保证并发扩容期间的正确性和效率。

### ConcurrentHashMap 1.7 和 1.8 的核心区别是什么？

JDK 1.7 的 `ConcurrentHashMap` 采用的是 `Segment[] + HashEntry[]` 的分段锁结构，本质上是把一个大 Map 拆成多个小 Hash 表，每个 `Segment` 自带一把锁。

JDK 1.8 去掉了 `Segment`，改成更接近 `HashMap` 的单层结构：

- 1.7：`Segment` 分段锁
- 1.8：`CAS + synchronized + Node[] + 链表/红黑树`

核心差异在于：

- 1.7 写操作通常先锁段
- 1.8 空桶写入优先 CAS，冲突时只锁桶头
- 1.8 支持多线程协同扩容
- 1.8 与 `HashMap 1.8` 的结构更统一

因此 JDK 1.8 的锁粒度更细，并发扩展性更好。

### 为什么 JDK 1.8 的 ConcurrentHashMap 去掉了 Segment？

JDK 1.7 的 `ConcurrentHashMap` 采用 `Segment[] + HashEntry[]` 的分段锁模型，写操作要先定位到某个 `Segment`，再锁整段。这样虽然比整表一把锁更好，但同一个段里的不同桶仍然会互斥，锁粒度还是偏粗，并发上限也容易被段数限制。

JDK 1.8 去掉 `Segment` 后，改成 `Node[] table + CAS + synchronized`：

- 空桶插入优先走 `CAS`，很多写操作根本不需要加锁
- 发生冲突时只锁当前桶头节点，而不是整段
- 底层结构和 `HashMap 1.8` 更统一，更容易支持红黑树和协同扩容

所以 1.8 去掉 `Segment` 的核心原因是：分段锁粒度不够细、并发扩展性不够好，而桶级锁 + CAS 可以提供更高并发度和更简洁的实现。

### 为什么 JDK 1.8 用 synchronized 反而比 JDK 1.7 的 ReentrantLock + Segment 更先进？

不是因为 `synchronized` 这个关键字本身一定比 `ReentrantLock` 更强，而是因为 JDK 1.8 的整体并发模型升级了。

JDK 1.7 的问题核心不在 `ReentrantLock` 本身，而在于它锁的是整个 `Segment`。即使两个线程操作的是同一段里两个不同的桶，也会互斥，锁粒度偏粗。

JDK 1.8 的思路变成了：

- 空桶插入先 `CAS`
- 只有桶不为空、真正发生冲突时才 `synchronized(f)` 锁桶头
- 锁的对象是单个桶头节点，不是整段

这意味着：

- 低冲突时大量写操作根本不走锁路径
- 发生冲突时也只是桶级互斥，不同桶之间仍可并行写
- 实现上不再需要维护 `Segment` 这层复杂结构

再加上 JDK 1.6 以后 JVM 对 `synchronized` 做了大量优化，所以在 JDK 1.8 这种“CAS 优先、桶级锁兜底”的模型里，`synchronized` 反而成了更合适、更简洁的选择。

<br />

<br />

## LongAdder 为什么比 AtomicLong 性能好

核心机制：

- `AtomicLong` 的所有线程都 CAS 同一个值，热点集中，冲突高时会频繁自旋重试。
- `LongAdder` 采用 `base + Cell[]` 两层结构：低冲突先更新 `base`，冲突后把线程分散到不同 `Cell` 进行 CAS。
- 单点竞争被拆为多点竞争，显著降低 CAS 失败概率与缓存行争用，因此高并发写场景吞吐更高。

代价与边界：

- `sum()` 是聚合 `base + 所有 Cell` 的结果，在并发更新期间读到的是近似实时值，不是全局强一致快照。
- 适合监控计数、QPS、埋点等“高并发累加 + 对瞬时强一致要求不高”的场景。
- 对严格一致计数（如库存/余额精确阈值判断）优先 `AtomicLong` 或加锁方案。

<br />

## ThreadLocal 是如何实现线程隔离的？数据存在哪里？为什么会内存泄漏？为什么 key 是弱引用？

先说threadlocal数据结构:

每个线程内部都有一个thredlocalMap : key是threadlocal本身   value是要存的对象

发生threadLocal.set(value)时; 取出自己的 ThreadLocalMap →\
以 threadLocal 作为 key → 存入 value  这样就是实现了 线程隔离

<br />

为什么会内存泄漏?

ThreadLocal 内存泄漏的原因是 ThreadLocalMap 中 key 是弱引用，ThreadLocal 被回收后 key 变为 null，但 value 仍然被线程强引用持有，尤其在线程池线程长期存活的情况下无法被及时清理，从而导致内存泄漏；key 使用弱引用是为了避免 ThreadLocal 本身无法被 GC，而 value 保持强引用是为了保证数据可用性。

<br />

为什么 key 必须设计成弱引用？

1:避免 ThreadLocal 本身无法回收

2:避免“ThreadLocal泄漏链条更严重”

3:设计目标：让 ThreadLocal “可被回收”

<br />

## 什么是 AQS？它为什么能同时支撑 ReentrantLock、Semaphore、CountDownLatch？

### 标准面试回答

AQS（`AbstractQueuedSynchronizer`）是一个“同步器模板框架”，核心是：

1. 一个 `volatile int state` 表示同步状态
2. 一套基于 CAS 的状态更新机制
3. 一个 FIFO 等待队列（CLH 变体）管理竞争失败线程

它把“排队、阻塞、唤醒”的公共逻辑抽象出来，把“state 的业务语义”交给子类实现，因此能复用到多种同步器。

### 详细讲解

#### 1）`state` 在不同同步器中的含义

- `ReentrantLock`：`state` 表示重入次数
  - `0`：无锁
  - `>0`：锁已被持有，同线程重入会继续递增
- `Semaphore`：`state` 表示可用许可数
  - `acquire` 递减
  - `release` 递增
- `CountDownLatch`：`state` 表示剩余倒计数
  - 只递减不递增
  - 归零后唤醒等待线程

关键点：AQS 不关心 `state` 是“锁次数”还是“许可数”，只关心你是否能原子更新它。

#### 2）独占模式和共享模式的本质差异

- 独占模式（如 `ReentrantLock`）：同一时刻只允许一个线程成功获取。
- 共享模式（如 `Semaphore`/`CountDownLatch`）：一个线程成功后可能继续传播唤醒，让后继也通过。

#### 3）获取失败后，线程经历了什么

以独占获取为例：

1. 线程先走 `tryAcquire`（CAS 抢 `state`）
2. 失败后封装为 `Node` 入 AQS 队列
3. 当前驱是 `head` 时才有资格重试获取
4. 仍失败则 `park` 挂起，等待前驱释放后 `unpark`
5. 被唤醒后再次重试，成功则成为新 `head`

这套机制是“短自旋 + 排队阻塞 + 唤醒重试”，避免无序竞争导致 CPU 空转。

#### 4）为什么是 `CAS + volatile + 队列`

- `CAS`：保证状态更新原子性，快路径无锁
- `volatile`：保证状态可见性和有序性
- 队列：把竞争失败线程有序管理，避免大量线程无效自旋

三者组合后，AQS 同时具备正确性、吞吐和可扩展性。

#### 5）AQS 里为什么要有哨兵 `head` 节点？没有会怎样？

`head` 的作用不是“装饰”，而是并发协议锚点：

1. 统一资格判定：通常前驱为 `head` 的节点才重试获取
2. 简化出队：成功线程通过“成为新 `head`”完成出队
3. 稳定唤醒：释放时可从 `head` 出发选择后继，降低丢唤醒风险
4. 降低边界复杂度：空队列/首节点/并发入队处理更稳定

没有 `head` 不是绝对不能实现，但会明显增加并发边界复杂度，更容易出现唤醒和链表一致性问题。

#### 6）既然有队列，为什么还会“非公平”？

队列存在 != 全局公平。

- 公平锁：新线程先检查 `hasQueuedPredecessors()`，队列里有人就不插队。
- 非公平锁：新线程先 CAS 抢一次，抢到就直接拿锁，即使队列里有人在等。

所以准确表述是：

- 队列内大体是近似 FIFO 重试顺序
- 但非公平策略允许“队列外线程插队”，因此全局不是严格公平

## ReentrantReadWriteLock 工作原理

读锁 共享 可重入

写锁 排他 可重入

- 读锁和写锁不能同时存在
- 写锁存在 → 新的读锁请求被阻塞
- 读锁存在 → 写锁请求被阻塞
- 同一线程可以多次获取写锁（state++）

内部 state 机制（基于 AQS）

- `state` 是一个 int，32 位
  - 高 16 位  ：读锁计数
  - 低 16 位  ：写锁计数
- 获取写锁：CAS + 判断当前线程是否已持有锁
- 获取读锁：
  1. 判断写锁是否存在
  2. 高 16 位自增（CAS 原子更新）
  3. 线程本地计数器（ThreadLocal）记录每个线程的读锁数，保证释放安全

## 请解释 Semaphore 的原理和典型应用：Semaphore 是计数信号量还是二元信号量？acquire() 和 release() 是怎么实现的？它是如何保证并发线程数控制的？给一个实际应用场景例子。

- 类型  ：`Semaphore` 是计数信号量，可用作二元信号量
- acquire/release  ：
- acquire() CAS 减 1，阻塞队列等待
- release() 增 1，唤醒等待线程
- 保证并发数  ：
- permits = 最大并发线程数
- 同时访问线程 ≤ permits
- 典型场景  ：
- 资源池限制并发
- 流量控制
- 模拟有限资源

```plain
Semaphore semaphore = new Semaphore(5);

Runnable task = () -> {
    try {
        semaphore.acquire(); // 获取许可
        System.out.println(Thread.currentThread().getName() + " 执行数据库操作");
        Thread.sleep(1000); // 模拟操作
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        semaphore.release(); // 释放许可
    }
};

for (int i = 0; i < 10; i++) {
    new Thread(task).start();
}Semaphore semaphore = new Semaphore(5);

Runnable task = () -> {
    try {
        semaphore.acquire(); // 获取许可
        System.out.println(Thread.currentThread().getName() + " 执行数据库操作");
        Thread.sleep(1000); // 模拟操作
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        semaphore.release(); // 释放许可
    }
};

for (int i = 0; i < 10; i++) {
    new Thread(task).start();
}
```

## CountDownLatch（控制时序）

- 一个或多个线程等待其他线程执行完毕
- 本质是“等待器”

只有coutdown  = state--  当state = 0  唤醒所有await线程

```java
CountDownLatch latch = new CountDownLatch(3);

new Thread(() -> {
    loadConfig();
    latch.countDown();
}).start();

new Thread(() -> {
    initCache();
    latch.countDown();
}).start();

new Thread(() -> {
    initDB();
    latch.countDown();
}).start();

latch.await();  // 主线程等待
System.out.println("系统启动完成");

并发测试（压测神器）
```

## CountDownLatch 为什么不能 reset？为什么它不像 CyclicBarrier 那样可以循环使用？

CountDownLatch 设计为一次性同步工具，基于 AQS 共享模式实现，state 只递减不递增。一旦计数归零，所有等待线程被释放。如果支持 reset，会破坏已唤醒线程的语义一致性，并引入复杂的竞态控制问题。因此 JDK 将循环同步场景交给 CyclicBarrier 处理，后者内部通过 generation 实现阶段性重置。

```java
import java.util.concurrent.CyclicBarrier;

public class Demo {

    public static void main(String[] args) {
        CyclicBarrier barrier = new CyclicBarrier(3);

        Runnable task = () -> {
            try {
                System.out.println(Thread.currentThread().getName() + " 准备中...");
                Thread.sleep(1000);

                barrier.await();  // 等待其他线程

                System.out.println(Thread.currentThread().getName() + " 开始执行任务");
            } catch (Exception e) {
                e.printStackTrace();
            }
        };

        for (int i = 0; i < 3; i++) {
            new Thread(task).start();
        }
    }
}
```

## CyclicBarrier 和 Phaser 有什么区别？Phaser 为什么说是增强版的 CyclicBarrier？

## ArrayBlockingQueue 和 LinkedBlockingQueue 有什么区别？

ArrayBlockingQueue 是基于数组的有界阻塞队列，使用单锁控制

- put 和 take 共用一把锁
- 生产和消费不能并行

LinkedBlockingQueue 是基于链表的阻塞队列，默认近似无界，采用双锁分离，吞吐量更高

- put 用 putLock
- take 用 takeLock
- count 用 CAS 控制

👉 生产和消费可以并行执行

put	队列满了会阻塞

offer	队列满了直接返回 false

## SynchronousQueue 是什么？为什么它适合配合 CachedThreadPool 使用

SynchronousQueue 是一个容量为 0 的阻塞队列，不存储元素，每个 put 操作必须等待 take 操作。\
它常用于线程之间的直接任务交接。\
在 CachedThreadPool 中，由于核心线程数为 0 且队列为 SynchronousQueue，任务无法排队，因此每来一个任务都会创建新线程执行，实现快速扩容，非常适合大量短期异步任务场景。

## ThreadPoolExecutor 的 execute() 流程源码逻辑是什么？

- 如果当前线程数小于核心线程数，则直接创建核心线程执行任务；
- 否则尝试将任务放入队列；
- 如果队列已满，则尝试创建非核心线程；
- 如果线程数达到最大且队列也满，则执行拒绝策略。

核心变量：

```plain
ctl = runState + workerCount
```

## 线程池的 ctl 为什么要把状态和线程数放在一个 int 里？

ThreadPoolExecutor 把线程池状态和线程数量压缩在一个 int 中，是为了通过一次 CAS 操作同时保证两者的一致性。\
如果使用两个变量，在并发情况下可能出现状态改变但线程数量未同步更新的竞态问题。\
这种位运算 + CAS 的设计是高并发组件常见的优化手段。

## 一个线程遍历 ArrayList，另一个线程写入导致 ConcurrentModificationException，如何改造并做性能取舍？

### 详细讲解整理

改造方案与取舍：

- 读多写少：优先 `CopyOnWriteArrayList`。遍历读快照、实现简单、不会触发该异常；代价是写时复制，写放大和内存开销较高。
- 写入频繁：使用 `ReentrantReadWriteLock + ArrayList`。遍历加读锁、修改加写锁；代价是锁竞争和代码复杂度上升。
- 临时止血：遍历前做快照 `new ArrayList<>(list)`，只遍历副本；代价是拷贝成本，不适合大集合高频调用。

落地建议：

- 先评估读写比、集合大小、写频率。
- 读写比高（如 90/10 以上）且集合中等规模可用 `CopyOnWriteArrayList`。
- 写频繁或集合较大时更建议读写锁方案或重构数据结构。

## 为什么双重检查锁（DCL）必须配合 volatile？去掉 volatile 会有什么风险？

### 详细讲解整理

核心结论：DCL 正确写法必须是 `private static volatile Singleton instance;`。否则可能发生“引用已发布但对象未初始化完成”的半初始化问题。

步骤拆解（无 volatile 的错误时序）：

1. 线程 A 进入 `getInstance()`，发现 `instance == null`。
2. 线程 A 进入同步块执行 `instance = new Singleton()`。
3. 这句底层可拆为：
   - 分配内存 `memory = allocate()`
   - 初始化对象 `ctor(memory)`
   - 赋值引用 `instance = memory`
4. 由于指令重排，可能变成：分配内存 -> 先赋值引用 -> 再初始化对象。
5. 线程 B 读取到 `instance != null`，直接返回该引用。
6. 线程 B 可能读到未初始化完成的对象状态（半初始化对象）。

为什么 `volatile` 能修复：

- `volatile` 写会建立内存屏障，禁止“发布引用”与“对象初始化”重排。
- `volatile` 还保证可见性：线程 B 读到非空引用时，能看到线程 A 构造后的最终状态。

正确写法示例：

```java
public class Singleton {
    private static volatile Singleton instance;

    private Singleton() {}

    public static Singleton getInstance() {
        if (instance == null) {
            synchronized (Singleton.class) {
                if (instance == null) {
                    instance = new Singleton();
                }
            }
        }
        return instance;
    }
}
```

## 你怎么设计一个可优雅关闭的线程池方案，做到尽量不丢任务且可观测？

核心流程（可落地）：

1. 进入 `DRAINING` 状态并摘流量，先拒绝新请求/新任务。
2. 调用 `shutdown()`，停止接收新任务，让队列与在途任务继续执行。
3. `awaitTermination(T1)` 分阶段等待（如 30\~120 秒），期间持续上报指标：`activeCount`、`queueSize`、完成任务数、关闭耗时。
4. 超时未结束时触发兜底：慢任务超时控制、可中断任务协作停止；再调用 `shutdownNow()`。
5. 对 `shutdownNow()` 返回的“未开始任务”做补偿（落库/MQ 重投/重试表），保证可追踪与重放。
6. 再次 `awaitTermination(T2)`，仍未结束则告警并输出诊断信息后退出。

关键保障：

- 任务实现要支持中断与超时，避免线程池无法收敛。
- 任务要幂等，便于失败重试与补偿。
- 关闭过程必须有可观测指标和日志，避免“看起来在关，实际卡死”。

常见误区：

- 直接 `shutdownNow()` 导致在途任务丢失。
- 吞掉中断信号导致线程池一直关不掉。
- 不先摘流量就关池，导致新任务持续进入形成死循环。

