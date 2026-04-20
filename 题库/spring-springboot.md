# spring/springboot

## Spring 框架的核心是什么

Spring 框架的核心主要是 IoC（控制反转）和 AOP（面向切面编程）。 其中 IoC 是最核心的思想，它通过 Spring 容器来管理对象的创建和对象之间的依赖关系，开发者不需要自己去 new 对象，而是通过依赖注入的方式让容器自动完成，从而降低系统的耦合度，提高代码的可维护性。而 AOP 则是用来处理一些与业务无关但又需要统一处理的功能，比如日志记录、事务管理和权限控制等，通过在不修改业务代码的情况下将这些功能统一织入到方法执行过程中，从而提高代码的复用性和系统的扩展性

## Spring 中依赖注入（DI）有哪几种方式？分别有什么区别？



Spring 中依赖注入主要有三种方式：构造器注入、Setter 注入和字段注入。

&#x20;构造器注入是通过类的构造方法将依赖对象传入，这种方式可以保证对象在创建时依赖已经完整，安全性较高，也更利于单元测试；Setter 注入是通过 setter 方法为对象设置依赖，灵活性较高，适合可选依赖的情况；字段注入通常是通过 `@Autowired` 直接作用在成员变量上，由 Spring 自动完成注入，写法最简洁，但不利于测试，也不方便体现依赖关系。在实际开发中，一般般般推荐使用构造器注入般般，因为它更加规范、依赖关系清晰，也更符合面向对象设计原则

## 请你说一下 Spring Bean 的生命周期。

Spring Bean 的生命周期主要包括   实例化、依赖注入、初始化、使用、销毁五个阶段  。在初始化过程中 Spring 提供了多个扩展点，例如 `Aware 接口`、`BeanPostProcessor`、`@PostConstruct` 等，用于在 Bean 创建前后进行增强或定制逻辑。其中 `BeanPostProcessor` 是 Spring 最重要的扩展机制之一，AOP、事务管理等核心功能都依赖于这个阶段实现代理对象的创建。

<!-- 这是一张图片，ocr 内容为： -->

![](https://cdn.nlark.com/yuque/0/2026/png/22451783/1773644729004-71ab62b5-055d-4df2-b44a-c40d2d9a6cb4.png)

首先 Spring 根据 BeanDefinition 创建 Bean 实例，然后进行属性填充完成依赖注入。

如果 Bean 实现了 Aware 接口，比如 BeanNameAware 或 ApplicationContextAware，Spring 会回调对应方法让 Bean 获取容器相关信息。

接着会执行 BeanPostProcessor 的 postProcessBeforeInitialization 方法。

然后进入初始化阶段，可能执行 @PostConstruct、InitializingBean 的 afterPropertiesSet 方法或者自定义 init-method。

初始化完成后会执行 BeanPostProcessor 的 postProcessAfterInitialization 方法，很多功能例如 Spring AOP、事务管理就是在这个阶段创建代理对象。

最后 Bean 就可以被容器正常使用。当 Spring 容器关闭时，会执行 @PreDestroy、DisposableBean 的 destroy 方法或者 destroy-method 完成 Bean 的销毁

## Spring 是如何解决循环依赖的？

Spring 通过三级缓存机制，解决“单例 + 字段/Setter 注入”场景下的循环依赖。

核心流程是：先实例化 Bean（未完成注入和初始化），再把该 Bean 的 `ObjectFactory` 放入三级缓存；如果后续出现反向依赖请求（如 `A -> B -> A`），就从三级缓存拿工厂调用 `getObject()` 生成 A 的早期引用，先完成依赖注入，再继续完整初始化。

三级缓存分别是：

- 一级缓存 `singletonObjects`：完整初始化后的成品 Bean
- 二级缓存 `earlySingletonObjects`：已经生成好的早期引用
- 三级缓存 `singletonFactories`：用于按需生成早期引用的 `ObjectFactory`

为什么一定要第三级：二级缓存存的是“结果对象”，三级缓存存的是“生成逻辑”。Spring 可以在“真正有人来拿早期引用”时再决定返回原始对象还是代理对象（`getEarlyBeanReference()`），避免过早决策和引用不一致问题。

注意边界：

- 只对单例 Bean 的字段/Setter 循环依赖有效
- 构造器循环依赖通常无解（实例化前就需要完整依赖）
- `prototype` 循环依赖通常无解
- 不发生循环依赖时（如单向 `B -> A`），一般不会使用早期引用，B拿到的是A的最终初始化结果

### 详细讲解整理（三级缓存与延迟决策）

很多人会问：既然二级缓存也能放早期对象，为什么还要三级缓存？

核心在于：二级缓存是“结果缓存”，三级缓存是“生成逻辑缓存”。

- `singletonObjects`（一级）：完整初始化后的成品 Bean
- `earlySingletonObjects`（二级）：已经生成好的早期引用
- `singletonFactories`（三级）：`ObjectFactory`，用于按需生成早期引用

Spring 的设计是先把“如何生成早期引用”的能力放在三级缓存，只有在**真的发生反向依赖请求**时才调用 `getObject()`，并在这一刻决定返回“原始对象”还是“代理对象”。这就是延迟决策。

如果只用二级缓存并在创建时就拍板，容易出现两类问题：

- 过早决策：还没发生实际依赖请求就提前做代理判断
- 引用不一致：依赖方拿到原始对象，但最终容器里是代理对象

因此三级缓存的价值不是“不能用二级实现”，而是把“按需生成、一次生成、统一入口”这件事工程化，避免过早暴露和分散分支逻辑。

另外一个常见误区：不发生循环依赖时，依赖方会不会拿到早期引用？

- 单向依赖（如 `B -> A`）通常不会拿到早期引用，B注入的是A的最终初始化结果
- 早期引用只在“创建中的 Bean 被反向请求”时才会使用（典型是 `A <-> B`）
- 三级缓存中的条目是创建期间的临时数据，Bean 完成后会清理，不会常驻

### 30秒背诵稿

Spring 通过三级缓存解决单例 Setter/字段注入的循环依赖：一级放成品 Bean，二级放早期引用，三级放 `ObjectFactory`。发生 `A -> B -> A` 时，B需要A会从三级工厂按需生成早期引用，先完成注入，再走完整初始化。第三级的意义是延迟决策：在真正被依赖时再判断返回原始对象还是代理对象，避免过早决策和引用不一致。构造器循环依赖和 `prototype` 循环依赖通常无解。

### 2分钟展开稿

先说结论：Spring 不是“无条件解决所有循环依赖”，它只在“单例 + 字段/Setter 注入”这个窗口可解。

流程上，Bean 会先实例化，再注入属性、执行初始化。实例化后 Spring 会把该 Bean 的 `ObjectFactory` 放入三级缓存，表示“如果后面有人反向依赖我，可以按需生成我的早期引用”。

当出现 `A -> B -> A` 时，B在注入A阶段会触发这条路径：先查一级成品缓存，没有就查二级早期引用，再从三级工厂 `getObject()` 生成早期A，注入B后继续完成后续初始化。

三级缓存的设计重点在“结果”和“生成逻辑”分离：二级是已经生成的早期对象，三级是生成规则。这样可以把“是否要代理（如AOP事务）”延迟到真正被请求时再决定，避免过早暴露原始对象导致依赖方拿到的引用与最终容器对象不一致。

最后说边界：构造器循环依赖因为实例化前就要完整依赖，通常解不了；`prototype` 作用域也通常不支持这套提前暴露机制；如果只是单向依赖（如 `B -> A`），通常拿到的是A的最终初始化结果，不会走早期引用。

## Spring AOP 为什么默认使用 JDK 动态代理和 CGLIB？两者有什么区别？

Spring AOP 通过动态代理实现方法增强，主要支持 JDK 动态代理和 CGLIB 动态代理两种方式。

JDK 动态代理是基于接口实现的，通过 `java.lang.reflect.Proxy` 在运行时生成代理对象，并通过 `InvocationHandler` 对方法调用进行拦截。

CGLIB 是基于继承实现的，它会在运行时生成目标类的子类，并通过方法拦截器对目标方法进行增强。

Spring 默认的选择策略是：如果目标类实现了接口，则优先使用 JDK 动态代理；如果没有实现接口，则使用 CGLIB。

两者的主要区别在于：

- JDK 动态代理必须基于接口
- CGLIB 基于继承，因此不能代理 final 类和 final 方法

在 Spring Boot 2.x 之后，默认配置 `spring.aop.proxy-target-class=true`，即默认使用 CGLIB 代理，这样可以统一代理行为，减少接口限制。

## Spring Boot 的自动配置（AutoConfiguration）是如何实现的？

Spring Boot 自动配置的核心是通过 `@EnableAutoConfiguration` 启动自动配置机制，该注解通过 `AutoConfigurationImportSelector` 使用 `SpringFactoriesLoader` 读取 `META-INF/spring.factories`（Spring Boot 3 中为 `AutoConfiguration.imports`）文件中的所有自动配置类，然后根据 `@Conditional` 条件注解判断当前环境是否满足条件，满足条件才会将对应 Bean 注册到 Spring 容器中。通过这种   SPI + 条件装配   的机制，实现了 Spring Boot 引入依赖即可自动完成大量配置的能力。

## 如果我想自己写一个 Spring Boot Starter，需要做哪些步骤？

<br />

## 为什么同类方法调用会导致 @Transactional 失效？怎么修复并做线上排查？
`@Transactional` 本质依赖 Spring AOP 代理（JDK 动态代理或 CGLIB）在方法调用前后织入事务边界：进入方法前开启事务，正常返回提交，异常按规则回滚。  
同类方法内调用（`this.xxx()`）绕过了代理对象，直接调用目标对象本身，因此事务拦截器不会执行，表现为注解“看起来加了但没生效”。

常见修复方案：
- **拆分为两个 Bean（推荐）**：把事务方法放到独立 Service，通过外部注入调用，天然走代理链，语义最清晰、可维护性最好。
- **通过代理对象调用自身方法**：如开启 `exposeProxy` 后用 `AopContext.currentProxy()` 调用，或注入自身代理；改动小但耦合 Spring AOP 细节，可读性较差。
- **编程式事务兜底**：使用 `TransactionTemplate` 明确事务边界，适合复杂分段事务场景，但样板代码更多。

线上“部分提交/数据不一致”排查与止血思路：
- 先止血：下线问题入口或降级写操作，避免脏数据继续扩大。
- 查日志链路：确认是否出现“内调绕过代理”、异常被吞、`rollbackFor` 未覆盖受检异常等。
- 对账修复：基于业务唯一键与时间窗口做数据核对，补偿或回滚错误数据。
- 加防回归：补充事务集成测试（重点覆盖同类调用、异常路径、传播行为），并在代码评审中禁止 `this.txMethod()` 这类写法。
