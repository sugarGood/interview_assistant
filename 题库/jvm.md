# jvm
##  JVM 内存结构  
线程私有区域

+ **程序计数器（Program Counter Register）**
+ **虚拟机栈（Java Virtual Machine Stack）**
+ **本地方法栈（Native Method Stack）**

特点：

+ 每个线程独有
+ 生命周期和线程一致
+ 不存在数据共享问题

---

2️⃣ 线程共享区域

+ **堆（Heap）**
+ **方法区（Method Area）**
    - 在 JDK 8 之后由 **元空间（Metaspace）** 实现

##  Java 对象在 JVM 中是如何创建的？  
 Java 对象创建主要分为七步：  
类加载检查 → 堆内存分配 → 并发安全处理（CAS 或 TLAB） → 内存初始化 → 设置对象头 → 执行构造方法 → 返回对象引用。  



一、对象创建的触发

Java 对象的创建通常是通过 `**new**`** 关键字**触发，例如：

```plain
User user = new User();
```

当 JVM 执行到 `new` 指令时，就会开始对象创建流程。

---

二、类加载检查

在真正创建对象之前，JVM 会先检查：

+ **该类是否已经被加载**
+ **是否已经完成类加载、连接、初始化**

如果没有，就会先执行 **类加载过程**：

1. **加载（Loading）**
2. **验证（Verification）**
3. **准备（Preparation）**
4. **解析（Resolution）**
5. **初始化（Initialization）**

如果类已经加载完成，就直接进入下一步。

---

三、为对象分配内存

类加载完成后，JVM 会在 **堆（Heap）** 中为对象分配内存。

分配方式主要有两种：

1️⃣ 指针碰撞（Bump the Pointer）

适用于 **内存规整的堆**（例如使用 **Serial / ParNew GC**）。

原理：

```plain
| 已使用 | 未使用 |
          ↑
        指针
```

创建对象时，只需要把指针向后移动对象大小。

特点：

+ 速度快
+ 实现简单

---

2️⃣ 空闲列表（Free List）

适用于 **内存不规整的堆**（如 CMS）。

原理：

+ JVM 维护一个 **空闲内存列表**
+ 从列表中找到合适的空间分配对象

特点：

+ 适合碎片化内存

---

四、处理并发安全问题

因为 **多个线程可能同时创建对象**，JVM 需要保证线程安全。

常见两种方式：

1️⃣ CAS + 失败重试

使用 **CAS（Compare And Swap）**保证原子性。

2️⃣ TLAB（Thread Local Allocation Buffer）

每个线程在堆中分配一个 **TLAB 私有区域**：

+ 线程优先在 TLAB 中分配对象
+ 减少线程竞争

这是 JVM **最常用的优化方式**。

---

五、对象内存初始化

内存分配完成后，JVM 会：

1️⃣ **将对象内存初始化为零值**

例如：

```plain
int -> 0
boolean -> false
reference -> null
```

这样可以保证：

对象字段即使没有显式赋值，也有默认值。

---

六、设置对象头（Object Header）

JVM 会为对象设置 **对象头信息**，包括：

1️⃣ Mark Word

存储运行时数据：

+ hashCode
+ GC 分代年龄
+ 锁状态
+ 偏向锁信息

2️⃣ Klass Pointer

指向对象的 **类元数据（Class）**。

JVM 通过它知道：

+ 这个对象属于哪个类
+ 方法在哪里

---

七、执行对象初始化（构造方法）

最后一步：

执行 `**<init>**`** 构造方法**。

流程：

1. 调用父类构造方法
2. 执行实例变量赋值
3. 执行构造函数代码

例如：

```plain
new User();
```

最终会执行：

```plain
User.<init>()
```

---

八、返回对象引用

对象创建完成后：

+ JVM 返回 **对象引用（reference）**
+ 赋值给变量

```plain
User user = new User();
```

`user` 保存的是 **对象地址引用**，而不是对象本身。



##  JVM 中对象在堆里的 内存布局是什么样的？
在 HotSpot JVM 中，对象在堆中的内存布局主要包括三部分：对象头、实例数据和对齐填充。

对象头主要包含 Mark Word 和类型指针，Mark Word 存储对象运行时数据，比如 hashCode、GC 年龄、锁状态等；类型指针指向类元数据。

实例数据存储对象的成员变量。

最后为了保证对象大小是 8 字节的倍数，JVM 可能会进行对齐填充。



##  对象什么时候会进入 老年代？  
 1️⃣ 年龄达到晋升阈值    15

 2️⃣ 大对象直接进入老年代  

 3️⃣ Survivor 区放不下    对象会 **提前进入老年代**。  

动态对象年龄判定（很多人不知道）

JVM 有一个 **动态年龄机制**。

如果某个年龄段对象总大小：> Survivor 空间一半

 >= 该年龄的对象全部晋升老年代  



## 如何破坏双亲委派机制
双亲委派是 **约定的默认行为**（`ClassLoader.loadClass` 会先委派父加载器），要“破坏/绕过”它通常有几种做法：

1. **实现子优先（child-first）/自定义 ClassLoader**：覆盖 `loadClass`，先尝试自己加载（`findClass`/`defineClass`），加载失败再委派。最常见且直接。
2. **直接读取字节并 **`**defineClass**`：不走父委派，直接把 `.class` 字节转换成 `Class`。
3. **改变 JVM 启动参数（bootclasspath / --patch-module 等）**：把自定义类放到 bootstrap 搜索路径前面，从根本上替换核心类。需要 JVM 启动时设置。
4. **使用 Instrumentation / Java Agent**：在运行时通过字节码替换/重定义类（更复杂，受限制）。
5. **使用反射/Unsafe 操作**：操作类加载器内部或方法区（非常危险，不建议）。



### tomcat破坏双亲委派机制


在普通 Java 程序中，**所有类** 都由系统类加载器（`AppClassLoader`）加载。  
但是 Tomcat 不一样：

+ Tomcat 是一个 **Web 容器（Web Server）**，
+ 它可以同时运行多个 Web 应用（每个 war 包一个独立的应用），
+ 不同应用之间要做到：
    - **类相互隔离**（不能相互访问），
    - 但又要**共享一些公共类**（如 Servlet API）。

举例：

```plain
Tomcat 目录结构：
├── bin/          # Tomcat 自己的类
├── lib/          # Tomcat 运行时公共类（如 servlet-api.jar）
├── webapps/
│   ├── app1/WEB-INF/classes/...  # App1 的类
│   ├── app2/WEB-INF/classes/...  # App2 的类
```

👉 目标：

+ app1 和 app2 各自的类要 **隔离**。
+ 但它们都要能访问 Tomcat 的 servlet API。
+ 这就跟默认的 “全局单一类加载器” 机制冲突了



Tomcat 为了实现“每个 Web 应用独立”，  
**部分破坏了双亲委派机制**，采用了一种称为：

🌀 “Child-First”（子优先加载）策略。

也就是说：

当 WebApp 的类加载器加载类时，它会 **先尝试自己加载（WEB-INF 下的类）**，  
如果找不到，才交给父加载器（例如 Tomcat 自己的类、JDK 类）。



 但注意：Tomcat 并不是彻底取消父委派，它是 **选择性地破坏**。   为了实现多 Web 应用隔离，但又能共享 Tomcat 公共类。  



**Tomcat 破坏双亲委派机制的核心目的**：

+ 让每个 Web 应用有独立的类加载空间；
+ 防止应用之间相互干扰；
+ 同时仍能访问共享的 servlet API；
+ 所以选择“子优先加载”模型，而非标准“父优先”。
+ 

## Minor GC、Major GC、Full GC 的区别。  
Minor GC 是发生在新生代的垃圾回收，当 Eden 区满时触发，采用复制算法，速度较快。  
Major GC 是发生在老年代的垃圾回收，由于老年代对象存活率高，所以回收时间较长。  
Full GC 是对整个堆（新生代 + 老年代）以及方法区进行回收，停顿时间最长，对系统影响最大。  



##  新生代为什么不用标记整理？  
 新生代对象的特点是存活率非常低，大部分对象都是朝生夕灭。如果使用标记整理算法，需要扫描和整理整个区域，效率较低。而复制算法只需要复制少量存活对象，其时间复杂度与存活对象数量相关，因此在新生代中效率更高，STW 时间也更短，所以 JVM 在新生代采用复制算法。  



## Eden : S0 : S1 为什么是 8 : 1 : 1？
 新生代采用 Eden:S0:S1 = 8:1:1 的比例，是为了在使用复制算法时提高内存利用率。如果采用 1:1 的复制结构，会浪费 50% 的空间。而 8:1:1 的结构中，Eden 占 80%，两个 Survivor 各占 10%，Minor GC 时只需要把少量存活对象复制到 Survivor 区即可，因为新生代对象存活率很低，所以 10% 的 Survivor 空间通常足够，从而在保证复制效率的同时减少内存浪费。  



## JVM 是如何判断一个对象是否是“垃圾对象”的？也就是说，对象什么时候会被认定为可以回收？
 JVM 判断对象是否是垃圾对象主要有两种算法：引用计数法和可达性分析算法。引用计数法通过统计对象的引用次数来判断对象是否可以回收，但它无法解决循环引用问题，因此主流 JVM 并没有采用这种方式。HotSpot JVM 使用的是可达性分析算法，它从 GC Roots 出发，通过引用链搜索对象，如果一个对象无法从 GC Roots 到达，就会被认为是垃圾对象，可以被 GC 回收。  

GC Roots 是 **可达性分析的起点**。

主要包括：

栈中的局部变量、静态变量、 常量池引用、JNI 引用



## 有一个机制叫：对象自救（finalize 机制）你讲一下 对象被真正回收前会经历什么过程
 当一个对象在可达性分析中被判定为 GC Roots 不可达时，JVM 不会立刻回收，而是会经历两次标记。第一次标记后，JVM 会判断该对象是否需要执行 finalize 方法，如果对象重写了 finalize 且还没有执行过，就会被放入 Finalizer Queue，由 Finalizer 线程执行 finalize 方法。在 finalize 中对象可以重新与 GC Roots 建立引用实现自救。如果 finalize 执行后对象仍然不可达，就会在第二次标记后被真正回收。  



## Java 中除了 强引用 以外，还有 三种引用类型。请说出： 这三种引用是什么、它们在 GC 时的回收策略有什么不同。
 Java 中除了强引用以外，还有三种引用类型：软引用、弱引用和虚引用。软引用表示对象在内存充足时不会被回收，但在内存不足时会被 GC 回收，常用于实现缓存。弱引用表示对象只要发生 GC 就会被回收，典型应用是 ThreadLocalMap。虚引用不会影响对象生命周期，get 方法永远返回 null，主要配合 ReferenceQueue 使用，用于在对象被回收时收到通知。  

##  CMS 和 G1 的核心区别是什么？  
 CMS 和 G1 的核心区别在于设计思想和内存管理方式。CMS 的目标是减少停顿时间，主要针对老年代，使用标记清除算法，因此会产生内存碎片，并且可能出现 Concurrent Mode Failure 导致 Full GC。而 G1 采用 Region 化内存管理，把堆划分为多个 Region，通过优先回收垃圾最多的 Region 来控制停顿时间，并使用标记整理算法避免内存碎片，因此能够实现可预测的 GC 停顿时间。  



## CMS 的工作流程（四个阶段）
CMS 的回收过程分为 **四个阶段**：

```plain
1 初始标记
2 并发标记
3 重新标记
4 并发清除
```

---

### （1）初始标记（Initial Mark）
特点：

```plain
需要 STW
```

作用：

```plain
标记 GC Roots 直接引用的对象
```

例如：

```plain
GC Roots → A
GC Roots → B
```

这个阶段：

```plain
时间很短
```

---

### （2）并发标记（Concurrent Mark）
特点：

```plain
GC线程和用户线程并发执行
```

作用：

```plain
从 GC Roots 开始遍历对象图
```

标记所有：

```plain
可达对象
```

---

### （3）重新标记（Remark）
特点：

```plain
需要 STW
```

作用：

修正 **并发标记阶段产生的变化**。

因为在并发标记期间：

```plain
用户线程还在运行
```

对象引用关系可能发生变化。

所以需要：

```plain
重新标记
```

---

### （4）并发清除（Concurrent Sweep）
特点：

```plain
GC线程和用户线程并发执行
```

作用：

```plain
删除垃圾对象
```

因为使用的是：

```plain
Mark-Sweep
```

所以：

```plain
不会移动对象
```

---

## G1的流程
和cms前三步一样 第四步为筛选回收（Evacuation）

这是 **G1 最核心阶段**。

G1 会计算：每个 Region 的垃圾比例

然后：优先回收垃圾最多的 Region 这就是：Garbage First的来源。

回收方式：复制存活对象



## CMS 的问题
CMS 有三个典型问题：

### 1 内存碎片
因为：

```plain
标记清除
```

不会整理内存。

---

### 2 浮动垃圾（Floating Garbage）
在：

```plain
并发标记 / 并发清除
```

期间产生的垃圾：

```plain
本次 GC 无法回收
```

只能：

```plain
下次回收
```

---

### 3 Concurrent Mode Failure
如果：

```plain
老年代空间不足
```

CMS 来不及回收：

就会触发：

```plain
Full GC
```

##  G1 为什么能做到“可预测停顿时间”？  
 G1 之所以能够做到可预测停顿时间，是因为它把堆划分为多个 **Region**，Region 可以扮演不同角色：Ede、Survivor、Old 因此 G1 不再是固定的新生代/老年代结构,在 GC 时会根据每个 Region 的 **垃圾比例和回收成本** 进行评估，然后只选择部分 Region 组成 **回收集合（Collection Set）** 进行回收，从而控制每次 GC 的工作量，使停顿时间可预测  







##  为什么 CMS 会产生内存碎片，而 G1 不会？
 CMS 会产生内存碎片是因为它使用的是标记清除算法，只清理垃圾对象而不移动存活对象，因此会留下不连续的空闲空间。而 G1 在回收时会把存活对象复制到新的 Region 并进行整理，从而保证内存连续，因此不会产生内存碎片。  



## 在 G1 中，如果一个对象非常大（比如几十 MB），它会被放到哪里？
 在 G1 中，如果对象大小超过一个 Region 的 50%，就会被认为是 Humongous Object。这样的对象不会分配到普通的 Eden Region，而是直接分配到连续的 Humongous Region 中，这些 Region 属于 Old Generation。第一个 Region 被标记为 Humongous Start，后续为 Humongous Continue。Humongous Object 通常在并发标记周期或 Full GC 时回收  



## G1 为什么需要 Remembered Set（RSet）？
 G1 中每个 Region 都维护一个 Remembered Set，用于记录哪些其他 Region 的对象引用了当前 Region。当 G1 回收某个 Region 时，只需要扫描该 Region 的 RSet 中记录的 Region，而不需要扫描整个堆，从而大幅降低 GC 扫描范围，提高回收效率。  



## 假设你的 Java 服务出现：CPU 正常 但 Full GC 非常频繁 你一般会 如何排查？说一下你的 排查思路。
 如果 Java 服务出现 CPU 正常但 Full GC 非常频繁，我一般先通过 jstat 或 GC 日志确认 Full GC 的触发情况和 Old 区使用情况，然后分析 GC 日志判断触发原因，比如是否是老年代空间不足或 Metaspace 不足。如果怀疑内存泄漏，我会通过 jmap 导出 heap dump，再用 MAT 或 VisualVM 分析对象分布和 Dominator Tree，找出占用内存最多的对象以及 GC Root 引用链，最终定位到具体代码，比如缓存未清理、ThreadLocal 泄漏或集合无限增长等问题。  



第一步：先确认 GC 情况（看 GC 日志）   jstat -gcutil <pid> 1s

重点关注：

+ **FGC（Full GC 次数）**
+ **FGCT（Full GC 时间）**
+ **Old 区使用率**
+ 如果看到：FGC 持续增加

Old 区很快被占满

**说明：老年代回收压力大**

****

第二步：查看 GC 日志  

 如果开启了 GC 日志：可以查看：Full GC 的触发原因

 例如日志可能出现：  Allocation Failure  说明：  老年代空间不足

 GC 前后内存变化   Full GC (Allocation Failure)  Old: 2GB -> 1.9GB 回收效果非常差 内存泄漏



第三步：查看堆内存对象分布  

如果怀疑内存泄漏，可以导出 **堆 Dump**。

常用命令： jmap -dump:format=b,file=heap.hprof <pid>

然后用工具分析：

常用工具：

+ **MAT (Memory Analyzer Tool)**
+ **VisualVM**
+ **JProfiler**

重点看：

1 哪些对象占用最多内存

例如：Top Consumers 

查看：对象类型、占用大小、实例数量

2 是否存在对象一直增长

例如： HashMap、ArrayList、缓存对象

如果对象数量：持续增长

可能是：内存泄漏



 第四步：分析对象引用链  

在 MAT 中查看：Dominator Tree

 找出：  GC Root → 对象引用链

 确认：  为什么对象没有被释放

常见问题：

+ 静态集合缓存未清理
+ ThreadLocal 未释放
+ Listener 未注销
+ 缓存设计问题

##  为什么 Java 中： String s = new String("abc"); 会创建 两个对象？ 分别在哪些内存区域？  
`String s = new String("abc");` 会创建两个对象。首先字符串字面量 "abc" 会在字符串常量池中创建一个对象，如果常量池中已经存在则直接使用。然后 `new String("abc")` 会在堆中创建一个新的 String 对象，并引用常量池中的字符串内容。变量 s 指向的是堆中的对象，因此总共会有两个对象  



## Strings=newString("abc").intern(); 会创建几个对象？  
这段代码通常会创建 **1 或 2 个对象**，取决于字符串常量池中是否已经存在 `"abc"`。

更具体：

+ 如果常量池 **没有 **`**"abc"**` → 创建 **2 个对象**
+ 如果常量池 **已经有 **`**"abc"**` → 只创建 **1 个对象**

