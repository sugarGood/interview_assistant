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

synchronized 是基于 JVM 的 Monitor 实现的，通过 monitorenter/monitorexit 指令控制，

每个对象都有一个对象头，在最初只有一个线程，没有竞争的时候 使用 CAS 将将将线程 ID将将 设置到对象的 Mark Word 头里 之后锁重入时 发现这个线程 ID是自己的就表示没有竞争，不用重新 CAS。 这是偏向锁,

- 撤销偏向需要将持锁线程升级为轻量级锁，这个过程中所有线程需要暂停（STW）
- 偏向锁在现代多核高并发场景下收益很低，但维护成本很高，所以在 JDK 15 被默认关闭，并在 JDK 18 彻底移除。

如果出现竞争，会升级成轻量级锁，此时会通过cas将对象头中的markword替换成 当前线程LockRecord的指针

如果替换成功，则加锁成功 替换失败  会开始自旋转尝试

失败一定次数之后就会升级成重量级锁，这时就会用到monitor对象，对象的 markWord指向重量级锁地址，当前对象进入entryList阻塞

此时当持有锁的退出同步块解锁时，使用cas将Mark Word 的值恢复给对象头，会失败。这时会进入重量级解锁流程，即按照Monitor地址找到Monitor对象，设置Owner为null，唤醒EntryList中 BLOCKED线程。

，

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

在jdk8里面 concurrentHashMap采用cas+synchronized 、 node数组:链表+红黑树  实现 无锁读 细粒度锁写 协作式扩容

get流程

1.计算 hash  2. 定位桶 index  3. 读取桶头节点

1. 判断三种情况：

- 头节点就是目标
- 红黑树 O(log n) 查找
- 链表遍历 线性遍历链表

4.返回结果

<br />

线程A正在扩容\
线程B正在 get()

流程：

1. 发现桶是 ForwardingNode
2. 调用 find()
3. 自动跳转到新 table
4. 查找成功

<br />

put流程

1. 计算 hash
2. 检查 table 是否初始化
3. 定位桶
4. 判断 4 种情况：

- 未初始化
- 桶为空（CAS插入）
- 扩容中 
  - 检测 ForwardingNode 判断当前桶是否已迁移：如果已迁移则转到新 table 继续操作；如果未迁移则在旧 table 中加锁插入
- 桶非空（加锁） 链表 or 红黑树插入

5.判断是否需要树化\
6.判断是否需要扩容

<br />

扩容流程

ConcurrentHashMap 的扩容不是单线程重建，而是通过 **创建新数组 + ForwardingNode 标记 + 多线程分段迁移（helpTransfer）** 实现的“并发扩容机制”，保证扩容过程中读写都不阻塞。

扩容触发条件

1. put 时触发
   size > threshold（容量 \* 负载因子）
2. 链表过长也可能触发（树化前）
   链表长度 >= 8 → treeify or resize

扩容核心变量

Node\<K,V>\[] nextTable; // 新数组

int transferIndex; // 迁移进度

<br />

扩容整体流程

1. 线程触发 resize
2. 创建 newTable（2倍）
3. 设置 transferIndex = oldTable.length
4. 每个线程分段搬迁
   1. 迁移单个桶:
      1. 判断是否已迁移
      2. 否则加锁搬迁
      3. 拆分链表（非常关键）
      4. 写入 newTable
5. 用 ForwardingNode 标记旧桶  oldTable\[i] = ForwardingNode  告诉其他线程：这里已经搬走了
6. 所有线程都可能参与：put / get / compute
   → 发现扩容
   → 帮忙搬
7. 所有桶迁移完成  transferIndex == 0
8. 替换 table  table = nextTable   nextTable = null

<br />

jdk1.7

是segment 分段锁

把一个大 Map 拆成多个小 Map，每个小 Map 一把锁（Segment锁）

Map

└─ Segment （一个“带锁的 HashMap”）

   └─ HashEntry\[]

      └─ 链表

扩容机制 是

- 只允许单线程扩容
- 锁住整个 Segment
- 扩容时其他线程阻塞

<br />

## 为什么concurrentHashMap get 为什么不加锁？

因为node数组是volatile volatile 保证可见性（最核心）

CAS + synchronized 只用于写，不影响读

Node 在插入时是：**完整构造后一次性发布**

<br />

## 为什么concurrentHashMap锁桶头节点就够了

因为 ConcurrentHashMap 在桶内操作时，所有对链表/红黑树的修改都必须通过“桶头节点作为入口”，并且桶内结构是**串行修改 + 可见性保证 + 不允许并发结构性破坏**，所以只要锁住桶头节点，就能天然保证整个桶的线程安全。

## 为什么 ConcurrentHashMap 不允许 null key 和 null value？

ConcurrentHashMap 为了保证 get() 返回 null 只表示 key 不存在，

在并发场景下避免语义歧义，

所以禁止存 null key 和 null value。

否则 containsKey + get 在并发下会产生逻辑错误。

<br />

## Segment 为什么用 ReentrantLock 而不是 synchronized

JDK7 中 Segment 选择 ReentrantLock 而不是 synchronized，是因为当时 synchronized 性能较弱且缺乏 tryLock、超时、中断等能力，而 ReentrantLock 基于 AQS 提供了更灵活的锁控制机制，更适合分段锁这种高并发可控模型。

<br />

## Segment 为什么最终被废弃

Segment 被废弃的核心原因是其分段锁模型存在固定并发上限（默认16段）、无法动态扩展并发粒度、热点分布不均导致性能瓶颈，以及结构复杂和内存开销较大，而 JDK8 通过 CAS + 桶级锁实现了基于哈希天然分布的细粒度并发模型，从根本上解决了并发扩展性问题。

## LongAdder 为什么比 AtomicLong 性能好

AtomicLong 所有线程竞争同一个变量，CAS 冲突严重。\
LongAdder 通过分段累加，把竞争分散到多个 Cell，\
减少 CAS 失败和缓存行冲突，\
所以在高并发下性能更好。

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

## 什么是AQS

AQS 是 JDK 提供的一个用于构建锁和同步器的基础框架，通过一个 volatile state 配合 CAS 维护同步状态，并使用 FIFO 队列管理等待线程，从而统一解决线程竞争、排队、阻塞和唤醒等并发控制问题，简化了锁和同步器的实现。

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
