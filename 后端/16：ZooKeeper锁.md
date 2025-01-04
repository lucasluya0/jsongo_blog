---
date: 2024-03-15
title: ZooKeeper分布式锁实现详解：从原理到实战
keywords: [ZooKeeper, 分布式锁, 分布式系统, 并发控制, Java, Curator, 分布式协调, InterProcessMutex, 实战教程]
tags: [ZooKeeper, 分布式锁, 分布式协调, 并发控制, 分布式系统]
summary: 详细介绍了基于ZooKeeper实现分布式锁的方案，包括ZooKeeper的配置、锁的实现机制、实际应用场景等内容，并提供了完整的代码实现。
author: 柳树下的程序员
---
实战基础-分布式锁 ZooKeeper

>`Zookeeper` 是一种实现了 `CP（Consistency and Partition tolerance）` 模型的分布式协调服务。这意味着 `Zookeeper` 在分布式系统中重点关注一致性和分区容忍性，牺牲了一部分可用性以保证数据的一致性。

![img.png](image%2Fimg.png)

`推荐使用`：

1. 分布式协调: 例如，分布式应用中的任务调度、配置管理、服务发现等。
2. 领导者选举: 在分布式系统中，Zookeeper 可以帮助选举出一个领导者来协调其他节点的工作。
3. 配置管理: 集中管理分布式系统的配置，确保所有节点配置的一致性。


>一、引入依赖
```
<dependency>
    <groupId>org.springframework.integration</groupId>
    <artifactId>spring-integration-zookeeper</artifactId>
</dependency>
```
>二、初始化CuratorFramework（也就是创建一个ZookeeperClient实例）
```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import java.nio.charset.StandardCharsets;

@Configuration
public class ZookeeperConfig {

    @Bean
    public CuratorFramework curatorFramework() {
        // 配置 Zookeeper 连接字符串和重试策略
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("127.0.0.1:2181") // 替换为你的 Zookeeper 地址
                .retryPolicy(new ExponentialBackoffRetry(1000, 3))
                .authorization("digest", "admin:admin123".getBytes(StandardCharsets.UTF_8)) // 明确指定字符集
                .build();

        client.start(); // 启动客户端
        return client;
    }
}
```

>三、定义锁接口
```java

public interface CustomLock {
    /**
     * 阻塞直到成功获取锁
     */
    void lock() throws Exception;

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
>四、用InterProcessMutex实现锁接口
```java

public class ZookeeperLockImpl implements CustomLock {

    private final InterProcessMutex lock;

    /**
     * 构造函数，初始化分布式锁
     *
     * @param client CuratorFramework实例，用于操作ZooKeeper
     * @param lockKey 分布式锁的路径，用于唯一标识锁  ，一定是一个路径（原因：分布式锁实现方式是：顺序临时节点）
     */
    public ZookeeperLockImpl(CuratorFramework client, String lockKey) {
        this.lock = new InterProcessMutex(client, lockKey);
    }

    /**
     * 实现锁的获取
     * 通过调用lock对象的acquire方法来实现，此方法会阻塞当前线程直到锁可用
     *
     * @throws Exception 如果获取锁失败，则抛出异常
     */
    @Override
    public void lock() throws Exception {
        lock.acquire();
    }

    /**
     * 尝试立即获取锁，如果无法获取则抛出异常。
     * 此方法用于实现需要立即获得锁的操作，不允许等待。
     *
     * @throws Exception 如果无法立即获取锁，将抛出异常，说明无法成功获取锁。
     */
    @Override
    public void tryLock() throws Exception {
        // 尝试立即获取锁，不等待。
        // 如果无法获取锁，lock.acquire方法返回false，随后抛出异常。
        if (!lock.acquire(0, java.util.concurrent.TimeUnit.SECONDS)) {
            throw new Exception("ZookeeperLock：无法获取锁，立即抛出异常");
        }
    }

    /**
     * 释放锁
     * 该方法覆盖自接口，旨在释放当前持有的锁
     * 它是线程安全的一个关键部分，确保在不再需要锁时释放它，以避免死锁情况
     */
    @Override
    public void unlock() {
        try {
            // 尝试释放锁
            lock.release();
        } catch (Exception e) {
            // 如果释放锁时出现异常，则打印异常堆栈跟踪
            e.printStackTrace();
        }
    }
}

```

>五、写一个操作锁的工具类


**1. 定义一个枚举类LockTypeEnum，用于表示锁的类型**
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
    /**
     * zookeeper 的锁
     */
    ZOOKEEPER_LOCK,
}

```
**2. 创建一个工厂类`LockFactory`，用于创建自定义锁实例**
```java

import org.apache.curator.framework.CuratorFramework;
import org.redisson.api.RedissonClient;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class LockFactory {


    static  RedissonClient redissonClient ;
    private static  CuratorFramework zookeeperClient;

    @Autowired
    public LockFactory(RedissonClient redissonClient, CuratorFramework zookeeperClient) {
        LockFactory.redissonClient = redissonClient;
        LockFactory.zookeeperClient = zookeeperClient;
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
            case ZOOKEEPER_LOCK:
                return new ZookeeperLockImpl(zookeeperClient, lockKey);
            default:
                // 如果锁类型不是已知的类型之一，抛出异常
                throw new IllegalArgumentException("未知的锁类型: " + lockTypeEnum);
        }
    }
}
```

**3. 创建一个工具类`LockUtil`，用于简化锁的使用**

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

    /**
     * 构造函数：根据锁类型和锁键创建一个自定义锁实例。
     *
     * @param lockTypeEnum 锁类型枚举，用于指定锁的类型（如 Redis 锁、Zookeeper 锁等）。
     * @param lockKey 锁的唯一键值，用于标识锁。
     *
     * <p>示例:</p>
     * <ul>
     *   <li>如果 <code>lockTypeEnum</code> 为 {@link LockTypeEnum#ZOOKEEPER_LOCK}，则 <code>lockKey</code> 必须是一个 Zookeeper 路径（如 <code>"/xx"</code>）。</li>
     *   <li>如果 <code>lockTypeEnum</code> 为 {@link LockTypeEnum#REDISSON_LOCK} 或 {@link LockTypeEnum#REENTRANT_LOCK}，则 <code>lockKey</code> 可以是任意字符串。</li>
     * </ul>
     */
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
**4. 使用示例**
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
                LockUtil.tryLock(LockTypeEnum.ZOOKEEPER_LOCK, "/testLock")
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
                LockUtil.tryLock(LockTypeEnum.ZOOKEEPER_LOCK, "/testLock")
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
