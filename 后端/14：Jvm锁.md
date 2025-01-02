实战基础-锁

在多线程环境下，锁是确保线程安全的关键工具。线程安全指的是在并发环境中多个线程同时运行时，程序的行为仍然是正确的。锁通过序列化对共享资源的访问来确保这一点。

`作用：`
- 可以避免重复处理数据（非阻塞锁实现）
- 避免数据竞争（阻塞锁实现）

**单机版：用ReentrantLock 实现。**

>一、定义锁接口

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
>二、用ReentrantLock实现锁接口
```java

import java.util.concurrent.locks.ReentrantLock;

public class ReentrantLockImpl implements CustomLock {
    private final ReentrantLock lock = new ReentrantLock();
    /**
     * 阻塞直到成功获取锁
     */
    @Override
    public void lock() throws InterruptedException {
        lock.lock();
    }

    /**
     * 尝试获取锁，失败时抛出异常
     */
    @Override
    public void tryLock() throws Exception {
        if (!lock.tryLock()) {
            throw new Exception("ReentrantLock：无法获取锁，立即抛出异常");
        }
    }

    /**
     * 释放锁
     */
    @Override
    public void unlock() {
        lock.unlock();
    }
}
```
>三、写一个操作锁的工具类

**1. 定义一个枚举类LockTypeEnum，用于表示锁的类型**
```java
public enum LockTypeEnum {
    /**
     * 可重入锁
     */
    REENTRANT_LOCK,
}

```
**2. 创建一个工厂类LockFactory，用于创建自定义锁实例**
```java

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public class LockFactory {

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
            default:
                // 如果锁类型不是已知的类型之一，抛出异常
                throw new IllegalArgumentException("未知的锁类型: " + lockTypeEnum);
        }
    }
}
```

**3. 创建一个工具类LockUtil，用于简化锁的使用**

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
**4. 使用示例**
```java

public class LockExample {
    public static void main(String[] args) {
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
                LockUtil.tryLock(LockTypeEnum.REENTRANT_LOCK, "testLock")
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
                LockUtil.tryLock(LockTypeEnum.REENTRANT_LOCK, "testLock")
                        .executeOnBlock(() -> {
                            System.out.println(Thread.currentThread().getName() + " 正在执行任务...");
                            task.run();
                        });
            }, "线程" + i).start();
        }
    }
}
```
>四、如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！