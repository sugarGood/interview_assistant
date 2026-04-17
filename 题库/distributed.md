# 分布式

## RPC 框架一次调用的完整链路是什么？哪些点最容易出问题？

### 标准面试回答

RPC 调用链路可以概括为：服务发现 -> 负载均衡 -> 建连与编解码 -> 调用超时/重试 -> 熔断限流 -> 返回结果。

高风险点主要有三类：

1. 超时设置不合理，导致线程堆积和重试风暴。
2. 重试未做幂等控制，导致重复写或重复扣款。
3. 序列化协议升级不兼容，出现反序列化失败。

### 详细讲解

面试时要体现“全链路视角”，不是只说网络层。\
一次调用失败可能是任一环节的问题：注册中心返回实例异常、负载均衡把流量打到慢节点、连接池耗尽、重试策略放大故障。

治理重点：

- 设置分层超时（连接超时、请求超时、总超时）。
- 限制重试次数并加退避策略。
- 序列化版本要可回滚，避免灰度期间双向不兼容。

## MQ 为什么能削峰填谷？它引入了哪些一致性问题？

### 标准面试回答

MQ 削峰填谷的本质是“把瞬时高并发同步请求，变成可控速率的异步消费”，从而保护下游系统。

但它会引入一致性问题：

1. 生产成功但消费失败，业务状态不一致。
2. 至少一次语义下可能重复消费。
3. 并发分区消费会带来顺序性问题。

常见治理方案：

- 幂等消费（业务唯一键 + 去重逻辑）。
- 事务消息或本地消息表。
- 重试队列、死信队列、补偿任务。

### 详细讲解

MQ 本质是“吞吐优先”的系统，天然偏向最终一致。\
所以面试里要讲清楚：你不能指望 MQ 自动保证业务一致，必须在业务层设计幂等、补偿和状态对账。

尤其是核心交易场景，要补上“对账任务 + 人工兜底”，否则长尾异常会长期沉淀。

## 分布式事务常见方案（2PC、TCC、Outbox+MQ、Saga）怎么选？在企业里通常怎么落地？

### 标准面试回答

分布式事务没有银弹，核心是在“一致性、性能、复杂度、业务侵入性”之间做取舍：

- 2PC/XA：一致性最强，但阻塞重、吞吐差，适合低并发短链路。
- TCC：把事务控制上移到业务层，适合余额、额度、券冻结这类高价值资源预留场景。
- Outbox + MQ：互联网最常见，依靠“本地事务 + 事件驱动 + 消费幂等”实现最终一致，适合高并发主链路。
- Saga：适合步骤多、链路长的业务流程，通过补偿动作保证最终一致。

企业里通常不是四选一，而是组合使用：高并发主链路用 Outbox+MQ 或 Saga，关键资金步骤局部用 TCC，2PC/XA 只放在低并发强一致场景。

### 详细讲解

先统一一个前提：分布式事务本质是在解决“一个业务动作跨多个服务/数据库时，如何让整体结果正确”。\
例如下单链路可能同时涉及：

- 订单服务写订单库
- 库存服务扣库存
- 账户服务扣余额
- 营销服务扣优惠券

这时已经不是单库本地事务能解决的问题，需要在多个系统之间设计一致性策略。

#### 一、2PC / XA

核心思想：

- 第一阶段 `prepare`：参与者先执行事务并锁住资源，但不真正提交。
- 第二阶段 `commit/rollback`：协调者根据所有参与者返回结果决定统一提交或统一回滚。

核心角色：

- 协调者（Coordinator）：统一发起和决策事务。
- 参与者（Participant / Resource Manager）：真正执行本地事务的数据库或服务。

流程图：

```text
客户端
  |
  v
协调者
  |------------------> 订单库 prepare
  |------------------> 账户库 prepare
  |------------------> 库存库 prepare
  |
  |<------------------ 全部返回 YES
  |
  |------------------> 订单库 commit
  |------------------> 账户库 commit
  |------------------> 库存库 commit
  |
  v
事务成功
```

如果某个参与者在 prepare 阶段失败：

```text
客户端
  |
  v
协调者
  |------------------> 订单库 prepare -> YES
  |------------------> 账户库 prepare -> NO
  |------------------> 库存库 prepare -> YES
  |
  |------------------> 订单库 rollback
  |------------------> 账户库 rollback
  |------------------> 库存库 rollback
```

典型使用场景：

- 财务后台小流量任务
- 跨两个库必须“同成同败”的短链路写入
- 对一致性要求极高，但并发量不大

项目中的实现方式：

- 数据库开启 XA 能力
- 应用接入 XA 数据源
- 使用事务协调器统一管理 prepare/commit

Java 侧常见写法（表面接近本地事务，底层由 XA 协调器接管）：

```java
@Transactional
public void transfer() {
    orderRepository.insert(...);     // DB1
    accountRepository.decrease(...); // DB2
}
```

企业常见方案：

- `Atomikos`
- `Narayana`
- 传统 JTA/XA 中间件

优点：

- 强一致语义最直接
- 对业务补偿逻辑要求低

缺点：

- 参与者在 prepare 后会阻塞等待，资源被锁住
- 高并发下吞吐差、锁冲突重、协调者压力大
- 不适合下单、秒杀这类主交易链路

#### 二、TCC（Try / Confirm / Cancel）

核心思想：

- `Try`：检查资源并预留资源
- `Confirm`：真正提交
- `Cancel`：失败后释放预留资源

关键概念：

- 预留资源：不是立刻最终扣减，而是先把资源冻结/占住
- 业务层事务：事务逻辑从数据库层上移到业务层控制

以“扣余额”为例：

- Try：可用余额减，冻结余额加
- Confirm：冻结余额真正扣掉，记正式流水
- Cancel：把冻结余额释放回可用余额

流程图：

```text
用户下单
  |
  v
事务协调器
  |------------------> 账户服务 Try: 冻结余额
  |------------------> 库存服务 Try: 预占库存
  |------------------> 券服务   Try: 锁券
  |
  |<------------------ 全部成功
  |
  |------------------> 账户服务 Confirm
  |------------------> 库存服务 Confirm
  |------------------> 券服务   Confirm
  |
  v
事务成功
```

如果某一步 Try 失败：

```text
用户下单
  |
  v
事务协调器
  |------------------> 账户服务 Try: 成功
  |------------------> 库存服务 Try: 成功
  |------------------> 券服务   Try: 失败
  |
  |------------------> 账户服务 Cancel
  |------------------> 库存服务 Cancel
```

典型使用场景：

- 余额扣减
- 授信额度占用
- 优惠券锁定/核销
- 高价值资源的预留与确认

项目中的实现方式：

- 每个服务提供 `Try / Confirm / Cancel` 三个接口
- 使用事务状态表记录 `xid + branchId + status`
- 全链路传递全局事务号 `xid`

示例 SQL（账户余额）：

```sql
-- Try: 预留资源
UPDATE account
SET available_amount = available_amount - #{amt},
    frozen_amount = frozen_amount + #{amt}
WHERE user_id = #{userId}
  AND available_amount >= #{amt};

-- Confirm: 正式扣减
UPDATE account
SET frozen_amount = frozen_amount - #{amt}
WHERE user_id = #{userId};

-- Cancel: 释放预留
UPDATE account
SET frozen_amount = frozen_amount - #{amt},
    available_amount = available_amount + #{amt}
WHERE user_id = #{userId};
```

Java 伪代码：

```java
public boolean tryDebit(String xid, Long userId, BigDecimal amt) {
    // 幂等判断 xid
    // available -= amt, frozen += amt
    // 记录状态 TRY_OK
    return true;
}

public boolean confirm(String xid) {
    // 幂等：已确认直接成功
    // frozen -= amt
    // 写扣款流水
    return true;
}

public boolean cancel(String xid) {
    // 空回滚：Try 没执行也可安全返回
    // 防悬挂：Cancel 后晚到的 Try 必须拒绝
    // frozen -= amt, available += amt
    return true;
}
```

三个必须会的概念：

- 幂等：重复 Confirm/Cancel 不能重复扣减
- 空回滚：Try 没执行，Cancel 先到也要安全返回
- 防悬挂：Cancel 执行后，延迟到达的 Try 不能再占资源

企业常见方案：

- `Seata TCC`
- 自研 TCC 框架
- 金融/支付类系统的定制事务协调框架

优点：

- 一致性比纯最终一致更可控
- 适合高价值资源

缺点：

- 业务侵入最大
- 每个参与服务都要改造三套接口和状态表
- 设计不当容易被幂等、空回滚、防悬挂问题击穿

#### 三、Outbox + MQ

这是互联网最常见的方案，核心目标是解决“业务写库成功，但消息丢失”问题。

为什么需要它：

- 如果先写业务表，再发 MQ，可能出现业务成功但消息没发出去
- 如果先发 MQ，再写业务表，可能出现消息发出但业务回滚

核心思想：

- 在同一个本地事务里同时写入：
  - 业务表
  - 本地消息表（Outbox）
- 事务成功后，再异步把 Outbox 里的消息投递到 MQ

流程图：

```text
用户下单
  |
  v
订单服务
  |
  |-- 本地事务开始
  |   1. 写 orders
  |   2. 写 outbox_events(order_created)
  |-- 本地事务提交
  |
  v
Outbox 投递器
  |
  |------------------> MQ
  |------------------> 标记消息已发送
  |
  v
库存服务 / 营销服务 / 通知服务 消费消息
```

项目中的表设计：

`orders`

```text
id
order_no
user_id
amount
status
create_time
```

`outbox_events`

```text
id
event_type
biz_id
payload
status
retry_count
next_retry_time
create_time
```

下单本地事务代码：

```java
@Transactional
public void createOrder(CreateOrderCommand cmd) {
    Order order = new Order(...);
    orderRepository.insert(order);

    OutboxEvent event = new OutboxEvent();
    event.setEventType("ORDER_CREATED");
    event.setBizId(order.getId().toString());
    event.setPayload(toJson(order));
    event.setStatus("NEW");
    outboxRepository.insert(event);
}
```

投递器代码：

```java
public void publishOutbox() {
    List<OutboxEvent> events = outboxRepository.findNewEvents(100);
    for (OutboxEvent event : events) {
        try {
            mqProducer.send("order-topic", event.getBizId(), event.getPayload());
            outboxRepository.markSent(event.getId());
        } catch (Exception e) {
            outboxRepository.incrRetry(event.getId());
        }
    }
}
```

消费端幂等处理：

```java
public void onOrderCreated(Event event) {
    if (consumeLogRepository.exists(event.getBizId(), "STOCK_DEDUCT")) {
        return;
    }

    stockRepository.deductIfEnough(event.getSkuId(), event.getNum());

    consumeLogRepository.insert(event.getBizId(), "STOCK_DEDUCT");
}
```

典型使用场景：

- 下单后异步扣库存
- 发积分、发优惠券、发通知
- 高并发主链路的跨服务协同

企业常见实现方式：

- 本地消息表 + 定时扫描投递
- `RocketMQ` 事务消息
- `Kafka + Outbox + Debezium CDC`
- `RabbitMQ + 本地消息表`

为什么消费端必须幂等：

- MQ 普遍是“至少一次投递”
- 消息可能重复
- 没有幂等，重试越多数据越乱

优点：

- 性能好，吞吐高
- 非常适合互联网主交易链路
- 改造成本比 TCC 低

缺点：

- 只能保证最终一致
- 有短暂不一致窗口
- 必须补齐重试、死信、对账、人工补偿

#### 四、Saga

Saga 适合“步骤多、链路长、不能长时间锁资源”的场景。

核心思想：

- 每一步先作为独立本地事务提交
- 如果后面某步失败，就执行前面已成功步骤的补偿动作

不是“数据库回滚”，而是“业务补偿”。

示例场景：

- 旅游订单：创建订单 -> 占库存 -> 锁优惠券 -> 冻结余额 -> 创建履约
- 如果冻结余额失败，就逆序补偿：
  - 解锁优惠券
  - 释放库存
  - 取消订单

流程图：

```text
正向流程：
创建订单 -> 预占库存 -> 锁优惠券 -> 冻结余额 -> 创建履约

如果第 4 步失败：
解锁优惠券 <- 释放库存 <- 取消订单
```

更标准的表达：

```text
Step1: createOrder      补偿: cancelOrder
Step2: reserveStock     补偿: releaseStock
Step3: lockCoupon       补偿: unlockCoupon
Step4: freezeBalance    补偿: unfreezeBalance
```

项目中的实现方式：

- 定义状态机：每一步的正向动作 + 补偿动作
- 记录 Saga 事务状态表
- 失败时按逆序触发补偿

Java 伪代码：

```java
saga.start("create-order")
    .step("createOrder", this::createOrder, this::cancelOrder)
    .step("reserveStock", this::reserveStock, this::releaseStock)
    .step("lockCoupon", this::lockCoupon, this::unlockCoupon)
    .step("freezeBalance", this::freezeBalance, this::unfreezeBalance)
    .execute();
```

Saga 有两种常见实现方式：

1. 编排式（Orchestration）

- 用一个中心编排器推进流程
- 优点是流程清晰、便于监控和排错

1. 协同式（Choreography）

- 靠事件流让各服务自行推进
- 解耦好，但链路长后排查困难

企业里复杂流程通常偏向编排式。

企业常见方案：

- `Seata Saga`
- `Temporal`
- `Camunda`
- `Netflix Conductor / Orkes`
- 自研状态机编排平台

典型使用场景：

- 订单、履约、营销、支付协同的长流程
- 跨多个业务域的流程编排
- 允许中间态，但要求最终能补偿收敛

优点：

- 没有全局锁，吞吐高
- 适合长事务

缺点：

- 一致性是最终一致
- 补偿动作设计复杂
- 有些操作天然不可逆，比如短信已发、外部回调已成功

#### 五、企业里通常怎么组合用？

真实项目里很少“只选一种”：

- 下单主链路：`Outbox + MQ`
- 复杂履约流程：`Saga`
- 余额、额度、券冻结：局部 `TCC`
- 低频强一致后台任务：少量 `2PC/XA`

一个典型电商下单链路可能是：

```text
下单主流程：Outbox + MQ
余额扣减：TCC
履约长流程：Saga
后台对账修复：定时任务 + 人工补偿
```

#### 六、除了事务模型，还必须补哪些工程能力？

如果只会讲事务模型，不讲这些，面试官会认为你只懂概念：

1. 幂等

- 同一个请求执行多次，结果只能生效一次
- 常见做法：幂等键、唯一索引、幂等表、状态机

1. 重试

- 临时失败后自动重试
- 前提是业务必须幂等

1. 死信队列

- 多次重试仍失败的消息进入死信队列
- 避免阻塞主消费链路

1. 对账

- 定期比对订单、库存、余额、补偿状态是否一致
- 是最终一致方案的最后保险

1. 人工补偿

- 长尾异常不能只靠自动程序收敛
- 必须有可查询、可重放、可人工修复的后台工具

#### 七、面试里如何做最后收口？

你可以这样总结：

```text
2PC/XA 适合低并发强一致短链路；
TCC 适合高价值资源预留；
Outbox + MQ 是互联网高并发主链路最常见方案；
Saga 适合长流程编排。

企业里通常是组合使用：主流程最终一致，关键步骤局部强一致，
再配合幂等、重试、死信、对账和人工补偿，确保异常可追踪、可修复。
```

## 如何设计幂等系统，避免重复提交和重复消费？

### 标准面试回答

幂等设计的核心是“同一业务请求重复执行，结果只生效一次”。

常用手段：

1. 幂等键 + 唯一索引（最常见）。
2. 状态机约束（非法状态迁移直接拒绝）。
3. 去重表/去重缓存（带 TTL）。

### 详细讲解

幂等不是单点功能，而是贯穿“入口、存储、消费”全链路：

- 入口层：请求必须带稳定幂等键。
- 存储层：唯一约束兜底，防止并发穿透。
- 消费层：消费重试时按幂等键判断是否已处理。

线上排查重复问题时，优先查三件事：幂等键是否一致、唯一约束是否生效、消费位点是否回退。

## 一致性哈希解决了什么问题？有哪些工程细节？

### 标准面试回答

一致性哈希主要解决“节点变化时数据迁移量过大”的问题。\
相比普通取模，一致性哈希在扩缩容时只迁移环上相邻区间的数据。

工程细节：

1. 使用虚拟节点降低数据倾斜。
2. 节点上下线做平滑迁移，避免流量抖动。
3. 配合热点 key 识别，防止局部热点打爆节点。

### 详细讲解

一致性哈希不是“自动均衡”，它只是“减少迁移”。\
如果虚拟节点配置不合理，仍会出现倾斜。若热点集中在少量 key，上环算法再好也会产生热点节点。

所以落地时要配合：

- 热点探测与主动打散。
- 分片副本与多级缓存。
- 扩容过程的限速迁移与观测告警。

