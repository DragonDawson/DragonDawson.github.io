---
title: "策略模式：告别一堆 if-else，让代码优雅切换行为"
date: 2026-06-15
lastmod: 2026-06-15
description: "用真实业务场景讲透策略模式，Java 8 Lambda 写法全程对比，配图解析。保险报价、促销折扣、支付方式……这些场景你一定遇到过。"
tags: ["Java", "设计模式", "策略模式", "Java8", "Lambda", "后端开发"]
---

## 前言

你有没有写过这种代码？

```java
if ("VIP".equals(userType)) {
    price = price * 0.8;
} else if ("SVIP".equals(userType)) {
    price = price * 0.6;
} else if ("NEW_USER".equals(userType)) {
    price = price - 10;
} else {
    // 普通用户，原价
}
```

刚写完觉得还行。三个月后来了个需求：再加一种"企业用户"折扣。  
你打开文件，翻到这一坨 `if-else`，内心：😰

这就是策略模式要解决的问题。

---

## 一、策略模式是什么？

> **策略模式（Strategy Pattern）**：定义一组算法（策略），把它们封装起来，让它们可以互换，而不影响使用它们的客户端代码。

用大白话说：**把"怎么做"从"做什么"里抽出来，单独管理。**

生活化比喻：你去北京，可以坐高铁、开车、坐飞机——**目的地一样，路线不同，随时切换**。策略模式就是把这些"路线"封装成独立的类，让你随时替换，不影响"去北京"这件事本身。

---

## 二、场景一：促销折扣计算

### 问题：if-else 地狱

```java
// 传统写法：每次加新折扣类型，都要改这个方法
public double calcPrice(String discountType, double price) {
    if ("VIP".equals(discountType)) {
        return price * 0.8;
    } else if ("SVIP".equals(discountType)) {
        return price * 0.6;
    } else if ("NEW_USER".equals(discountType)) {
        return Math.max(0, price - 10);
    } else {
        return price;
    }
}
```

**问题在哪？**
- 每次新增折扣类型，都要改这个方法 → 违反**开闭原则**（对扩展开放，对修改关闭）
- 逻辑越堆越多，难以测试，难以复用

---

### 策略模式重构（Java 8 版）

**第一步：定义策略接口**

```java
@FunctionalInterface
public interface DiscountStrategy {
    double apply(double originalPrice);
}
```

用 `@FunctionalInterface` 注解，表示这是一个函数式接口，可以直接用 Lambda 表达式实现。

**第二步：用 Lambda 定义各种策略**

```java
public class DiscountStrategies {

    // VIP 打八折
    public static final DiscountStrategy VIP =
            price -> price * 0.8;

    // SVIP 打六折
    public static final DiscountStrategy SVIP =
            price -> price * 0.6;

    // 新用户减 10 元
    public static final DiscountStrategy NEW_USER =
            price -> Math.max(0, price - 10);

    // 无折扣
    public static final DiscountStrategy NONE =
            price -> price;
}
```

**第三步：上下文类负责"使用"策略**

```java
public class PriceCalculator {

    private DiscountStrategy strategy;

    public PriceCalculator(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    // 随时切换策略
    public void setStrategy(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    public double calculate(double originalPrice) {
        return strategy.apply(originalPrice);
    }
}
```

**第四步：调用方代码**

```java
public class OrderService {

    // 用 Map 替代 if-else，策略查表
    private static final Map<String, DiscountStrategy> STRATEGY_MAP = new HashMap<>();

    static {
        STRATEGY_MAP.put("VIP",      DiscountStrategies.VIP);
        STRATEGY_MAP.put("SVIP",     DiscountStrategies.SVIP);
        STRATEGY_MAP.put("NEW_USER", DiscountStrategies.NEW_USER);
    }

    public double calcPrice(String discountType, double price) {
        DiscountStrategy strategy = STRATEGY_MAP.getOrDefault(
                discountType,
                DiscountStrategies.NONE   // 找不到类型时用默认策略
        );
        return new PriceCalculator(strategy).calculate(price);
    }
}
```

**效果对比：**

| 维度 | if-else 写法 | 策略模式 |
|------|------------|--------|
| 新增折扣类型 | 改原方法，高风险 | 加一行 Map，零风险 |
| 单元测试 | 整个方法一起测 | 每个策略单独测 |
| 代码行数 | 越积越长 | 职责清晰，短小精悍 |
| 复用性 | 差 | 策略可跨业务复用 |

---

## 三、场景二：支付方式选择

电商系统里，用户结账可以选：支付宝、微信、银行卡。每种支付的调用方式完全不同，但对外的接口是一样的——**"付钱"**。

```java
@FunctionalInterface
public interface PayStrategy {
    boolean pay(String orderId, double amount);
}
```

```java
public class PayStrategyFactory {

    private static final Map<String, PayStrategy> strategies = new HashMap<>();

    static {
        strategies.put("alipay",    (orderId, amount) -> {
            System.out.println("调用支付宝 API，订单=" + orderId + "，金额=" + amount);
            return true; // 实际项目这里调 SDK
        });
        strategies.put("wechat",    (orderId, amount) -> {
            System.out.println("调用微信支付 API，订单=" + orderId + "，金额=" + amount);
            return true;
        });
        strategies.put("bankcard",  (orderId, amount) -> {
            System.out.println("调用银行卡支付 API，订单=" + orderId + "，金额=" + amount);
            return true;
        });
    }

    public static PayStrategy get(String type) {
        PayStrategy strategy = strategies.get(type);
        if (strategy == null) {
            throw new IllegalArgumentException("不支持的支付方式: " + type);
        }
        return strategy;
    }
}
```

调用方：

```java
// 用户选择了微信支付
String payType = "wechat";
boolean success = PayStrategyFactory.get(payType).pay("ORDER-20260615-001", 299.0);
```

以后加个"数字人民币"支付？**只需在 static 块里加一行**，调用方代码完全不动。

---

## 四、场景三：保险报价计算（保险系统中的真实场景）

在保险核心系统中，不同险种的报价计算逻辑差异巨大：

- **车险**：根据车龄、座位数、出险记录计算
- **责任险**：根据营业额、保额档位计算
- **意外险**：根据年龄、职业类别计算

```java
// 报价上下文
public class PolicyQuoteContext {
    private String policyType;    // 险种
    private double baseAmount;    // 保额
    private int age;              // 被保人年龄
    private String occupation;    // 职业
    // ... 其他属性
}

// 报价策略接口
@FunctionalInterface
public interface QuoteStrategy {
    double calculate(PolicyQuoteContext ctx);
}

// 策略注册中心
public class QuoteStrategyRegistry {

    private static final Map<String, QuoteStrategy> REGISTRY = new HashMap<>();

    static {
        // 车险：保额 * 费率（简化示意）
        REGISTRY.put("car", ctx -> ctx.getBaseAmount() * 0.012);

        // 意外险：年龄越大费率越高
        REGISTRY.put("accident", ctx -> {
            double rate = ctx.getAge() < 40 ? 0.003 : 0.006;
            return ctx.getBaseAmount() * rate;
        });

        // 公众责任险：保额档位计费
        REGISTRY.put("liability", ctx -> {
            double baseAmount = ctx.getBaseAmount();
            if (baseAmount <= 100_000) return 800;
            if (baseAmount <= 500_000) return 2000;
            return 5000;
        });
    }

    public static double quote(String policyType, PolicyQuoteContext ctx) {
        return REGISTRY.getOrDefault(policyType, c -> {
            throw new RuntimeException("未知险种: " + policyType);
        }).calculate(ctx);
    }
}
```

这个模式在我们实际的 **prpins 公众责任险报价系统** 中非常有用——每个险种独立一个策略类，新险种上线只需注册，不动原有逻辑。

---

## 五、策略模式的结构图

```
┌─────────────────────────────────────────────────────┐
│                   调用方 (Client)                    │
│                                                     │
│   PriceCalculator calc = new PriceCalculator(       │
│       DiscountStrategies.VIP   ←── 注入策略          │
│   );                                                │
└──────────────────┬──────────────────────────────────┘
                   │ 使用
                   ▼
┌─────────────────────────────────────────────────────┐
│             上下文 (Context)                         │
│         PriceCalculator                             │
│                                                     │
│   - strategy: DiscountStrategy                      │
│   + calculate(price): double                        │
└──────────────────┬──────────────────────────────────┘
                   │ 调用 apply()
                   ▼
┌─────────────────────────────────────────────────────┐
│           策略接口 (Strategy)                        │
│        DiscountStrategy                             │
│                                                     │
│   + apply(price: double): double                    │
└───────┬──────────────────────────────┬──────────────┘
        │                              │
        ▼                              ▼
┌───────────────┐              ┌───────────────────┐
│  具体策略 A   │              │    具体策略 B      │
│  VIP 8折      │              │  SVIP 6折          │
└───────────────┘              └───────────────────┘
```

---

## 六、Java 8 Lambda 的加分点

在 Java 8 之前，每个策略都要写一个 class 文件，代码量大、文件多。

Java 8 的 Lambda + 函数式接口让策略模式**轻量化**：

```java
// Java 7 写法（需要单独的类或匿名类）
DiscountStrategy vip = new DiscountStrategy() {
    @Override
    public double apply(double price) {
        return price * 0.8;
    }
};

// Java 8 Lambda（一行搞定）
DiscountStrategy vip = price -> price * 0.8;
```

甚至可以直接传 Lambda 给上下文：

```java
// 临时策略：满 200 减 30
PriceCalculator calc = new PriceCalculator(price -> price >= 200 ? price - 30 : price);
System.out.println(calc.calculate(250)); // 输出 220.0
```

---

## 七、什么时候用策略模式？

| 适合用 ✅ | 不适合用 ❌ |
|----------|-----------|
| 多种行为只有算法不同 | 策略之间差异极小，一两行就能区分 |
| 未来会频繁扩展新行为 | 策略数量固定，永远不会增加 |
| 想对算法单独做单元测试 | 代码本就简单，无需抽象 |
| if-else / switch 已经很长 | 过度设计只会增加复杂度 |

---

## 总结

策略模式的核心思想：**把变化的部分（算法）封装起来，让不变的部分（使用算法的代码）稳定下来。**

Java 8 的函数式接口让策略模式写起来更爽——不用一堆类文件，Lambda 直接当策略用。

下次看到超长的 `if-else` 判断"做什么"，可以想想：**这里的每个分支，是不是一个可以独立管理的策略？**

---

> 本文结合 Java 保险核心系统的真实开发经验撰写，代码均可直接运行。  
> 如有问题，欢迎留言讨论 🌸
