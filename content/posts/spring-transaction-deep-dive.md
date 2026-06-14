---
title: "Spring 事务全面解析：配置方式、代理机制与回滚原理"
date: 2026-06-15
lastmod: 2026-06-15
description: "深入浅出讲解 Spring 事务的三种配置方式、代理机制原理、同类调用事务失效原因、XML 切点表达式含义，以及异常触发回滚的完整链路。附真实项目代码分析。"
tags: ["Spring", "事务", "Java", "AOP", "代理模式", "后端开发"]
---

## 前言

Spring 事务是 Java 后端开发的核心知识点，但很多人对它的理解停留在"加个 `@Transactional` 就行"，遇到事务不回滚的问题就一头雾水。

本文用**生活化比喻 + 代码 + 图解**，把下面几个问题讲透：

1. Spring 事务有几种配置方式，分别怎么用？
2. 什么是代理？为什么它管得了事务？
3. 为什么 A → B → C 抛异常会回滚，但 C 内部方法调方法就不回滚？
4. `execution(...)` 切点表达式到底是什么意思？
5. 数据库保存失败，为什么事务会回滚？

---

## 一、Spring 事务的三种配置方式

Spring 事务管理本质上是对数据库事务的封装，有三种配置方式，按使用频率排序：

### 方式一：注解声明式事务（推荐 ★）

**适用版本**：Spring 2.5+  
**核心注解**：`@Transactional`

**第一步：XML 开启注解驱动（只需一行）**

```xml
<!-- applicationContext.xml -->
<tx:annotation-driven transaction-manager="transactionManager"/>
```

**第二步：在 Service 类或方法上加注解**

```java
@Service
public class OrderService {

    // 加在类上 = 所有方法都有事务
    @Transactional
    public void createOrder(Order order) {
        orderDao.save(order);
        inventoryDao.reduce(order.getProductId(), order.getQty());
    }

    // 单独方法可覆盖类级别配置
    @Transactional(propagation = Propagation.SUPPORTS, readOnly = true)
    public Order findById(Long id) {
        return orderDao.findById(id);
    }

    // 指定回滚异常类型（重要！）
    @Transactional(rollbackFor = Exception.class)
    public void transfer(Long from, Long to, BigDecimal amount) {
        accountDao.debit(from, amount);
        accountDao.credit(to, amount);
    }
}
```

**优点**：简洁清晰，现代项目首选  
**缺点**：需注意同类内部调用导致代理失效的问题（后面详解）

---

### 方式二：XML 声明式事务（老项目常见）

**适用版本**：Spring 2.0+  
**核心**：`<tx:advice>` + `<aop:config>`

```xml
<!-- 1. 配置事务管理器 -->
<bean id="transactionManager"
    class="org.springframework.orm.hibernate3.HibernateTransactionManager">
    <property name="sessionFactory" ref="sessionFactory"/>
</bean>

<!-- 2. 定义事务通知（哪些方法用什么事务属性） -->
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <!-- 查询方法：支持事务但不强制，只读优化 -->
        <tx:method name="find*"   propagation="SUPPORTS" read-only="true"/>
        <tx:method name="get*"    propagation="SUPPORTS" read-only="true"/>
        <tx:method name="query*"  propagation="SUPPORTS" read-only="true"/>
        <!-- 写方法：必须有事务，所有 Exception 都回滚 -->
        <tx:method name="*" propagation="REQUIRED"
                   rollback-for="java.lang.Exception"/>
    </tx:attributes>
</tx:advice>

<!-- 3. AOP 切点绑定：把事务通知织入到哪些方法 -->
<aop:config>
    <aop:pointcut id="serviceMethods"
        expression="execution(* com.example.service..*ServiceImpl.*(..))"/>
    <aop:advisor advice-ref="txAdvice" pointcut-ref="serviceMethods"/>
</aop:config>
```

**优点**：零代码侵入，适合不允许用注解的老项目  
**缺点**：XML 配置繁琐，可读性差

---

### 方式三：编程式事务（不推荐生产使用）

手动在代码中控制事务的开启、提交、回滚。

```java
@Autowired
private TransactionTemplate transactionTemplate;

public void doSomething() {
    transactionTemplate.execute(status -> {
        try {
            dao.insert(...);
            dao.update(...);
        } catch (Exception e) {
            status.setRollbackOnly(); // 标记回滚
            throw e;
        }
        return null;
    });
}
```

**优点**：控制粒度最细，适合极特殊的业务场景  
**缺点**：代码侵入性强，事务逻辑和业务逻辑耦合

---

### 三种方式对比

| 方式 | 配置位置 | 代码侵入 | 推荐度 |
|------|---------|---------|--------|
| 注解声明式 | 方法/类上的 `@Transactional` | 低 | ★★★ |
| XML 声明式 | `applicationContext.xml` | 无 | ★★ |
| 编程式 | Java 代码内部 | 高 | ★ |

---

## 二、代理是什么？为什么它能管事务？

要理解 Spring 事务，必须先理解**代理**。

### 生活化比喻：公司报销审批

假设你们公司有个**财务审批代理公司**（= Spring 代理），专门帮各部门处理报销流程：

```
正常流程：
小帅 → [财务代理] → 审核 → 打款 → 出问题就撤回 ✓

坑：
老张签完字，又自己随手签了个补充说明（没经过代理）
→ 财务代理根本不知道这件事
→ 补充内容出错了，撤不回来 ✗
```

**对应到代码**：

Spring 注入给你的 `Service` 对象，不是你自己写的那个类，而是 Spring **偷偷生成的一个"代理对象"**。

```
你以为：   Controller → QuotePubLiabilityServiceImpl（真实对象）

实际上： Controller → QuotePubLiabilityServiceImpl$$Proxy（代理对象）
                            ↓ 代理在调真实方法前后偷偷加事务逻辑
                          → QuotePubLiabilityServiceImpl（真实对象）
```

### 代理对象长什么样（伪代码）

```java
// Spring 实际生成并注入的代理对象（伪代码，帮助理解）
public class QuotePubLiabilityServiceImpl$$Proxy
        extends QuotePubLiabilityServiceImpl {

    @Override
    public QuotePubLiabilityMain buildQuoteFromProposal(PrpCmain prpCmain) {
        // 代理偷偷加的：
        TransactionStatus status = txManager.getTransaction(...);  // 开事务
        try {
            // 调真实对象的方法
            QuotePubLiabilityMain result = super.buildQuoteFromProposal(prpCmain);
            txManager.commit(status);  // 提交
            return result;
        } catch (Exception e) {
            txManager.rollback(status);  // 回滚
            throw e;
        }
    }
}
```

**所以：所有外部对 Service 方法的调用，都先经过代理对象，事务拦截器才有机会执行。**

---

## 三、同类调用事务失效的原理

这是 Spring 事务最经典的坑。

### 问题现象

```java
@Service
public class CServiceImpl implements CService {

    @Transactional(rollbackFor = Exception.class)
    public void methodX() {
        // 业务逻辑...
        methodY();  // ← 等价于 this.methodY()
    }

    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void methodY() {
        dao.insert(...);
        throw new RuntimeException("出错了");
    }
}
```

**期望**：`methodY` 用独立事务，失败了只回滚 `methodY` 的操作  
**实际**：`methodY` 上的 `REQUIRES_NEW` 不生效，它和 `methodX` 共用同一个事务

### 根本原因：this 指向真实对象，不是代理

```
外部调用
    ↓
[代理对象]  ← 这里拦截了，开了事务 ✓
    ↓ 调 super
[真实对象.methodX()]
    ↓
    this.methodY()   ← ⚠️ 直接调真实对象，代理管不到！
    ↓
    抛异常
    ↓
    异常回到代理 → 代理回滚（但 methodY 想独立事务的期望没实现）
```

**一句话总结**：  
> 找别人办事 = 走代理 = 事务生效  
> 自己叫自己 = 绕过代理 = 事务注解失效

---

### 调用链对比图

<div style="display:grid;grid-template-columns:1fr 1fr;gap:16px;margin:20px 0;">

<div style="border:1px solid #3B6D11;border-radius:8px;padding:12px;background:#EAF3DE;">
<h4 style="margin:0 0 8px;color:#3B6D11;font-size:14px;">✓ 跨类调用 — 事务生效</h4>
<pre style="font-size:12px;line-height:1.8;margin:0;color:#27500A;">
Controller
  → [代理B] 开事务
    → B.method1()
      → [代理C] 开事务
        → C.methodX() 抛异常
          → 异常上抛到代理C
            → 代理C 回滚 ✓
</pre>
</div>

<div style="border:1px solid #A32D2D;border-radius:8px;padding:12px;background:#FCEBEB;">
<h4 style="margin:0 0 8px;color:#A32D2D;font-size:14px;">✗ 同类调用 — 事务失效</h4>
<pre style="font-size:12px;line-height:1.8;margin:0;color:#791F1F;">
Controller
  → [代理C] 开事务
    → C.methodX()  (真实对象)
      → this.methodY()  ← 绕过代理！
        → methodY 上的 @Transactional 不生效 ✗
          → 抛异常
            → 回到代理C → 整个事务回滚（但 methodY 想独立事务没实现）
</pre>
</div>

</div>

---

### 四种解决方案

| 方案 | 做法 | 推荐度 |
|------|------|--------|
| **拆类** | 把 `methodY` 移到新的 `DService`，C 注入 D | ★★★ |
| **自注入** | C 里 `@Autowired C self;` 用 `self.methodY()` | ★★ |
| **AopContext** | `((C) AopContext.currentProxy()).methodY()` | ★★ |
| **编程式事务** | 用 `TransactionTemplate` 手动管理 | ★ |

**方案 1（拆类）示例**：

```java
// 把需要独立事务的方法拆到新类
@Service
public class CTransactionService {

    @Transactional(propagation = Propagation.REQUIRES_NEW, rollbackFor = Exception.class)
    public void methodY() {
        dao.insert(...);
        throw new RuntimeException("出错了");
    }
}

@Service
public class CServiceImpl implements CService {

    @Autowired
    private CTransactionService cTransactionService;  // 注入的是代理 ✓

    @Transactional(rollbackFor = Exception.class)
    public void methodX() {
        // 跨类调用，经过代理，REQUIRES_NEW 生效！
        cTransactionService.methodY();
    }
}
```

---

## 四、XML 切点表达式详解

你们项目 `applicationContext.xml` 里有这样一行：

```xml
<aop:advisor
    pointcut="execution(public * com.sinosoft.intf..*service..*Service*Impl.*(..))"
    advice-ref="txAdvice"/>
```

### 逐段拆解

```
execution(public * com.sinosoft.intf..*service..*Service*Impl.*(..))
          │       │  │                            │  │                  │  │
          │       │  │                            │  │                  │  └─ (..)      参数任意
          │       │  │                            │  │                  └─ *(..)          任意方法名
          │       │  │                            │  └─ *Service*Impl        类名含 Service 且以 Impl 结尾
          │       │  │                            └─ ..                      任意子包
          │       │  └─ *service                包名含 service
          │       └─ com.sinosoft.intf..         com.sinosoft.intf 及其所有子包
          └─ public                          只匹配 public 方法
```

### 能匹配到的例子

```
✓ com.sinosoft.intf.quotelog.service.spring.QuotePubLiabilityServiceImpl.buildQuoteFromProposal(...)
✓ com.sinosoft.intf.service.OrderServiceImpl.save(...)
✓ com.sinosoft.intf.xxx.yyy.service.zzz.SomeServiceImpl.anyMethod(String, int)
```

### 匹配不了的 example

```
✗ com.sinosoft.other.service.OrderServiceImpl.save(...)  ← 包不是 com.sinosoft.intf 开头
✗ com.sinosoft.intf.dao.QuoteDaoImpl.find(...)          ← 类名不含 Service
✗ com.sinosoft.intf.service.QuoteService.query(...)     ← 类名没有 Impl 后缀
✗ private method                                               ← 非 public 方法
```

### 翻译成人话

> 对所有 `com.sinosoft.intf` 包及其子包下、包名含 `service`、类名含 `Service` 且以 `Impl` 结尾的 **public 方法**，施加 `txAdvice`（事务通知）。

---

## 五、异常与回滚的完整触发链路

### 完整链条

```
步骤1：业务代码执行
    dao.save(entity)
        ↓
步骤2：Hibernate/MyBatis 把 SQL 发给数据库
        ↓
步骤3：数据库执行失败（主键冲突、字段超长、连接断了...）
        ↓
步骤4：数据库返回错误 → Hibernate 抛出 DataAccessException（运行时异常）
        ↓
步骤5：异常没有在方法内被 catch → 继续往外抛
        ↓
步骤6：异常走到代理对象的事务拦截器
        ↓
步骤7：拦截器看到异常 → 调用 txManager.rollback() → 回滚 ✓
```

**关键点**：回滚不是数据库自动做的，是 **Spring 看到异常后主动调用的**。

---

### 为什么有时存不进去却不回滚？

```java
@Transactional(rollbackFor = Exception.class)
public void saveData() {
    try {
        dao.save(entity1);  // 存失败了
        dao.save(entity2);
    } catch (Exception e) {
        log.error("出错了", e);
        // ⚠️ 异常被吞掉了，没有继续往外抛！
        // Spring 拦截器根本不知道出错了
    }
}
// 方法正常结束 → Spring 认为没异常 → 提交事务 → 数据不一致 ✗
```

**异常被 try-catch 吞掉 = Spring 不知道出错了 = 不会回滚。**

---

### 你们项目的配置特别好的一点

```xml
<tx:method name="*" propagation="REQUIRED"
    rollback-for="java.lang.Exception"/>
```

`rollback-for="java.lang.Exception"` 意思是：**所有 Exception（包括受检异常）都触发回滚**。

> 默认 Spring 只回滚 `RuntimeException` 和 `Error`，受检异常（如 `SQLException`、`IOException`）默认不回滚。你们这个配置把这个坑填了。

---

## 六、结合真实项目代码分析

以你们项目的 `QuotePubLiabilityServiceImpl` 为例。

### 类路径

`com.sinosoft.intf.quotelog.service.spring.QuotePubLiabilityServiceImpl`

### 切点是否匹配？

```
execution(public * com.sinosoft.intf..*service..*Service*Impl.*(..))
              │           │  │                    │  │                │
     com.sinosoft.intf  ✓  （包路径匹配）
     ..                  ✓  （quotelog.service.spring 是子包）
     *service            ✓  （service 是包名的一部分）
     ..                  ✓
     *Service*Impl       ✓  （类名含 Service 且以 Impl 结尾）
     .*(..)             ✓  （所有 public 方法）
```

**结论：切点命中，事务拦截器会生效。✓**

---

### `saveOrUpdate` 方法的事务行为

```java
// QuotePubLiabilityServiceImpl.java 第 253 行
@Override
public QuotePubLiabilityMain saveOrUpdate(QuotePubLiabilityMain entity) {
    super.getHibernateTemplate().merge(entity);
    return entity;
}
```

- 这是 **public 方法**，切点匹配 ✓
- 如果外部直接调用 `saveOrUpdate()`，事务生效 ✓
- 如果是**同类内部** `this.saveOrUpdate()` 调用，绕过代理，但注意：
  - 如果调用方（`buildQuoteFromProposal` 等）已经在事务中，`saveOrUpdate` 会**加入当前事务**（REQUIRED 传播行为），整个调用链一起回滚 ✓
  - 如果想让 `saveOrUpdate` 用**独立事务**，则必须避免 `this` 调用（用拆类或代理自调方案）

---

### 实际建议

对于你们项目，最实用的两种改法：

**改法 1：拆类（推荐）**

把数据库写操作拆到独立 Service：

```java
// 新类：专门负责数据库持久化
public class QuotePubLiabilityDBServiceImpl
        extends GenericDaoHibernate<Serializable, Serializable>
        implements QuotePubLiabilityService {

    @Override
    @Transactional(rollbackFor = Exception.class)
    public QuotePubLiabilityMain saveOrUpdate(QuotePubLiabilityMain entity) {
        super.getHibernateTemplate().merge(entity);
        return entity;
    }
}
```

**改法 2：开启 expose-proxy（快速修复）**

```xml
<!-- applicationContext.xml 第 20 行，加上 expose-proxy="true" -->
<aop:config proxy-target-class="true" expose-proxy="true">
```

代码中用代理自调：

```java
// 不用 this，用当前代理对象
((QuotePubLiabilityService) AopContext.currentProxy()).saveOrUpdate(quoteInfo);
```

---

## 总结

```
Spring 事务核心知识图谱
│
├─ 三种配置方式
│   ├─ 注解式（推荐）   @Transactional
│   ├─ XML 声明式        tx:advice + aop:config
│   └─ 编程式            TransactionTemplate
│
├─ 代理机制（核心原理）
│   ├─ Spring 注入的是代理对象，不是真实对象
│   ├─ 代理在调真实方法前后偷偷加事务逻辑
│   └─ this 调用 = 绕过代理 = 事务失效
│
├─ 回滚触发条件
│   ├─ 异常必须抛到代理层（不能被 catch 吞掉）
│   ├─ rollback-for 配置决定哪些异常触发回滚
│   └─ 默认只回滚 RuntimeException，你们配了 Exception（✓）
│
└─ 同类调用失效的解法
    ├─ 拆类（最推荐）
    ├─ expose-proxy + AopContext（快速）
    └─ 自注入（备选）
```

---

*如有问题或不同见解，欢迎讨论。*
