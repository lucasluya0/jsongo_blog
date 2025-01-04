---
date: 2024-03-15
title: Spring Event驱动-优雅代码的助推器
keywords: Spring Event, 事件驱动, 观察者模式, 事件发布, 事件监听, 系统解耦, ApplicationEvent
tags: [Spring事件, 事件驱动, 观察者模式, Spring框架, 系统解耦]
summary: 详细介绍了Spring事件发布机制的实现和应用，包括事件定义、发布和监听的完整流程，以及在实际项目中的最佳实践。
author: 柳树下的程序员
---
SpringEvent驱动-优雅代码的助推器

事件发布机制是一种基于观察者模式的设计方案。在这种模式中，事件源（Event Source）会生成事件，而事件监听器（Event Listener）则对这些事件进行处理。通过这种方式，我们可以在不同的组件之间实现松耦合的通信。

>一、 Spring事件发布的核心组件

- 事件（Event）： 一个继承自ApplicationEvent的类，用于表示具体的事件。
- 事件发布器（Event Publisher）： 一般指ApplicationEventPublisher接口（或者用ApplicationContext进行发布），用于发布事件。
- 事件监听器（Event Listener）： 实现ApplicationListener接口，用于监听和处理特定类型的事件。

>二、Spring 的事件发布订阅是默认是同步调用，可以通过在事件监听器上添加@Async注解形成异步

>三、事件发布机制的使用场景

- 各种通知系统（如邮件通知、短信通知、推送通知）
- 给三方系统发送信息

>四、实现方式

**- 定义事件类**
```java
import org.springframework.context.ApplicationEvent;

public class CustomEvent extends ApplicationEvent {
    private String message;

    public CustomEvent(Object source, String message) {
        super(source);
        this.message = message;
    }

    public String getMessage() {
        return message;
    }
}

```
**- 定义事件监听器**
```java
@Component
public class EventHandler {
    @EventListener
    public void customerEvent(CustomEvent event) {
        System.out.println("Received event - " + event.getMessage());
    }
}

```
**- 发布事件**
```java
@Component
public class EventPublisher {
    @Autowired
    private ApplicationEventPublisher applicationEventPublisher;

    public void publishEvent(String message) {
        CustomEvent event = new CustomEvent(this, message);
        applicationEventPublisher.publishEvent(event);
    }
}
```
>五、如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！