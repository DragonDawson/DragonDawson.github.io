---
title: "Spring 事务管理实战指南：常见问题、最佳实践与案例分析"
date: 2026-06-25
lastmod: 2026-06-25
description: "Spring 事务管理的实战指南，深入探讨事务传播行为、隔离级别、常见坑点及解决方案。包含大量实战代码示例和真实项目问题分析。"
tags: ["Spring", "事务管理", "Java", "后端开发", "实战", "最佳实践"]
---

## 前言

在上一篇文章中，我们深入剖析了 Spring 事务的底层原理、代理机制和回滚机制。本篇将从**实战角度**出发，通过真实案例和代码示例，帮你解决日常开发中遇到的 Spring 事务问题。

**本文涵盖的核心问题**：

1. 事务传播行为怎么选？REQUIRED 和 REQUIRES_NEW 有什么区别？
2. 事务隔离级别如何设置？能解决脏读、不可重复读吗？
3. 为什么 @Transactional 放在 private 方法上不生效？
4. 大事务导致性能问题，如何拆分？
5. 读写分离场景下，事务该怎么处理？
6. 真实项目中的事务最佳实践

---

## 一、事务传播行为详解

事务传播行为（Propagation Behavior）定义了**当一个事务方法被另一个事务方法调用时，事务如何传播**。

### 7 种传播行为对比

| 传播行为 | 含义 | 适用场景 |
|---------|------|---------|
| **REQUIRED**（默认） | 有则加入，无则新建 | 大部分业务方法 |
| **REQUIRES_NEW** | 总是新建，挂起外部事务 | 日志记录、独立操作 |
| **SUPPORTS** | 有则加入，无则非事务执行 | 查询方法 |
| **NOT_SUPPORTED** | 非事务执行，挂起外部事务 | 不需要事务的操作 |
| **MANDATORY** | 必须存在事务，否则抛异常 | 强制在事务中执行 |
| **NEVER** | 必须非事务，否则抛异常 | 禁止在事务中执行 |
| **NESTED** | 嵌套事务，依赖 JDBC 3.0+ | 部分回滚场景 |

### 代码示例：REQUIRED vs REQUIRES_NEW

```java
@Service
public class OrderService {

    @Autowired
    private LogService logService;

    @Autowired
    private InventoryService inventoryService;

    /**
     * 场景1：REQUIRED（默认）
     * - 外部有事务：加入
     * - 外部无事务：新建
     */
    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
    public void createOrder(Order order) {
        // 保存订单
        orderDao.save(order);

        // 记录操作日志
        logService.log("创建订单: " + order.getId());

        // 扣减库存
        inventoryService.reduce(order.getProductId(), order.getQty());

        // 如果这里抛异常，logService 的日志也会回滚！
        // 因为它们在同一个事务中
        if (order.getQty() > 1000) {
            throw new RuntimeException("数量超限");
        }
    }

    /**
     * 场景2：REQUIRES_NEW
     * - 总是新建事务
     * - 外部事务被挂起
     * - 内部事务提交/回滚不影响外部事务
     */
    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
    public void createOrderWithIndependentLog(Order order) {
        // 保存订单
        orderDao.save(order);

        // 记录操作日志（独立事务）
        try {
            logService.logIndependent("创建订单: " + order.getId());
        } catch (Exception e) {
            // 日志失败不影响订单创建
            log.warn("日志记录失败", e);
        }

        // 扣减库存
        inventoryService.reduce(order.getProductId(), order.getQty());
    }
}

@Service
public class LogService {

    /**
     * 独立事务：无论外界是否回滚，日志都会保存
     */
    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void logIndependent(String message) {
        Log log = new Log();
        log.setMessage(message);
        log.setCreateTime(new Date());
        logDao.save(log);
    }

    /**
     * 加入当前事务：外界回滚，日志也会回滚
     */
    @Transactional(propagation = Propagation.REQUIRED, rollbackFor = Exception.class)
    public void log(String message) {
        Log log = new Log();
        log.setMessage(message);
        log.setCreateTime(new Date());
        logDao.save(log);
    }
}
```

### 传播行为选择建议

```
✓ 大部分业务方法 → REQUIRED
✓ 日志记录、审计、发送消息等辅助操作 → REQUIRES_NEW
✓ 查询方法 → SUPPORTS（只读优化）
✓ 必须保证在事务中执行 → MANDATORY
✗ 避免使用 NESTED（依赖数据库支持，移植性差）
```

---

## 二、事务隔离级别

### 并发事务的 5 类问题

| 问题 | 描述 | 后果 |
|------|------|------|
| **脏读** | 读到别的事务未提交的数据 | 数据可能回滚，读到无效数据 |
| **不可重复读** | 同一事务内，两次读到的数据不一致 | 被别的事务修改了 |
| **幻读** | 同一事务内，两次查询的记录数不一致 | 被别的事务插入/删除了 |
| **第一类丢失更新** | 一个事务回滚，覆盖了另一个事务的提交 | 数据丢失 |
| **第二类丢失更新** | 一个事务提交，覆盖了另一个事务的提交 | 数据覆盖 |

### 4 种标准隔离级别

| 隔离级别 | 脏读 | 不可重复读 | 幻读 | 性能 |
|---------|------|-----------|------|------|
| **READ_UNCOMMITTED** | ✗ | ✗ | ✗ | 最高 |
| **READ_COMMITTED** | ✓ | ✗ | ✗ | 高 |
| **REPEATABLE_READ** | ✓ | ✓ | ✗ | 中 |
| **SERIALIZABLE** | ✓ | ✓ | ✓ | 最低 |

### Spring 中设置隔离级别

```java
@Service
public class AccountService {

    /**
     * 设置隔离级别为 READ_COMMITTED
     * 防止脏读，但允许不可重复读
     */
    @Transactional(
        isolation = Isolation.READ_COMMITTED,
        rollbackFor = Exception.class
    )
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountDao.findById(fromId);
        Account to = accountDao.findById(toId);

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));

        accountDao.update(from);
        accountDao.update(to);
    }

    /**
     * 设置隔离级别为 SERIALIZABLE
     * 完全串行化，性能差，慎用！
     */
    @Transactional(
        isolation = Isolation.SERIALIZABLE,
        rollbackFor = Exception.class
    )
    public void lockAccount(Long accountId) {
        // 需要完全串行化的操作
        Account account = accountDao.findById(accountId);
        account.setLocked(true);
        accountDao.update(account);
    }
}
```

### 重要提醒

> **Spring 的隔离级别设置，最终生效的是底层数据库！**
>
> - MySQL：默认 REPEATABLE_READ
> - Oracle/PostgreSQL：默认 READ_COMMITTED
> - 如果数据库不支持某个隔离级别，会使用更高的级别

---

## 三、@Transactional 的常见坑

### 坑 1：注解放在 private 方法上不生效

```java
@Service
public class UserService {

    /**
     * ✗ 错误：private 方法上的 @Transactional 不生效
     * Spring 代理基于 AOP，无法拦截 private 方法
     */
    @Transactional(rollbackFor = Exception.class)
    private void saveUserPrivate(User user) {
        userDao.save(user);
        throw new RuntimeException("测试回滚");
        // 结果：数据不会回滚，因为事务没生效！
    }

    /**
     * ✓ 正确：public 方法上的 @Transactional 生效
     */
    @Transactional(rollbackFor = Exception.class)
    public void saveUser(User user) {
        userDao.save(user);
        throw new RuntimeException("测试回滚");
        // 结果：数据回滚 ✓
    }
}
```

### 坑 2：异常被 catch 吞掉，导致不回滚

```java
@Service
public class PaymentService {

    /**
     * ✗ 错误：异常被 catch 吞掉，事务不会回滚
     */
    @Transactional(rollbackFor = Exception.class)
    public void payWrong(Order order) {
        try {
            orderDao.updateStatus(order.getId(), "PAID");
            accountDao.debit(order.getUserId(), order.getAmount());
        } catch (Exception e) {
            log.error("支付失败", e);
            // ⚠️ 异常被吞掉了，事务拦截器看不到异常
            // 结果：数据不会回滚！
        }
    }

    /**
     * ✓ 正确做法 1：catch 后重新抛出
     */
    @Transactional(rollbackFor = Exception.class)
    public void payCorrect1(Order order) {
        try {
            orderDao.updateStatus(order.getId(), "PAID");
            accountDao.debit(order.getUserId(), order.getAmount());
        } catch (Exception e) {
            log.error("支付失败", e);
            throw e; // ✓ 重新抛出，事务会回滚
        }
    }

    /**
     * ✓ 正确做法 2：手动标记回滚
     */
    @Transactional(rollbackFor = Exception.class)
    public void payCorrect2(Order order) {
        try {
            orderDao.updateStatus(order.getId(), "PAID");
            accountDao.debit(order.getUserId(), order.getAmount());
        } catch (Exception e) {
            log.error("支付失败", e);
            // ✓ 手动标记回滚
            TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
        }
    }
}
```

### 坑 3：默认只回滚 RuntimeException

```java
@Service
public class ImportService {

    /**
     * ✗ 危险：受检异常默认不回滚
     */
    @Transactional  // 注意：没有配置 rollbackFor
    public void importData() throws IOException {
        dao.save(data1);
        dao.save(data2);

        // 抛出受检异常
        throw new IOException("文件格式错误");
        // 结果：事务不会回滚！数据已经保存了！
    }

    /**
     * ✓ 正确：明确指定回滚异常
     */
    @Transactional(rollbackFor = Exception.class)
    public void importDataCorrect() throws IOException {
        dao.save(data1);
        dao.save(data2);

        throw new IOException("文件格式错误");
        // 结果：事务回滚 ✓
    }
}
```

### 坑 4：同类方法互调，事务失效

这个问题在上一篇文章中详细讲解过，这里再强调一下解决方案：

```java
@Service
public class QuoteService {

    /**
     * 方案 1（推荐）：拆类
     * 把需要事务管理的方法拆到另一个 Service
     */
    @Autowired
    private QuoteTransactionService txService;

    @Transactional(rollbackFor = Exception.class)
    public void createQuote(Quote quote) {
        // 跨类调用，事务生效 ✓
        txService.saveQuote(quote);
    }
}

@Service
public class QuoteTransactionService {

    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void saveQuote(Quote quote) {
        dao.save(quote);
    }
}

/**
 * 方案 2：自注入（不推荐，但可用）
 */
@Service
public class QuoteService2 {

    @Autowired
    private QuoteService2 self;  // 注入自己（实际上是代理对象）

    @Transactional(rollbackFor = Exception.class)
    public void createQuote(Quote quote) {
        // 通过 self 调用，经过代理，事务生效 ✓
        self.saveQuote(quote);
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void saveQuote(Quote quote) {
        dao.save(quote);
    }
}
```

---

## 四、大事务问题及优化

### 什么是大事务？

```
特征：
- 事务内包含大量数据库操作（成百上千条 SQL）
- 事务内包含远程调用（HTTP、RPC）
- 事务内包含耗时操作（文件处理、循环计算）
- 事务执行时间超过几秒钟

危害：
- 数据库连接长时间占用
- 数据库锁持有时间过长
- 并发性能严重下降
- 容易导致死锁
```

### 大事务示例与优化

```java
@Service
public class BatchImportService {

    /**
     * ✗ 错误示例：大事务
     * - 导入 10000 条数据在一个事务内
     * - 包含远程调用
     * - 事务执行时间可能超过 1 分钟
     */
    @Transactional(rollbackFor = Exception.class)
    public void importBatchWrong(List<Data> dataList) {
        for (Data data : dataList) {
            // 1. 数据库查询
            Data existing = dao.findByUniqueKey(data.getUniqueKey());

            // 2. 远程调用（非常耗时！）
            RemoteResult remote = remoteService.validate(data);

            // 3. 数据库保存
            if (existing == null) {
                dao.insert(data);
            } else {
                dao.update(data);
            }

            // 4. 记录日志
            logService.log("导入数据: " + data.getId());
        }
        // 事务持续整个循环，锁持有时间极长！
    }

    /**
     * ✓ 正确做法 1：拆分成小事务
     */
    public void importBatchCorrect1(List<Data> dataList) {
        for (Data data : dataList) {
            // 每条数据独立事务
            importSingleData(data);
        }
    }

    @Transactional(rollbackFor = Exception.class, propagation = Propagation.REQUIRES_NEW)
    public void importSingleData(Data data) {
        Data existing = dao.findByUniqueKey(data.getUniqueKey());
        if (existing == null) {
            dao.insert(data);
        } else {
            dao.update(data);
        }
    }

    /**
     * ✓ 正确做法 2：批量处理 + 分批提交
     */
    public void importBatchCorrect2(List<Data> dataList) {
        int batchSize = 100;
        List<List<Data>> batches = partition(dataList, batchSize);

        for (List<Data> batch : batches) {
            importBatch(batch);
        }
    }

    @Transactional(rollbackFor = Exception.class)
    public void importBatch(List<Data> batch) {
        // 批量插入/更新
        dao.batchInsertOrUpdate(batch);
    }

    /**
     * ✓ 正确做法 3：移除事务外的耗时操作
     */
    public void importBatchCorrect3(List<Data> dataList) {
        // 1. 预先查询所有已存在的数据（一次查询）
        Set<String> existingKeys = dao.findAllUniqueKeys();

        // 2. 预先验证所有数据（事务外）
        List<Data> validData = preValidate(dataList);

        // 3. 批量保存（在事务内）
        for (Data data : validData) {
            if (existingKeys.contains(data.getUniqueKey())) {
                dao.update(data);
            } else {
                dao.insert(data);
            }
        }
    }
}
```

### 大事务优化建议

```
1. 事务内只做数据库操作，避免远程调用
2. 将大循环拆分成小事务
3. 使用批量操作减少 SQL 条数
4. 将查询操作移到事务外
5. 考虑使用编程式事务，精确控制事务边界
6. 对于超大批量数据，考虑使用存储过程或批处理框架
```

---

## 五、读写分离场景下的事务处理

### 问题：读写分离后，事务内的读操作怎么处理？

```java
@Service
public class OrderQueryService {

    @Autowired
    private OrderDao orderDao;  // 读库

    @Autowired
    private OrderWriteDao orderWriteDao;  // 写库

    /**
     * 场景：先写后读
     * 问题：写操作在写库，读操作在读库，可能读不到刚才写的数据
     */
    @Transactional(rollbackFor = Exception.class)
    public Order createAndQuery(Order order) {
        // 1. 写入写库
        orderWriteDao.save(order);

        // 2. 从读库查询
        // ⚠️ 问题：主从延迟，可能查不到！
        Order saved = orderDao.findById(order.getId());

        return saved;  // 可能返回 null
    }
}
```

### 解决方案

```java
@Service
public class OrderService {

    @Autowired
    private OrderDao orderDao;

    @Autowired
    private OrderWriteDao orderWriteDao;

    /**
     * 方案 1：事务内强制读主库
     * 使用 MyBatis 的 @Transactional + 强制路由
     */
    @Transactional(rollbackFor = Exception.class)
    public Order createAndQuery1(Order order) {
        // 写入主库
        orderWriteDao.save(order);

        // 强制从主库读取（具体实现依赖中间件）
        Order saved = orderWriteDao.findById(order.getId());

        return saved;
    }

    /**
     * 方案 2：拆分读写，避免事务内查询
     */
    public Order createAndQuery2(Order order) {
        // 1. 创建订单（在事务内）
        createOrder(order);

        // 2. 查询订单（事务外，允许短暂延迟）
        // 如果是用户立即查询，前端可以轮询或等待几秒
        Order saved = orderDao.findById(order.getId());

        return saved;
    }

    @Transactional(rollbackFor = Exception.class)
    public void createOrder(Order order) {
        orderWriteDao.save(order);
    }
}
```

---

## 六、真实项目最佳实践

### 最佳实践 1：统一事务配置

```xml
<!-- applicationContext.xml -->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <!-- 查询方法：支持事务，只读 -->
        <tx:method name="find*"     propagation="SUPPORTS" read-only="true"/>
        <tx:method name="get*"      propagation="SUPPORTS" read-only="true"/>
        <tx:method name="query*"    propagation="SUPPORTS" read-only="true"/>
        <tx:method name="list*"     propagation="SUPPORTS" read-only="true"/>
        <tx:method name="count*"    propagation="SUPPORTS" read-only="true"/>
        <tx:method name="exists*"   propagation="SUPPORTS" read-only="true"/>

        <!-- 写方法：必须有事务，所有异常都回滚 -->
        <tx:method name="save*"     propagation="REQUIRED" rollback-for="java.lang.Exception"/>
        <tx:method name="update*"   propagation="REQUIRED" rollback-for="java.lang.Exception"/>
        <tx:method name="delete*"   propagation="REQUIRED" rollback-for="java.lang.Exception"/>
        <tx:method name="create*"   propagation="REQUIRED" rollback-for="java.lang.Exception"/>
        <tx:method name="batch*"    propagation="REQUIRED" rollback-for="java.lang.Exception"/>

        <!-- 默认：必须有事务，所有异常都回滚 -->
        <tx:method name="*"         propagation="REQUIRED" rollback-for="java.lang.Exception"/>
    </tx:attributes>
</tx:advice>
```

### 最佳实践 2：Controller 层不写事务

```
✓ Service 层管理事务
✗ Controller 层不添加 @Transactional

原因：
1. 事务应该与业务逻辑一起定义在 Service 层
2. Controller 主要负责参数校验、请求转发
3. 在 Controller 开启事务会导致事务边界不清晰
```

### 最佳实践 3：事务方法要尽可能小

```java
@Service
public class UserService {

    /**
     * ✓ 推荐：事务方法只做数据库操作
     */
    @Transactional(rollbackFor = Exception.class)
    public void saveUser(User user) {
        userDao.save(user);
        roleDao.assignRoles(user.getId(), user.getRoleIds());
    }

    /**
     * ✗ 不推荐：事务方法包含非数据库操作
     */
    @Transactional(rollbackFor = Exception.class)
    public void saveUserWrong(User user) {
        userDao.save(user);

        // ⚠️ 这些操作不应该在事务内
        sendEmail(user.getEmail());
        pushToMQ(user);
        writeToFile(user);
    }
}
```

### 最佳实践 4：使用编程式事务精确控制

```java
@Service
public class ComplexService {

    @Autowired
    private TransactionTemplate transactionTemplate;

    /**
     * 使用编程式事务处理复杂场景
     */
    public void complexOperation(Data data) {
        // 1. 非事务操作
        preProcess(data);

        // 2. 事务操作 1
        boolean success1 = transactionTemplate.execute(status -> {
            try {
                dao.save(data);
                return true;
            } catch (Exception e) {
                status.setRollbackOnly();
                return false;
            }
        });

        if (!success1) {
            return;
        }

        // 3. 非事务操作
        postProcess(data);

        // 4. 事务操作 2（独立事务）
        transactionTemplate.setPropagationBehavior(Propagation.REQUIRES_NEW.value());
        boolean success2 = transactionTemplate.execute(status -> {
            try {
                logService.log("操作完成");
                return true;
            } catch (Exception e) {
                status.setRollbackOnly();
                return false;
            }
        });
    }
}
```

---

## 七、事务问题排查清单

当你遇到事务不回滚或事务失效的问题时，按这个清单排查：

```
排查清单：

□ 1. 是否使用了 @Transactional 注解？
  → 检查注解是否存在

□ 2. 注解是否放在 public 方法上？
  → private/protected 方法不生效

□ 3. 是否通过代理对象调用？
  → 检查是否是同类方法互调（this.xxx()）

□ 4. 异常是否被 catch 吞掉了？
  → 异常必须抛到代理层才能触发回滚

□ 5. rollbackFor 是否配置正确？
  → 默认只回滚 RuntimeException，建议配置 rollbackFor = Exception.class

□ 6. 数据库引擎是否支持事务？
  → MySQL MyISAM 不支持事务，必须使用 InnoDB

□ 7. 是否配置了事务管理器？
  → 检查 transactionManager 配置

□ 8. 是否启用了注解驱动？
  → 检查 <tx:annotation-driven/> 或 @EnableTransactionManagement

□ 9. 是否是自调用问题？
  → 参考坑 4 的解决方案

□ 10. 事务传播行为是否正确？
  → 检查 propagation 属性
```

---

## 总结

```
Spring 事务管理实战要点
│
├─ 传播行为选择
│   ├─ 大部分业务方法 → REQUIRED
│   ├─ 日志记录等辅助操作 → REQUIRES_NEW
│   └─ 查询方法 → SUPPORTS
│
├─ 隔离级别
│   ├─ 根据业务需求选择
│   └─ 注意：最终生效的是数据库
│
├─ 常见坑点
│   ├─ private 方法上注解不生效
│   ├─ 异常被 catch 吞掉
│   ├─ 默认只回滚 RuntimeException
│   └─ 同类方法互调失效
│
├─ 大事务优化
│   ├─ 拆分成小事务
│   ├─ 批量处理
│   └─ 移除事务内的耗时操作
│
└─ 最佳实践
    ├─ Service 层管理事务
    ├─ 事务方法尽可能小
    ├─ 统一事务配置
    └─ 使用编程式事务精确控制
```

---

## 参考资源

- [Spring 官方文档 - 事务管理](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#transaction)
- [上一篇文章：Spring 事务全面解析]({{< ref "spring-transaction-deep-dive.md" >}})

---

*如果你在项目中遇到其他事务问题，欢迎留言讨论。*
