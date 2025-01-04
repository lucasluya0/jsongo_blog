---
date: 2024-03-15
title: Java Optional最佳实践：优雅处理空指针的完美方案
keywords: [Java Optional, 空指针处理, NPE, 函数式编程, 代码优化, Optional最佳实践, Java 8, 链式调用, 空值处理]
tags: [Java Optional, 空指针处理, 函数式编程, 代码优化, 最佳实践]
summary: 深入讲解了Java Optional的使用技巧和最佳实践，包括如何优雅处理空值、链式调用、实际应用场景等内容，帮助开发者写出更健壮的代码。
author: 柳树下的程序员
---
别再写 null 判断了！Java Optional 才是解决空指针的最佳实践
> 你还在写一堆 if(xxx != null) 吗？来看看如何用 Optional 优雅地解决空指针问题！
1. 从一个真实场景说起
今天无意间从项目中看到这样一个代码片段：
```java
public String getUserAddress(User user) {
    if (user != null) {
        Company company = user.getCompany();
        if (company != null) {
            Address address = company.getAddress();
            if (address != null) {
                String city = address.getCity();
                if (city != null) {
                    return city;
                }
            }
        }
    }
    return "Unknown";
}

```
看到这样层层嵌套的 null 检查，你是否感到头疼？这样的代码不仅难以维护，而且极易出错。一旦漏掉某个 null 检查，就可能导致著名的 `NullPointerException（NPE）`。
据统计，`NullPointerException` 是 `Java` 中最常见的异常之一，约占所有异常的 70%。那么，如何优雅地解决这个问题呢？

> Optional：优雅的空值处理方案
- 2.1 什么是 `Optional`？

`Optional` 是 `Java 8` 引入的一个很有用的工具类，它可以看作是一个容器，用于表示一个值是否存在。Optional 的本质是对 null 的封装，它提供了一种更优雅的方式来处理可能为 null 的对象。
- 2.2 `Optional` 的核心方法

```java
// 1. 创建 Optional 对象
Optional<String> optional1 = Optional.of("Hello");           // 值不能为 null
Optional<String> optional2 = Optional.ofNullable(null);      // 值可以为 null
Optional<String> optional3 = Optional.empty();               // 创建空的 Optional

// 2. 判断值是否存在
optional1.isPresent();    // 返回 true
optional2.isPresent();    // 返回 false

// 3. 获取值
optional1.get();          // 返回 "Hello"
optional2.get();          // 抛出 NoSuchElementException

// 4. 安全地获取值
optional1.orElse("default");              // 值不存在时返回默认值
optional1.orElseGet(() -> "default");     // 值不存在时通过供应商函数获取默认值
optional1.orElseThrow(() -> new RuntimeException());  // 值不存在时抛出异常
```
> 实战案例：改造之前的代码

让我们用 Optional 改造开头的示例代码：
```java
public String getUserAddress(User user) {
    return Optional.ofNullable(user)
            .map(User::getCompany)
            .map(Company::getAddress)
            .map(Address::getCity)
            .orElse("Unknown");
}
```

是不是一下子变得清爽多了？这就是 Optional 的魅力！

> Optional 的最佳实践

4.1 使用 map() 进行对象转换
map() 方法用于对 Optional 对象的值进行转换，并返回一个新的 Optional 对象。
```java
// 场景：获取用户的邮箱域名
public String getEmailDomain(User user) {
    return Optional.ofNullable(user)
            .map(User::getEmail)
            .filter(email -> email.contains("@"))
            .map(email -> email.substring(email.indexOf("@") + 1))
            .orElse("unknown");
}
```
4.2 使用 filter() 进行过滤
filter() 方法用于对 Optional 对象的值进行过滤，如果值满足条件则返回 Optional 对象，否则返回空的 Optional 对象。
```java
// 场景：获取成年用户的姓名
public String getAdultName(User user) {
    return Optional.ofNullable(user)
            .filter(u -> u.getAge() >= 18)
            .map(User::getName)
            .orElse("Unnamed");
}
```
4.3 orElse() vs orElseGet() 的重要区别
orElse() 方法在值不存在时返回默认值，而 orElseGet() 方法在值不存在时通过供应商函数获取默认值。
```java
// 示例：展示 orElse 和 orElseGet 的区别
public class OptionalDemo {
    public static String getDefaultValue() {
        System.out.println("Computing default value...");
        return "Default";
    }

    public static void main(String[] args) {
        Optional<String> optional = Optional.of("Hello");
        
        // orElse() 会执行 getDefaultValue()
        String result1 = optional.orElse(getDefaultValue());
        
        // orElseGet() 不会执行 getDefaultValue()
        String result2 = optional.orElseGet(() -> getDefaultValue());
    }
}
```

4.4 避免的错误用法
❌ 错误示例：
```java
// 1. 不要使用 Optional.get() 而不检查
optional.get();  // 可能抛出 NoSuchElementException

// 2. 不要将 Optional 作为字段类型
public class User {
    private Optional<String> name;  // 错误！
}

// 3. 不要将 Optional 作为方法参数
public void processUser(Optional<User> user) {  // 错误！
}

// 4. 不要用 Optional 包装集合类型
Optional<List<String>> optionalList;  // 没必要，直接用空集合即可
```

✅ 正确示例：
```java
// 1. 安全地获取值
optional.orElse("default");

// 2. 正确的字段定义
public class User {
    private String name;
}

// 3. 正确的方法参数
public void processUser(User user) {
    Optional.ofNullable(user)
           .ifPresent(this::process);
}

// 4. 直接使用空集合
List<String> list = Collections.emptyList();
```

5. 实际业务场景示例
5.1 订单处理系统
```java
public class OrderService {
    public OrderDTO processOrder(String orderId) {
        return Optional.ofNullable(orderRepository.findById(orderId))
                .filter(order -> order.getStatus() != OrderStatus.CANCELLED)
                .map(order -> {
                    order.setStatus(OrderStatus.PROCESSING);
                    order.setProcessTime(LocalDateTime.now());
                    orderRepository.save(order);
                    return orderConverter.toDTO(order);
                })
                .orElseThrow(() -> new OrderNotFoundException("Order not found: " + orderId));
    }
}
```
5.2 用户配置处理
```java
public class UserConfigService {
    public UserConfig getUserConfig(String userId) {
        return Optional.ofNullable(userConfigRepository.findByUserId(userId))
                .map(config -> {
                    if (config.isExpired()) {
                        return refreshConfig(config);
                    }
                    return config;
                })
                .orElseGet(() -> createDefaultConfig(userId));
    }
}
```
>如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！