# spring/springboot
## Spring 框架的核心是什么  
Spring 框架的核心主要是 IoC（控制反转）和 AOP（面向切面编程）。 其中 IoC 是最核心的思想，它通过 Spring 容器来管理对象的创建和对象之间的依赖关系，开发者不需要自己去 new 对象，而是通过依赖注入的方式让容器自动完成，从而降低系统的耦合度，提高代码的可维护性。而 AOP 则是用来处理一些与业务无关但又需要统一处理的功能，比如日志记录、事务管理和权限控制等，通过在不修改业务代码的情况下将这些功能统一织入到方法执行过程中，从而提高代码的复用性和系统的扩展性  

##  Spring 中依赖注入（DI）有哪几种方式？分别有什么区别？  
**Spring 中依赖注入主要有三种方式：构造器注入、Setter 注入和字段注入。** 构造器注入是通过类的构造方法将依赖对象传入，这种方式可以保证对象在创建时依赖已经完整，安全性较高，也更利于单元测试；Setter 注入是通过 setter 方法为对象设置依赖，灵活性较高，适合可选依赖的情况；字段注入通常是通过 `@Autowired` 直接作用在成员变量上，由 Spring 自动完成注入，写法最简洁，但不利于测试，也不方便体现依赖关系。在实际开发中，一般**推荐使用构造器注入**，因为它更加规范、依赖关系清晰，也更符合面向对象设计原则  



## 请你说一下 Spring Bean 的生命周期。  
Spring Bean 的生命周期主要包括 **实例化、依赖注入、初始化、使用、销毁五个阶段**。在初始化过程中 Spring 提供了多个扩展点，例如 `Aware 接口`、`BeanPostProcessor`、`@PostConstruct` 等，用于在 Bean 创建前后进行增强或定制逻辑。其中 `BeanPostProcessor` 是 Spring 最重要的扩展机制之一，AOP、事务管理等核心功能都依赖于这个阶段实现代理对象的创建。  

<!-- 这是一张图片，ocr 内容为： -->
![](https://cdn.nlark.com/yuque/0/2026/png/22451783/1773644729004-71ab62b5-055d-4df2-b44a-c40d2d9a6cb4.png)

首先 Spring 根据 BeanDefinition 创建 Bean 实例，然后进行属性填充完成依赖注入。

如果 Bean 实现了 Aware 接口，比如 BeanNameAware 或 ApplicationContextAware，Spring 会回调对应方法让 Bean 获取容器相关信息。

接着会执行 BeanPostProcessor 的 postProcessBeforeInitialization 方法。

然后进入初始化阶段，可能执行 @PostConstruct、InitializingBean 的 afterPropertiesSet 方法或者自定义 init-method。

初始化完成后会执行 BeanPostProcessor 的 postProcessAfterInitialization 方法，很多功能例如 Spring AOP、事务管理就是在这个阶段创建代理对象。

最后 Bean 就可以被容器正常使用。当 Spring 容器关闭时，会执行 @PreDestroy、DisposableBean 的 destroy 方法或者 destroy-method 完成 Bean 的销毁



## Spring 是如何解决循环依赖的？
Spring 通过三级缓存机制解决单例 Bean 的属性注入循环依赖问题。

当 Spring 创建 Bean 时，首先会进行实例化，但此时还没有完成属性注入和初始化。

在实例化完成后，Spring 会将该 Bean 对应的 ObjectFactory 提前放入三级缓存 singletonFactories 中，用于生成 Bean 的早期引用。

如果在属性注入阶段发生循环依赖，例如 A 依赖 B，B 又依赖 A，当 B 需要注入 A 时，Spring 就会从三级缓存中获取 A 的 ObjectFactory，并调用 getObject() 得到 A 的早期引用，从而打破循环依赖。

三级缓存的结构分别是：

+ 一级缓存 singletonObjects：存放完全初始化的 Bean
+ 二级缓存 earlySingletonObjects：存放早期 Bean 引用
+ 三级缓存 singletonFactories：存放 Bean 的 ObjectFactory

之所以设计三级缓存，是因为 Spring 需要在循环依赖发生时通过 ObjectFactory 调用 getEarlyBeanReference()，判断该 Bean 是否需要创建 AOP 代理，从而保证提前暴露的是代理对象而不是原始对象。

需要注意的是，Spring 只能解决 **单例 Bean 的 setter 或字段注入循环依赖**，对于 **构造器循环依赖** 和 **prototype Bean 循环依赖**是无法解决的。



##  Spring AOP 为什么默认使用 JDK 动态代理和 CGLIB？两者有什么区别？  
Spring AOP 通过动态代理实现方法增强，主要支持 JDK 动态代理和 CGLIB 动态代理两种方式。

JDK 动态代理是基于接口实现的，通过 `java.lang.reflect.Proxy` 在运行时生成代理对象，并通过 `InvocationHandler` 对方法调用进行拦截。

CGLIB 是基于继承实现的，它会在运行时生成目标类的子类，并通过方法拦截器对目标方法进行增强。

Spring 默认的选择策略是：如果目标类实现了接口，则优先使用 JDK 动态代理；如果没有实现接口，则使用 CGLIB。

两者的主要区别在于：

+ JDK 动态代理必须基于接口
+ CGLIB 基于继承，因此不能代理 final 类和 final 方法

在 Spring Boot 2.x 之后，默认配置 `spring.aop.proxy-target-class=true`，即默认使用 CGLIB 代理，这样可以统一代理行为，减少接口限制。



## Spring Boot 的自动配置（AutoConfiguration）是如何实现的？
 Spring Boot 自动配置的核心是通过 `@EnableAutoConfiguration` 启动自动配置机制，该注解通过 `AutoConfigurationImportSelector` 使用 `SpringFactoriesLoader` 读取 `META-INF/spring.factories`（Spring Boot 3 中为 `AutoConfiguration.imports`）文件中的所有自动配置类，然后根据 `@Conditional` 条件注解判断当前环境是否满足条件，满足条件才会将对应 Bean 注册到 Spring 容器中。通过这种 **SPI + 条件装配** 的机制，实现了 Spring Boot 引入依赖即可自动完成大量配置的能力。  



##  如果我想自己写一个 Spring Boot Starter，需要做哪些步骤？  

## MySQL InnoDB 事务提交时，redo log 和 binlog 是怎么配合保证主从一致与崩溃恢复的？为什么需要两阶段提交？
redo log 是 InnoDB 层的物理日志，核心作用是崩溃恢复（保证已提交事务在宕机后可重做）；binlog 是 Server 层的逻辑日志，核心作用是主从复制、归档与时间点恢复。

事务提交的关键流程是两阶段提交：先写 redo log prepare，再写并落盘 binlog，最后写 redo log commit。这个顺序的目标是让两类日志保持原子一致，避免只成功一半。

如果只有 redo log，没有 binlog，则主库可恢复但从库无法完整复制，容易主从不一致；如果只有 binlog，没有 redo log，则 InnoDB 崩溃后无法保证页级恢复一致性。

MySQL 重启恢复时，会检查处于 prepare 状态的事务：若在 binlog 中能找到完整对应事务，则提交；否则回滚。通过这个判断可保证崩溃恢复结果与 binlog 一致，从而保障主从一致性。
