---
date: 2024-12-30
tags: 
summary: 介绍如何使用Arthas工具进行Java类文件的热部署，包括将arthas-boot.jar部署到服务器、上传修改后的class文件、启动Arthas并执行retransform命令等完整操作步骤
coverImage: https://mmbiz.qpic.cn/sz_mmbiz_png/aAia1M9Vb9bicypPFaFMqhicicUAtWvRw62nKcgRjgQGicibK3Cq7cnTsqNdFvWDFm0YrdAjbS4O0Gibr0ILPFtzf6ssQ/640?wx_fmt=png&amp;from=appmsg
author: 柳树下的程序员
---
项目应急箱-Arthas部署类文件

`推荐使用：`
改动不大的代码，可以直接在本地改完后得到`class`文件，然后使用`Arthas`热部署到应用中。

>第一步：将arthas-boot.jar 复制到 同一个应用的服务器中，我这里是用的docker 启动的springboot 应用，所以我直接将arthas-boot.jar 拷贝到 docker 容器中

![img.png](img.png)
```shell
复制arthas到容器内根目录下
docker cp arthas-boot.jar 容器id:/arthas-boot.jar

进入容器内部
docker exec -it 容器id bash;
```

>第二步：将需要修改的class文件 上传至arthas 能访问的文件夹下，然后使用命令热部署这个class文件
1、增加打印语句
2、上传至容器内部
![img_2.png](img_2.png)
![img_3.png](img_3.png)

>第三步：启动arthas,然后可以使用命令 `help` 查看所有命令
```shell
java -jar arthas-boot.jar
```

![img_1.png](img_1.png)


>第四步: 选择修改的应用，arthas 会自动进入应用 ，然后执行命令
```shell
retransform + 文件路径
```
出现success则说明成功 ,操作完成后关闭arthas

![img_4.png](img_4.png)

> 如果觉得对你有点帮助，请点点最下面的关注、点赞、转发、在看。谢谢支持！公众号：`ddtongxiang`在线等你！