---
date: 2024-03-15
title: Spring Bean生命周期详解：从创建到销毁的完整过程
keywords: [Spring Bean, Bean生命周期, 依赖注入, 初始化方法, Bean实例化, ApplicationContextAware, BeanPostProcessor, PostConstruct, InitializingBean, DisposableBean]
tags: [Spring Bean, 生命周期, 依赖注入, Bean初始化, Spring框架]
summary: 深入探讨了Spring Bean的完整生命周期，包括实例化、依赖注入、初始化等各个阶段，并结合实际应用场景说明了各个生命周期阶段的作用。
author: 柳树下的程序员
---
探索 Spring Bean 生命周期及其在企业开发中的应用

`推荐理由：`在`Spring`项目中，`sao`操作能用到
>一、Spring 提供了多种机制来管理 Bean 的创建、初始化和销毁。Spring Bean 的生命周期主要包括以下几个阶段：
- Bean 实例化
- 依赖注入
- 调用 ApplicationContextAware.setApplicationContext
- 调用 BeanPostProcessor.postProcessBeforeInitialization
- 调用 @PostConstruct 注解的方法 或 InitializingBean.afterPropertiesSet
- 调用 init-method 或 @Bean(initMethod = "method")
- 调用 BeanPostProcessor.postProcessAfterInitialization
- Bean 使用
- Bean 销毁



>1、Bean 实例化：在 Spring 容器启动时，会根据配置的 Bean 定义创建 Bean 实例。

**应用场景：**
- 对象创建：Spring 容器通过构造函数或工厂方法创建 Bean 实例。例如，一个服务类 UserService 通过构造函数实例化，并在 Bean 的生命周期中注入依赖。
```java
@Component
public class UserService {
    private final UserRepository userRepository;

    @Autowired
    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }
}
```
>2、依赖注入：在 Bean 实例化之后，Spring 会根据配置的依赖关系，将依赖注入到 Bean 中。

**应用场景：**
- 依赖管理：通过注入服务、DAO 或其他组件来解耦系统组件。例如，将 UserRepository 注入到 UserService 中，实现数据访问的解耦。
```java
@Component
public class OrderService {
    @Autowired
    private UserService userService;
}
```
>3、调用 `ApplicationContextAware.setApplicationContext：`在 Bean 实例化之后，Spring 会调用 `ApplicationContextAware.setApplicationContext` 方法，将 `ApplicationContext` 注入到 Bean 中。

**应用场景：**
- 访问容器资源：需要在 Bean 中访问 ApplicationContext 来获取其他 Bean 或发布事件。例如，在一个 Bean 中动态获取其他 Bean，或者发布自定义事件
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
>4、调用 `BeanPostProcessor.postProcessBeforeInitialization：`在 Bean 实例化之后，Spring 会调用 BeanPostProcessor.postProcessBeforeInitialization 方法，允许在 Bean 初始化之前进行自定义操作。

**应用场景：**
- 动态代理和修改：用于创建代理、修改 Bean 或在初始化之前进行自定义操作。例如，自动为服务 Bean 添加日志功能。
```java
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        if (bean instanceof SomeService) {
            // Wrap the bean with a proxy to add logging functionality
            return Proxy.newProxyInstance(bean.getClass().getClassLoader(),
                    bean.getClass().getInterfaces(),
                    (proxy, method, args) -> {
                        System.out.println("Before method: " + method.getName());
                        Object result = method.invoke(bean, args);
                        System.out.println("After method: " + method.getName());
                        return result;
                    });
        }
        return bean;
    }
}
```
>5、调用 `@PostConstruct` 注解的方法 或 `InitializingBean.afterPropertiesSet ：`在 Bean 实例化之后，Spring 会调用 `@PostConstruct` 注解的方法 或 `InitializingBean.afterPropertiesSet` 方法，允许在 Bean 初始化之后进行自定义操作。

**应用场景：**
- 初始化逻辑：在 Bean 完成属性注入后进行初始化操作，例如加载缓存、初始化连接池、设置默认值等。适用于需要在 Bean 完成依赖注入后执行额外的初始化操作的场景。
```java
@Component
public class CacheService implements InitializingBean {
    private Map<String, String> cache = new HashMap<>();

    @PostConstruct
    public void init() {
        System.out.println("Init method has been called");
    }

    @Override
    public void afterPropertiesSet() {
        // Load initial cache data
        System.out.println("Cache initialized");
    }
}
```
>6、如果使用了 @Bean(initMethod = "method")： 在 @Bean 注解中指定了 initMethod 属性，Spring 会在 Bean 初始化之后调用该方法。

**应用场景：**
- 适用于需要自定义初始化逻辑，并且希望在 Bean 实例化后、依赖注入完成后调用。
```java
@Configuration
public class AppConfig {
    @Bean(initMethod = "init")
    public MyBean myBean() {
        return new MyBean();
    }
}

```
>7、调用 `BeanPostProcessor.postProcessAfterInitialization：`在 Bean 初始化之后，Spring 会调用 `BeanPostProcessor.postProcessAfterInitialization` 方法，允许在 Bean 初始化之后进行自定义操作。

**应用场景：**
- 后处理和增强：在 Bean 完成初始化之后进行最后的修改、增强或代理。例如，注入监控、添加自定义的功能等。
```java
@Component
public class MonitoringBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof MonitoredService) {
            // Add monitoring or metrics collection
            return Proxy.newProxyInstance(bean.getClass().getClassLoader(),
                    bean.getClass().getInterfaces(),
                    (proxy, method, args) -> {
                        long start = System.currentTimeMillis();
                        Object result = method.invoke(bean, args);
                        long end = System.currentTimeMillis();
                        System.out.println("Method " + method.getName() + " took " + (end - start) + " ms");
                        return result;
                    });
        }
        return bean;
    }
}
```
>8、Bean 使用 ：在 Bean 使用过程中，Spring 会根据 Bean 的生命周期顺序依次执行下述步骤。
- 调用构造函数：在 Bean 实例化之前，Spring 会调用构造函数，创建一个新的 Bean 实例。
- 调用 @Autowired 注解的构造函数：如果 Bean 有 @Autowired 注解的构造函数，Spring 会调用该构造函数，将依赖注入到 Bean 中。

**应用场景：**
- 正常业务操作：在 Bean 完成初始化后，可以用于正常业务操作，例如服务调用、数据处理等。此时 Bean 已完全准备好，可以安全地进行业务逻辑操作。
```java
@Service
public class UserService {
    public void performBusinessOperation() {
        // Execute business logic
    }
}
```
>9、Bean 销毁：调用 `DisposableBean.destroy` 方法：在 Bean 使用结束后，Spring 会调用 `DisposableBean.destroy` 方法，允许在 Bean 销毁之前进行自定义操作。

**应用场景：**
- 资源清理：在应用程序关闭时，进行资源释放和清理操作，如关闭数据库连接、释放文件句柄等。适用于需要在 Bean 销毁时执行清理操作的场景。
```java
@Bean(destroyMethod = "cleanup")
public MyBean myBean() {
    return new MyBean();
}
```
>二、CommandLineRunner 、ApplicationRunner 是 Spring Boot 提供的一个接口，用于在应用程序启动后执行一段特定的代码。 它常用于执行一些启动时需要进行的初始化工作，例如加载数据、检查系统状态、配置应用程序参数等

**应用场景：**
- 数据初始化：在应用启动时加载一些初始数据到数据库中。

  ps: 假设有一个用户实体 User 和对应的 UserRepository，在应用启动时，我们希望预先插入一些用户数据。
  ps: Spring Boot 应用程序启动后，会打印 "Hello, World!"。
- 检查系统状态：在应用启动时检查某些系统状态或依赖服务的可用性。
- 配置初始化：根据环境或启动参数初始化应用程序的配置。
- 执行一次性任务：在应用启动时执行一些一次性的任务，例如清理旧数据、迁移数据等。
>三、如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！

