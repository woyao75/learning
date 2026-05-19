## 🤖 Assistant

```markdown
```

时回滚。

默认遇到 checked exception 不回滚。

如果希望 checked exception 也回滚，需要配置：

```java
@Transactional(rollbackFor = Exception.class)
```

---

#### 问题 3：Spring 事务传播行为是什么？

答：

事务传播行为决定一个事务方法调用另一个事务方法时，事务如何处理。

常见传播行为：

| 传播行为 | 说明 |
|---|---|
| `REQUIRED` | 默认，有事务就加入，没有就新建 |
| `REQUIRES_NEW` | 总是新建事务，挂起当前事务 |
| `SUPPORTS` | 有事务就加入，没有事务就非事务执行 |
| `MANDATORY` | 必须有事务，否则报错 |
| `NOT_SUPPORTED` | 总是非事务执行，有事务则挂起 |
| `NEVER` | 必须无事务，否则报错 |
| `NESTED` | 嵌套事务，依赖保存点 |

最常用的是 `REQUIRED` 和 `REQUIRES_NEW`。

---

#### 问题 4：`REQUIRED` 和 `REQUIRES_NEW` 区别？

答：

`REQUIRED` 是默认传播行为。

如果当前有事务，就加入当前事务；如果没有事务，就新建事务。

`REQUIRES_NEW` 总是新建一个事务。如果当前已有事务，会先挂起外层事务，内层事务独立提交或回滚。

示例：

```java
@Transactional
public void createOrder() {
    orderMapper.insert(order);
    auditLogService.saveLog("创建订单");
    throw new RuntimeException();
}
```

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void saveLog(String content) {
    auditLogMapper.insert(content);
}
```

如果 `saveLog` 是通过另一个 Spring Bean 调用的，那么即使 `createOrder` 回滚，`saveLog` 也可以独立提交。

---

#### 问题 5：`@Transactional` 常见失效原因？

答：

常见原因：

- 同类内部方法调用
- 方法不是 `public`
- 异常被捕获后没有继续抛出
- checked exception 没有配置 `rollbackFor`
- 类没有被 Spring 管理
- 数据库表不支持事务
- 使用了错误的事务管理器
- 多线程中事务上下文不传播
- `final` 类或 `final` 方法导致代理无法增强
- 在构造方法或 `@PostConstruct` 中调用事务方法

---

#### 问题 6：为什么事务中不建议调用远程接口？

答：

因为远程接口耗时不可控。

事务中调用远程接口会导致：

- 数据库连接被长时间占用
- 数据库锁持有时间变长
- 系统并发能力下降
- 死锁概率上升
- 远程调用失败时事务处理复杂

更合理的方式通常是缩短本地事务，使用状态机、消息队列、补偿任务或最终一致性方案处理外部系统交互。

---

#### 问题 7：Spring 事务和数据库事务是什么关系？

答：

Spring 事务是对数据库事务的统一抽象和管理。

真正提交、回滚事务的是数据库连接。

以 JDBC 为例，底层仍然是：

```java
connection.setAutoCommit(false);
connection.commit();
connection.rollback();
```

Spring 负责在合适的时机获取连接、开启事务、绑定线程、提交或回滚，并和 MyBatis、JPA 等框架集成。

---

#### 问题 8：为什么多线程中事务不传播？

答：

Spring 事务上下文通常保存在 `ThreadLocal` 中。

一个线程开启的事务资源只绑定在当前线程。

如果在事务方法中启动新线程，新线程没有父线程的事务上下文，因此不会自动加入原事务。

示例：

```java
@Transactional
public void createOrder() {
    orderMapper.insert(order);

    new Thread(() -> {
        logMapper.insert(log);
    }).start();
}
```

`logMapper.insert(log)` 不在 `createOrder` 的事务中。

---

## 43. 本模块练习

### 练习 1：构造方法注入

要求：

创建 `UserService`，通过构造方法注入 `UserMapper`。

参考代码：

```java
@Service
public class UserService {

    private final UserMapper userMapper;

    public UserService(UserMapper userMapper) {
        this.userMapper = userMapper;
    }

    public User getById(Long id) {
        return userMapper.selectById(id);
    }
}
```

重点：

- 不使用字段注入
- 依赖使用 `final`
- 构造方法表达必需依赖

---

### 练习 2：多个实现类注入

要求：

定义 `PayService` 接口，并创建支付宝和微信两个实现。

```java
public interface PayService {

    void pay(BigDecimal amount);
}
```

```java
@Service
public class AliPayService implements PayService {

    @Override
    public void pay(BigDecimal amount) {
        System.out.println("支付宝支付：" + amount);
    }
}
```

```java
@Service
public class WeChatPayService implements PayService {

    @Override
    public void pay(BigDecimal amount) {
        System.out.println("微信支付：" + amount);
    }
}
```

在 `OrderService` 中使用 `@Qualifier` 指定微信支付：

```java
@Service
public class OrderService {

    private final PayService payService;

    public OrderService(@Qualifier("weChatPayService") PayService payService) {
        this.payService = payService;
    }

    public void createOrder(BigDecimal amount) {
        payService.pay(amount);
    }
}
```

---

### 练习 3：自定义 Bean

要求：

使用 `@Configuration` 和 `@Bean` 注册一个 `ObjectMapper`。

```java
@Configuration
public class JacksonConfig {

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper();
    }
}
```

然后在 Service 中注入：

```java
@Service
public class JsonService {

    private final ObjectMapper objectMapper;

    public JsonService(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }
}
```

---

### 练习 4：Bean 生命周期

要求：

创建一个 Bean，在初始化和销毁时打印日志。

```java
@Component
public class DemoLifecycleBean {

    @PostConstruct
    public void init() {
        System.out.println("DemoLifecycleBean 初始化");
    }

    @PreDestroy
    public void destroy() {
        System.out.println("DemoLifecycleBean 销毁");
    }
}
```

Spring Boot 3 使用：

```java
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
```

---

### 练习 5：自定义 AOP 注解

要求：

实现一个 `@LogCost` 注解，统计方法耗时。

注解：

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogCost {
}
```

切面：

```java
@Aspect
@Component
public class LogCostAspect {

    @Around("@annotation(logCost)")
    public Object around(ProceedingJoinPoint joinPoint, LogCost logCost) throws Throwable {
        long start = System.currentTimeMillis();

        try {
            return joinPoint.proceed();
        } finally {
            long cost = System.currentTimeMillis() - start;
            System.out.println(joinPoint.getSignature().toShortString() + " cost=" + cost + "ms");
        }
    }
}
```

使用：

```java
@Service
public class OrderService {

    @LogCost
    public void createOrder() {
        System.out.println("创建订单");
    }
}
```

---

### 练习 6：事务回滚

要求：

实现账户转账方法，两个数据库操作必须在同一个事务中。

```java
@Service
public class AccountService {

    private final AccountMapper accountMapper;

    public AccountService(AccountMapper accountMapper) {
        this.accountMapper = accountMapper;
    }

    @Transactional
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        accountMapper.decreaseBalance(fromId, amount);

        if (amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("转账金额必须大于 0");
        }

        accountMapper.increaseBalance(toId, amount);
    }
}
```

验证：

- 正常转账，两条 SQL 都提交
- 中途抛出运行时异常，两条 SQL 都回滚

---

### 练习 7：checked exception 回滚

要求：

让受检异常也触发事务回滚。

```java
@Transactional(rollbackFor = Exception.class)
public void importUsers() throws IOException {
    userMapper.insert(user);
    throw new IOException("导入失败");
}
```

如果没有 `rollbackFor = Exception.class`，默认不会因为 `IOException` 回滚。

---

### 练习 8：事务失效验证

要求：

验证同类内部调用导致事务失效。

错误示例：

```java
@Service
public class DemoService {

    public void outer() {
        inner();
    }

    @Transactional
    public void inner() {
        userMapper.insert(user);
        throw new RuntimeException();
    }
}
```

调用 `outer()` 时，`inner()` 的事务可能不会生效。

改造方式：

```java
@Service
public class DemoService {

    private final DemoTransactionService demoTransactionService;

    public DemoService(DemoTransactionService demoTransactionService) {
        this.demoTransactionService = demoTransactionService;
    }

    public void outer() {
        demoTransactionService.inner();
    }
}
```

```java
@Service
public class DemoTransactionService {

    @Transactional
    public void inner() {
        userMapper.insert(user);
        throw new RuntimeException();
    }
}
```

---

## 44. 本模块总结

本模块覆盖了 Spring Framework 的核心机制：

- IoC 控制反转
- DI 依赖注入
- BeanFactory 与 ApplicationContext
- Bean 注册方式
- 组件扫描
- 多 Bean 注入
- Bean 作用域
- Bean 生命周期
- BeanPostProcessor
- 循环依赖
- AOP 核心概念
- Spring AOP 代理机制
- Pointcut 表达式
- AOP 失效原因
- 声明式事务
- 事务传播行为
- 事务隔离级别
- 回滚规则
- 事务失效原因
- Spring 常用扩展点
- Profile、配置绑定、条件装配

重点掌握：

- IoC 是思想，DI 是实现方式
- Bean 是由 Spring 容器管理的对象
- 构造方法注入优先于字段注入
- Spring AOP 基于代理实现
- 同类内部调用不会经过代理
- `@Transactional` 本质上依赖 AOP
- 默认只对 `RuntimeException` 和 `Error` 回滚
- 事务边界通常放在 Service 层
- 长事务和事务中远程调用都应谨慎
- 循环依赖优先重构，不要依赖框架兜底

下一模块建议进入：Spring MVC、REST API、参数绑定、校验、统一响应与异常处理。
```

