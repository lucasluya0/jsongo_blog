---
date: 2024-03-15
title: Spring Bean生命周期实战：工具类开发与依赖注入最佳实践
keywords: [Spring Bean, 生命周期, 工具类开发, 依赖注入, Bean管理, 实战应用, SpringContextUtil, RedissonClient]
tags: [Spring Bean, 工具类开发, 依赖注入, Bean管理, 实战应用]
summary: 通过实战案例详细介绍了Spring Bean生命周期的应用，重点讲解了如何利用Bean生命周期特性开发工具类，包括多种依赖注入方式的实现。
author: 柳树下的程序员
---
实战工具类-Spring Bean 生命周期

Spring Bean 生命周期是指 Spring 容器管理 Bean 的整个生命周期，从 Bean 的创建、初始化到销毁。在企业开发中，Bean 的生命周期 plays 着举足轻重的角色，因为 Bean 是企业应用的核心部件，它负责应用的业务逻辑和数据处理。

**`推荐使用：`** 利用 SpringContextUtil 进行手动获取

在编写工具类时，通常会将 Spring Bean 注入到工具类的静态变量中以便于使用。以下是几种可能的实现方式:
> 1. 使用 @Autowired 和 @PostConstruct
```java

import jakarta.annotation.PostConstruct;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class LockFactory {

    /**
     * Redisson客户端实例，用于获取分布式锁
     */
    private final RedissonClient redissonClientTemp;
    
    /**
     * 静态Redisson客户端实例，用于被多个对象共享
     */
    private static RedissonClient redissonClient;
    
    /**
     * 通过构造函数注入Redisson客户端
     * 
     * @param redissonClient Redisson客户端实例
     */
    @Autowired
    public LockFactory(RedissonClient redissonClient) {
        this.redissonClientTemp = redissonClient;
    }
    
    /**
     * 在对象加载完成后初始化静态Redisson客户端实例
     * 这个方法确保了在对象实例化后，将临时Redisson客户端实例设置为静态成员变量
     */
    @PostConstruct
    public void init() {
        LockFactory.redissonClient = this.redissonClientTemp;
    }
    

    /**
     * 根据指定的锁类型和锁键创建自定义锁实例
     *
     * @param lockTypeEnum 锁的类型，决定了创建锁的实现类型
     * @param lockKey 锁的键，用于区分不同锁实例，以便于管理和重用
     * @return 返回一个自定义锁实例，用于管理和控制并发访问共享资源
     * @throws IllegalArgumentException 如果锁类型未知，抛出此异常
     */
    public static CustomLock createLock(LockTypeEnum lockTypeEnum, String lockKey) {
        // 根据锁类型选择合适的锁实现
        switch (lockTypeEnum) {
            case REDISSON_LOCK:
                return new RedissonLockImpl(redissonClient, lockKey);
            default:
                // 如果锁类型不是已知的类型之一，抛出异常
                throw new IllegalArgumentException("未知的锁类型: " + lockTypeEnum);
        }
    }
}

```

> 2. 使用 SpringContextUtil 进行手动获取

可以通过一个 Spring 工具类（SpringContextUtil）来手动获取 RedissonClient 并赋值给静态变量：
- 创建一个工具类，用于获取 Spring 上下文，然后通过 getBean 方法获取 Bean。
```java
import org.springframework.context.ApplicationContext;
import org.springframework.context.ApplicationContextAware;
import org.springframework.context.ApplicationEvent;
import org.springframework.stereotype.Component;
/**
 * SpringContextUtil工具类，用于全局访问Spring应用上下文ApplicationContext。
 * 通过实现ApplicationContextAware接口，自动注入ApplicationContext。
 */
@Component
public class SpringContextUtil implements ApplicationContextAware {

    /**
     * Spring应用上下文，静态变量，供全局访问。
     */
    private static ApplicationContext applicationContext;

    /**
     * 实现ApplicationContextAware接口的回调方法，用于注入应用上下文。
     *
     * @param applicationContext Spring应用上下文
     */
    @Override
    public void setApplicationContext(ApplicationContext applicationContext) {
        SpringContextUtil.applicationContext = applicationContext;
    }

    /**
     * 通过bean名称获取bean实例。
     *
     * @param name bean的名称
     * @param <T>  bean的类型
     * @return 符合类型的bean实例
     */
    public static <T> T getBean(String name) {
        return (T) applicationContext.getBean(name);
    }

    /**
     * 通过bean类型获取bean实例。
     *
     * @param clazz bean的类型
     * @param <T>   bean的类型
     * @return 符合类型的bean实例
     */
    public static <T> T getBean(Class<T> clazz) {
        return applicationContext.getBean(clazz);
    }

    /**
     * 发布事件到应用上下文。
     *
     * @param event 需要发布的事件
     */
    public static void publishEvent(ApplicationEvent event) {
        applicationContext.publishEvent(event);
    }

}
```
- 在工具类中使用 SpringContextUtil 进行获取
```java

import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class LockFactory {
    // RedissonClient实例，用于与Redis进行通信实现分布式锁
    private static RedissonClient redissonClient;

    /**
     * LockFactory构造函数，用于注入RedissonClient实例
     * 通过@Autowired注解，Spring框架会自动为LockFactory注入一个RedissonClient实例
     * 这个构造函数的作用是初始化LockFactory中的RedissonClient实例，以便于后续使用
     *
     * @param redissonClient RedissonClient实例，用于分布式锁的实现
     */
    @Autowired
    public LockFactory(RedissonClient redissonClient) {
        LockFactory.redissonClient = redissonClient;
    }


    /**
     * 根据指定的锁类型和锁键创建自定义锁实例
     *
     * @param lockTypeEnum 锁的类型，决定了创建锁的实现类型
     * @param lockKey 锁的键，用于区分不同锁实例，以便于管理和重用
     * @return 返回一个自定义锁实例，用于管理和控制并发访问共享资源
     * @throws IllegalArgumentException 如果锁类型未知，抛出此异常
     */
    public static CustomLock createLock(LockTypeEnum lockTypeEnum, String lockKey) {
        // 根据锁类型选择合适的锁实现
        switch (lockTypeEnum) {
            case REDISSON_LOCK:
                return new RedissonLockImpl(redissonClient, lockKey);
            default:
                // 如果锁类型不是已知的类型之一，抛出异常
                throw new IllegalArgumentException("未知的锁类型: " + lockTypeEnum);
        }
    }
}
```
> 3. 使用构造函数注入和静态块

通过构造函数注入 RedissonClient，然后在静态块中将其赋值给静态变量
```java

import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class LockFactory {

    // RedissonClient实例，用于与Redis进行交互，实现分布式锁的功能
    private static RedissonClient redissonClient;
    
    // 构造函数：初始化LockFactory时，注入RedissonClient实例
    // 参数redissonClient：用于分布式锁操作的RedissonClient实例，通过@Autowired自动注入
    @Autowired
    public LockFactory(RedissonClient redissonClient) {
        LockFactory.redissonClient = redissonClient;
    }

    /**
     * 根据指定的锁类型和锁键创建自定义锁实例
     *
     * @param lockTypeEnum 锁的类型，决定了创建锁的实现类型
     * @param lockKey 锁的键，用于区分不同锁实例，以便于管理和重用
     * @return 返回一个自定义锁实例，用于管理和控制并发访问共享资源
     * @throws IllegalArgumentException 如果锁类型未知，抛出此异常
     */
    public static CustomLock createLock(LockTypeEnum lockTypeEnum, String lockKey) {
        // 根据锁类型选择合适的锁实现
        switch (lockTypeEnum) {
            case REDISSON_LOCK:
                return new RedissonLockImpl(redissonClient, lockKey);
            default:
                // 如果锁类型不是已知的类型之一，抛出异常
                throw new IllegalArgumentException("未知的锁类型: " + lockTypeEnum);
        }
    }
}

```


> 4. 使用 @Autowired 静态方法注入

虽然 @Autowired 通常用于非静态字段，但你可以通过一个静态方法来注入 RedissonClient:
```java

import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class LockFactory {
    // RedissonClient实例，用于与Redis服务器进行交互
    private static RedissonClient redissonClient;
    
    /**
     * 设置RedissonClient实例
     * 该方法通过Spring的自动装配机制，接收并设置RedissonClient实例
     * 使得LockFactory能够使用这个共享的Redis连接
     * 
     * @param redissonClient RedissonClient实例，用于与Redis服务器进行交互
     */
    @Autowired
    public void setRedissonClient(RedissonClient redissonClient) {
        LockFactory.redissonClient = redissonClient;
    }

    /**
     * 根据指定的锁类型和锁键创建自定义锁实例
     *
     * @param lockTypeEnum 锁的类型，决定了创建锁的实现类型
     * @param lockKey 锁的键，用于区分不同锁实例，以便于管理和重用
     * @return 返回一个自定义锁实例，用于管理和控制并发访问共享资源
     * @throws IllegalArgumentException 如果锁类型未知，抛出此异常
     */
    public static CustomLock createLock(LockTypeEnum lockTypeEnum, String lockKey) {
        // 根据锁类型选择合适的锁实现
        switch (lockTypeEnum) {
            case REDISSON_LOCK:
                return new RedissonLockImpl(redissonClient, lockKey);
            default:
                // 如果锁类型不是已知的类型之一，抛出异常
                throw new IllegalArgumentException("未知的锁类型: " + lockTypeEnum);
        }
    }
}

```
>5、如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！