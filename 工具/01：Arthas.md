---
date: 2024-03-15
title: Arthas - Java应用诊断利器
keywords: [Arthas, Java诊断工具, 热部署, 线程分析, 性能监控, 线上问题排查, 阿里巴巴开源, JVM工具]
tags: [Arthas, Java诊断, 性能监控, 热部署, 线上排查]
summary: 介绍阿里巴巴开源的Java诊断工具Arthas的核心功能，包括热部署、线程分析、性能监控等12个常用命令的详细使用方法，帮助开发者高效排查线上问题。
author: 柳树下的程序员
---
项目应急箱-Arthas
> 在日常开发中，面对复杂的线上环境，线上问题的排查经常让开发者头疼不已。而 Arthas，作为阿里巴巴开源的一款强大的 Java
> 诊断工具，正是为了帮助开发者轻松定位问题、实时诊断线上应用所设计的利器。

`推荐使用:`
动态修改代码：通过热部署快速修复线上问题，避免繁琐的重启流程。

> 为什么需要热部署？

在线上环境中，如果排查某些线上 bug 时发现问题来源于一小部分代码，那么最理想的情况就是修改后立即生效，而不是通过重启或重新发布整个应用。这不仅可以避免业务中断，还能大大节省时间成本。这时，Arthas
的 热部署功能 就派上了用场。我常用来热部署测试环境

> `Arthas` 热部署的注意事项

1. 热部署并不是解决所有问题的万能药，必须小心使用，避免对线上系统的稳定性产生影响。
2. `Arthas` 热部署针对 class 文件 有一定的局限性，并不能完全取代发版流程。例如，一些复杂的代码逻辑修改、方法签名的更改等情况，热部署可能无法生效。详细的失效场景可以参考官方文档。
3. `Arthas` 热部署特别适合用于线上环境快速修复那些 代码改动小、影响范围有限 的问题。 例如：
    - 某个线上接口参数校验有问题，可以通过修改校验逻辑并热部署，立即生效。
    - 页面显示存在细微错误，可以在热部署后验证效果。

> Arthas 是一个非常强大的 Java 诊断工具，它的命令行功能覆盖了多种线上问题排查的场景。以下是一些 常见且实用的 Arthas
> 命令，帮助你快速上手并高效解决线上问题。

1. `dashboard`

   查看当前系统的整体健康状况，包括 CPU 使用率、内存、GC 状况、线程运行情况等。

   应用场景：当你需要查看应用整体健康状况或排查系统性能瓶颈时使用。
2. `thread`
   查看当前 Java 进程中的线程信息，支持查找死锁、查看 CPU 占用最高的线程。
   应用场景：用于排查线程数过多、线程堵塞、死锁等问题。

```
   thread -n 3 # 查看 CPU 占用最高的 3 个线程
   thread -b # 查找死锁
```

3. `jad`
   反编译类文件，查看 Java 类的源码。

```
   jad <class-name>
   应用场景：想要动态调试某个类时，使用该命令来查看类的具体实现代码。
   jad com.example.MyClass # 查看 MyClass 类的源码
```

4. `watch`
   监控方法的执行情况，实时打印出方法调用的输入、输出和异常信息。

   ```
   watch <class-name> <method-name> {params | returnObj | throwExp} -x 2   
   应用场景：当你需要监控某个方法的输入输出或异常信息时使用，非常适合排查线上 bug。
   watch com.example.MyClass myMethod returnObj # 监控 myMethod 方法的返回值
   ```

5. `trace`
   跟踪方法调用过程，并打印每个方法的耗时情况。

```
   trace <class-name> <method-name>
   应用场景：用于查找性能瓶颈，了解方法调用的详细时间消耗。
   trace com.example.MyClass myMethod # 跟踪 myMethod 方法的调用过程
```

6. `tt (Time Tunnel)`
   录制方法的调用数据，允许回放方法调用过程，分析方法的历史调用数据。

    ```
    tt -t <class-name> <method-name>
    应用场景：用于调试多次调用的历史数据，类似“时光隧道”的功能，可以反复查看某次调用的细节。


    tt -t com.example.MyClass myMethod # 录制 myMethod 方法的调用数据
    tt -l # 列出所有录制的调用
    tt -r <index>  # 回放指定调用的细节
    ```

7. `redefine`
   热替换已加载的类，实现热部署功能，避免重启应用。

    ```
   redefine /path/to/your/Class.class
    应用场景：在修改代码后不重启应用的情况下，动态替换 class 文件。
    
    redefine /tmp/com/example/MyClass.class # 热部署 MyClass
   ```

8. `ognl`
   执行 OGNL 表达式，可以获取和修改 JVM 内部信息。

 ```
 ognl <expression>
    应用场景：当你想动态操作 Java 对象的属性、方法或执行复杂表达式时使用。
    
    ognl '@java.lang.System@out.println("Hello Arthas")'  # 执行打印
   ```

9. `heapdump`
   导出当前 JVM 的堆内存快照，用于分析内存泄漏等问题。

    ```
   heapdump <file-path>
    应用场景：当你怀疑应用存在内存泄漏或需要进行堆内存分析时使用。
    
    heapdump /tmp/heapdump.hprof # 导出堆快照
   ```

10. `getstatic`
    获取类的静态变量的值。

```
getstatic <class-name> <field-name>
应用场景：查看类的静态字段值，排查静态变量的状态问题。

    getstatic com.example.MyClass SOME_STATIC_FIELD # 查看某静态变量的值

```

11. `sysprop`
    查看或修改系统属性。

   ```
    sysprop
    应用场景：查看当前 JVM 的系统属性，或者需要动态调整某些属性值时使用。
    
    sysprop # 查看所有系统属性
    sysprop file.encoding UTF-8 # 修改某个系统属性
   ```

12. `classloader`
    查看并操作当前应用的 ClassLoader，支持查询类的加载信息。

```
classloader
应用场景：用于排查类加载问题，特别是复杂项目中多重 ClassLoader 的场景。

    classloader -t # 查看所有 ClassLoader
    classloader --urls <hashcode>  # 查看指定 ClassLoader 加载的路径
```

> 如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！