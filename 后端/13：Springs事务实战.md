Spring下的事务管理实战

`推荐做法：`在企业中尽可能使用编程式事务写法，应避免使用注解式写法@Transactional。
- 由于事务管理可能需要与同步锁结合使用，以确保并发环境下的数据一致性和线程安全。一般将事务管理只能放在同步锁内部使用
- `@Transactional`注解放在方法上容易引发长事务问题，而长事物会带来锁的竞争影响性能，同时也会导致数据库连接池被耗尽，影响程序的正常执行
- `@Transactional`注解放在方法上时，容易出现漏写（`rollbackFor属性值`）

>一、编程式事务管理 

**1 推荐使用`TransactionTemplate` 可以进行平替@Transactional**
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.support.TransactionTemplate;

@Service
public class TestServiceImpl {

    @Autowired
    TransactionTemplate transactionTemplate;

    /**
     *
     * 通过这个方法，我们可以封装事务的开始和结束逻辑，确保在执行块内的操作时统一管理事务
     * 这里没有具体的业务逻辑，只是为了展示如何使用事务模板执行一个事务操作
     */
    public void test() {

        // 使用事务模板执行一个匿名内部类，该内部类实际上没有进行任何操作，只是用作演示
        Boolean testReturnValue=transactionTemplate.execute(status -> {
            // 业务代码
            return true;
        });
        System.out.println(testReturnValue);
    }

}
```
**2 使用`PlatformTransactionManager`更加精细化控制事务**
```java

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.PlatformTransactionManager;
import org.springframework.transaction.TransactionDefinition;
import org.springframework.transaction.TransactionStatus;
import org.springframework.transaction.support.DefaultTransactionDefinition;

import java.util.UUID;

@Service
public class TestServiceImpl {

    @Autowired
    SysMenuMapper sysMenuMapper;
    @Autowired
    SysTestMapper sysTestMapper;

    @Autowired
    private PlatformTransactionManager transactionManager;

    /**
     * 在事务中执行操作
     * 它使用了编程式事务管理，适用于需要手动控制事务边界的情况
     */
    public void executeInTransaction() {
        SysTest sysTest = new SysTest();
        sysTest.setId(UUID.randomUUID().toString());
        sysTest.setName("张哈");

        SysMenu sysMenu = new SysMenu();
        sysMenu.setId(UUID.randomUUID().toString());
        sysMenu.setShowName("菜单");


        // 创建一个默认的事务定义
        DefaultTransactionDefinition def = new DefaultTransactionDefinition();
        // 设置事务传播行为为REQUIRED，这是最常见的选择
        // 表示如果当前存在事务则加入该事务，如果当前没有事务则创建一个新的事务
        def.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        // 设置事务隔离级别为READ_COMMITTED，表示读取已提交的数据
        // 这是大多数情况下的最佳选择，因为它提供了良好的隔离性同时又能保持性能
        def.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);

        // 从事务管理器中获取当前的事务状态
        TransactionStatus status = transactionManager.getTransaction(def);
        try {
            // 执行业务逻辑（本例中未显示具体业务逻辑代码）
            sysMenuMapper.insert(sysMenu);
            sysTestMapper.insert(sysTest);
            // 如果业务逻辑执行成功，则提交事务
            transactionManager.commit(status);
        } catch (Exception ex) {
            // 如果业务逻辑执行失败，则回滚事务
            transactionManager.rollback(status);
            // 重新抛出异常
            throw ex;
        }
    }
}
```
>二、声明式事务管理
```java
@Transactional(rollbackFor = Exception.class)
public void exeInsertMethod(){

}
```
>三、事务的回滚规则

- 默认情况下：运行时异常（RuntimeException）和错误（Error）会导致事务回滚，检查异常（Checked Exception）不会。
- 自定义回滚规则：使用 rollbackFor 和 noRollbackFor 属性来指定哪些异常会导致事务回滚。

**ps: `@Transactional`**

默认情况下，使用 @Transactional 注解的方法在遇到运行时异常（RuntimeException）或错误（Error）时会触发事务回滚。对于检查异常（Checked Exception），事务不会自动回滚，除非在方法内部显式调用 TransactionStatus#setRollbackOnly()。
- 当抛出 RuntimeException 时，事务将会回滚。
- 当抛出 IOException 时，事务不会回滚。
```java
@Transactional
public void defaultRollbackMethod() throws IOException {
    // 抛出运行时异常（RuntimeException）
    if (someCondition) {
        throw new RuntimeException("Runtime exception occurred");
    }
    // 抛出检查异常（Checked Exception）
    if (someOtherCondition) {
        throw new IOException("Checked exception occurred");
    }
}

```

**ps: `@Transactional(rollbackFor = Exception.class)`**

使用 `@Transactional(rollbackFor = Exception.class)` 注解的方法不仅会在遇到运行时异常（RuntimeException）和错误（Error）时回滚事务，还会在遇到所有类型的异常（包括检查异常，Checked Exception）时也回滚事务。
- 当抛出 RuntimeException 时，事务将会回滚。
- 当抛出 IOException 时，事务也会回滚。
```java
@Transactional(rollbackFor = Exception.class)
public void customRollbackMethod() throws IOException {
    // 抛出运行时异常（RuntimeException）
    if (someCondition) {
        throw new RuntimeException("Runtime exception occurred");
    }
    // 抛出检查异常（Checked Exception）
    if (someOtherCondition) {
        throw new IOException("Checked exception occurred");
    }
}
```

>四、检查异常 vs 运行时异常

**`1、检查异常（Checked Exception）：`**

- 在编译时检查，必须在方法签名中声明或捕获处理。
- 例子包括 IOException、SQLException、ClassNotFoundException 等。
- 必须通过 throws 关键字声明或使用 try-catch 块捕获。

**`2、运行时异常（RuntimeException）：`**

- 在运行时抛出，不需要在方法签名中声明或捕获处理。
- 例子包括 NullPointerException、IndexOutOfBoundsException、IllegalArgumentException 等。
- 可以选择捕获处理，但不强制要求。

>五、如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！