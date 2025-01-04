---
date: 2024-03-15
title: Redisson分布式锁：高性能分布式锁解决方案
keywords: [Redisson, Redis, 分布式锁, 并发控制, 分布式系统, Java, Spring Boot, 锁实现, 高并发]
tags: [Redisson, 分布式锁, Redis, 并发控制, 分布式系统]
summary: 深入介绍了使用Redisson实现分布式锁的方案，包括Redisson的配置、锁的实现、工具类开发等内容，并提供了完整的代码示例和最佳实践。
author: 柳树下的程序员
---
Redisson 分布式锁

>`Redisson` 是基于redis【AP（可用性和分区容忍性）】的一套工具库，适用于对性能要求高、场景简单且无需强一致性的场景，例如高并发控制和缓存一致性

`推荐使用`: `redisson` 来实现分布式锁

`应用场景`： `Redisson` 是基于 `Redis` 实现的分布式锁工具，它利用 `Redis` 的高性能和多种数据结构提供了丰富的分布式锁功能。适用的场景包括：

1. 高并发访问控制: 例如，在秒杀、抢购、抢红包等场景中，需要对高并发的请求进行排队，避免数据冲突。
2. 分布式事务协调: 在分布式系统中，需要跨多个服务或资源进行事务操作时，通过 Redisson 的锁机制可以协调这些操作，确保一致性。
3. 缓存更新: 在需要频繁更新缓存的数据时，通过分布式锁可以确保缓存一致性，避免多个节点同时写入数据。
4. 任务调度: 在分布式系统中，通过分布式锁可以确保某个定时任务在某一时刻只有一个节点执行，避免重复执行。



**一、引入依赖**
```java
<dependency>
    <groupId>org.redisson</groupId>
    <artifactId>redisson-spring-boot-starter</artifactId>
    <version>3.34.1</version>
</dependency>
```
**二、初始化RedissonClient**
```java

import org.redisson.Redisson;
import org.redisson.api.RedissonClient;
import org.redisson.codec.JsonJacksonCodec;
import org.redisson.config.Config;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;


@Configuration
public class RedissonConfig {

    @Value("${spring.data.redis.host}")
    private String host;

    @Value("${spring.data.redis.port}")
    private String port;

    @Value("${spring.data.redis.database}")
    private int database;

    @Value("${spring.data.redis.password}")
    private String password;

    @Bean
    public RedissonClient redissonClient() {
        Config config = new Config().setCodec(JsonJacksonCodec.INSTANCE);
        config.useSingleServer().setAddress("redis://" + host + ":" + port).setDatabase(database).setPassword(password);
        return Redisson.create(config);
    }
}
```

**三、定义锁接口**
```java
public interface CustomLock {
    /**
     * 阻塞直到成功获取锁
     */
    void lock() throws InterruptedException;

    /**
     * 尝试获取锁，失败时抛出异常 （非阻塞）  ,
     */
    void tryLock() throws Exception;
    

    /**
     * 释放锁
     */
    void unlock();
}
```
**四、用RLock实现锁接口**
```java

import org.redisson.api.RLock;
import org.redisson.api.RedissonClient;

public class RedissonLockImpl implements CustomLock {
    private final RLock lock;

    public RedissonLockImpl(RedissonClient redissonClient, String lockKey) {
        this.lock = redissonClient.getLock(lockKey);
    }

    /**
     * 实现线程互斥的锁操作
     * 该方法使用一个外部的锁对象来确保线程安全和互斥访问
     * 在多线程环境下，确保同一时刻只有一个线程可以执行临界区代码
     *
     * @throws InterruptedException 如果线程在等待锁的过程中被中断，则会抛出此异常
     */
    @Override
    public void lock() throws InterruptedException {
        lock.lock();
    }


    /**
     * 尝试立即获取锁
     * 如果无法获取锁，则直接抛出异常
     *
     * @throws Exception 如果无法获取锁，抛出异常说明原因
     */
    @Override
    public void tryLock() throws Exception {
        if (!lock.tryLock()) {
            throw new Exception("RedissonLock：无法获取锁，立即抛出异常");
        }
    }

    /**
     * 实现接口中定义的解锁操作，通过调用内部锁对象的解锁方法来完成解锁
     */
    @Override
    public void unlock() {
        lock.unlock();
    }
}

```

**五、写一个操作锁的工具类**
1. 定义一个枚举类LockTypeEnum，用于表示锁的类型
```java
public enum LockTypeEnum {
    /**
     * 可重入锁
     */
    REENTRANT_LOCK,
    /**
     * redisson 的可重入锁
     */
    REDISSON_LOCK,
}


```
2. 创建一个工厂类LockFactory，用于创建自定义锁实例
```java

import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class LockFactory {


    static  RedissonClient redissonClient ;

    @Autowired
    public LockFactory(RedissonClient redissonClient) {
        LockFactory.redissonClient = redissonClient;
    }
    //存储锁实例
    //key->锁住的key，value->锁
    private static final Map<String, CustomLock> keyLockMap = new ConcurrentHashMap<>();

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
            case REENTRANT_LOCK:
                // 对于相同锁键，使用计算方式获取或创建一个可重入锁实例
                return keyLockMap.computeIfAbsent(lockKey, k -> new ReentrantLockImpl());
            case REDISSON_LOCK:
                return new RedissonLockImpl(redissonClient, lockKey);
            default:
                // 如果锁类型不是已知的类型之一，抛出异常
                throw new IllegalArgumentException("未知的锁类型: " + lockTypeEnum);
        }
    }
}
```

3. 创建一个工具类LockUtil，用于简化锁的使用

```java

import java.util.function.Supplier;

public class LockUtil {

    /**
     * 锁实例
     */
    private final CustomLock customLock;
    /**
     *加锁失败的提示语
     */
    private String failMessage;

    private LockUtil(LockTypeEnum lockTypeEnum, String lockKey) {
        this.customLock = LockFactory.createLock(lockTypeEnum, lockKey);
    }

    /**
     * 尝试获取锁
     *
     * 本方法用于创建一个锁工具实例，以便在业务操作中进行锁的管理和释放
     *
     * @param lockTypeEnum 锁的类型，如分布式锁、本地锁等，用于指定锁的实现方式
     * @param lockKey 锁的唯一标识，用于标识具体的锁对象，以便在并发环境中区分不同的锁
     * @return 返回一个LockUtil实例，包含指定的锁类型和锁键，用于后续的加锁和解锁操作
     */
    public static LockUtil tryLock(LockTypeEnum lockTypeEnum, String lockKey) {
        return new LockUtil(lockTypeEnum, lockKey);
    }

    /**
     * 设置失败信息
     *
     * @param message 失败信息的详细描述
     * @return 当前LockUtil实例，支持链式调用
     */
    public LockUtil failMsg(String message) {
        this.failMessage = message;
        return this;
    }

    /**
     * 非阻塞获取锁，失败则抛出异常
     * 该方法使用自定义锁来尝试执行给定的任务如果任务成功执行，它将被解锁
     * 如果获取锁失败或任务执行过程中出现异常，将打印错误消息
     *
     * @param task 要执行的锁任务
     */
    public void executeOnNonBlocking(Runnable task) {
        try {
            // 尝试获取自定义锁如果无法立即获取锁，此方法将返回false，而不是阻塞
            customLock.tryLock();
            try {
                // 在锁保护的范围内执行任务
                task.run();
            } finally {
                // 确保在任务完成后释放锁，无论任务执行成功还是出现异常
                customLock.unlock();
            }
        } catch (Exception e) {
            // 捕获在尝试获取锁或执行任务过程中可能抛出的任何异常，并打印相应的错误消息
            String thisFailMessage=failMessage != null ? failMessage : "锁获取失败";
            System.out.println(thisFailMessage);
            //自定义异常,将错误信息进行抛出
            //throw
        }
    }


    /**
     * 阻塞等待获取锁
     * 此方法通过获取自定义锁来确保任务的执行不会与其他任务并发进行
     * 如果在任务执行过程中发生异常，将根据failMessage变量是否有值打印相应的错误信息
     * 不管任务执行成功还是发生异常，最终都会释放自定义锁
     *
     * @param task 待执行的任务
     */
    public void executeOnBlock(Runnable task) {
        try {
            customLock.lock(); // 阻塞等待获取锁
            task.run();
        } catch (Exception e) {
            if (failMessage != null) {
                System.out.println(failMessage);
            }
            //自定义异常,将错误信息进行抛出
            //throw
        } finally {
            customLock.unlock();
        }
    }


    /**
     * 阻塞等待获取锁（有返回值）
     * 此方法通过获取自定义锁来确保任务的执行不会与其他任务并发进行
     * 如果在任务执行过程中发生异常，将根据failMessage变量是否有值打印相应的错误信息
     * 不管任务执行成功还是发生异常，最终都会释放自定义锁
     *
     * @param task 待执行的任务
     */
    public <T> T executeOnBlock(Supplier<T> task) {
        try {
            customLock.lock(); // 阻塞等待获取锁
            return task.get();
        } catch (Exception e) {
            if (failMessage != null) {
                System.out.println(failMessage);
            }
            //建议自定义异常,将错误信息进行抛出 ,不要返回 null
            //throw
            return null;
        } finally {
            customLock.unlock();
        }
    }

}
```
4. 使用示例
```java

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class TestTestController {
    @GetMapping("/lock/test")
    public void test(){
        // 创建一个共享的任务实例
        Runnable task = () -> {
            try {
                // 模拟任务执行
                Thread.sleep(2000); // 睡眠2秒
                System.out.println(Thread.currentThread().getName() + " 完成任务");
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        };

        // 启动多个线程来测试非阻塞锁
        for (int i = 0; i < 3; i++) {
            new Thread(() -> {
                LockUtil.tryLock(LockTypeEnum.REDISSON_LOCK, "testLock")
                        .failMsg("ReentrantLock 锁获取失败，请稍后再试")
                        .executeOnNonBlocking(() -> {
                            System.out.println(Thread.currentThread().getName() + " 正在执行任务...");
                            task.run();
                        });
            }, "线程" + i).start();
        }

        // 启动多个线程来测试阻塞锁
        for (int i = 0; i < 3; i++) {
            new Thread(() -> {
                LockUtil.tryLock(LockTypeEnum.REDISSON_LOCK, "testLock")
                        .executeOnBlock(() -> {
                            System.out.println(Thread.currentThread().getName() + " 正在执行任务...");
                            task.run();
                        });
            }, "线程" + i).start();
        }
    }
}
```

>六、如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！