---
layout: posts
title: 我永远学不会系列---ActiveMQ(二)
date: 2018-08-17 00:38:36
categories: MQ
tags: [ActiveMQ]
top: false
---
这篇文章主要讨论的话题是：消息的顺序消费、JMS Selectors、消息的同步/异步接受方式、Message、P2P/PubSub、持久化订阅、持久化消息到MySQL以及与SpringBoot整合等知识。其实在使用的时候了解一些原理是很重要的，知道它是干嘛的，实现流程是怎么样的，然后在编程过程中就不会留下一些大坑。哪种情况下该使用什么方式去实现，我在工作中或者自己写项目时候常用的就是SpringBoot，使用Spring过程是痛苦的，一堆配置文件巴拉巴拉，而SpringBoot对ActiveMQ的自动配置其实也省了很多麻烦。

<!--more--> 

#### 一、消息的顺序消费

我们已经明确知道了**ActiveMQ并不能保证消费的顺序性**，即便我们使用了消息优先级。而在实际开发中，有些场景又是需要对消息进行顺序消费的，比如：用户从下单、到支付、再到发货等。如果使用ActiveMQ该如何保证消费的顺序性呢？

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0jYyASgSjAAA0BiwQHxk836.png)

首先来说，在实际中，并不需要的是对全部消息的全局有序消费，仅仅需要的是局部业务有序性消费。比如说，需要的是一个用户的下订单、支付、发货这个过程的3条消息有序消费。

比如，可以根据用户ID简单做一个HASH，将消息定位到不同的队列上，也就意味着同一个用户的消息将发往同一个队列。这样做的好处在于，多个队列之间可以并行处理。

然后，在队列上可以对一段时间上的消息按照用户分组进行排序，这只是一个少量消息的局部排序而已，比如Queue-A上有一个用户的3条消息（订单消息msg1、支付消息msg2、发货消息msg3），那么，msg1将交给订单业务系统，处理完成后，msg2交给支付系统，处理完成后，msg3交给发货系统。**虽然这个处理过程是同步的（一条消息处理完，在接着处理），但是它的并发性，系统的处理能力并没有下降！为什么这么说呢？**

假设，msg1/msg2/msg3处理各需要0.1S，如果订单业务系统、支付系统、发货系统并没有分开，而是一个“大系统”，那么显然订单业务在0.1S完成后，需要等待后面的支付、发货逻辑处理完才能继续工作，意味着订单业务干了0.1S的活，等了0.2S，导致在0.3秒内订单业务只处理了1条消息。而现在这3个系统是分开的，那么在0.3S内，订单业务系统可以处理3条消息，而且没有业务系统闲着！

#### 二、有关消息选择器

JMS Selectors，即消息选择器。消息对象有消息属性，用于消息选择器。来看一个代码片段，你就会明白：

生产者片段：

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0kB-AREzDAABeZASnGek039.png)

消费者片段：

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0kEaAIL8kAABPDCaMafc772.png)

需要注意：

> 第一，生产者端需要设置消息属性，一定要注意的是setXxxProperty(filed,value)
>
> 第二，给出条件，其实本质上就是SQL92语法
>
> 第三，创建消费者的时候，指定条件即可

#### 三、消息的同步 AND 异步 接受

消息的接受，可以通过消费者的`receive()`/`receive(long time)`/`receiveNoWait()`，这几种方式是client端主动接受消息，可以理解为消息的同步接受。要知道这种同步的消息接受方式，是让我们很难受的，我们不得不写一个死循环来不断接受消息。那么有没有一种比较优雅的方式，比如我们设置一个类似消息监听的机制，一旦队列上有消息了，那么回调我们的message handler进行处理呢？

![img](https://upload-images.jianshu.io/upload_images/4943997-3509c1aee5e74506.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/633/format/webp)

消息的异步接受是指当消息到达时，ActiveMQ主动通知客户端。可以通过注册一个实现了MessageListener接口的对象到MessageConsumer。MessageListener只有一个必须要实现的方法，即onMessage。在发往Destination的消息时，会调用该方法。

> 这种异步接受“貌似”是ActiveMQ主动的推送消息给消费者，其本质还是消费者轮询消息服务器导致的，只不过这个过程被封装了！

#### 四、有关Message

JMS程序的核心在于，生产和消费的消息能够被其他程序所使用到。JMS Message是一个既简单又不乏灵活的基本格式，由`消息头`、`属性`、`消息体`3部分组成。

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_FxuSUOAE3ilAAB94MXbxXU415.png)

> 接受到消息后，一般需要通过instanceof来判断类型后在进行处理！

#### 五、P2P or Pub/Sub

P2P：![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0kYaAM0QUAAAa6GGuIYs334.png)

> 生产者端发送一条消息，消费者端只会有一个消费者消费这个消息。好像打电话，一对一通信！

Pub/Sub:![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0kbuANQ_uAABMnOcuTCg770.png)

> 一对多通信，发送一条消息，所有订阅了该目标的消费者都会收到消息。
>
> P2P、Pub/Sub在代码上的区别点仅仅在于，目标类型的创建是`createQueue` or `createTopic`，其他一切照旧！
>
> 对于订阅模式，对订阅者提出了特殊的要求，要想收到消息，必须先订阅，而且订阅进程必须一直处于运行状态！实际上，有时候消费者重启了下，那么这个消费者将丢失掉一些消息，那么能否避免这样的情况呢？ActiveMQ已经替我们想好了，就是持久化订阅！

#### 六、持久化订阅

所谓持久化订阅，打个比方，就是说跟MQ打声招呼，即便我不在，那么给我发送的消息暂存在MQ，等我来了，再给我发过来。说白了，持久化订阅，需要给MQ备个案（你是谁，想在哪个Topic上搞特殊化）！看一个代码片段：

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0kgmAUIASAACO2z_93Cw264.png)

> 每一个持久化订阅者都应该有一个唯一的ID作为标示以及要在哪个Topic上进行持久化订阅，一旦这些信息告知MQ之后，那么以后不论持久化订阅者在不在线，那么他的消息会暂存在MQ，以后都会发给他！

#### 七、持久化消息到MySQL

在前文中已经提及默认情况下，ActiveMQ是开启持久化消息机制的，并且是持久化到Kahadb的，但是”很可惜”kahadb对我们不是很友好的可视化，其实ActiveMQ提供了配置的方式让我们来选择持久化消息到哪里，这里我以到MySQL为例来说明。

在activemq.xml的节点中增加MySQL信息

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0kmWASrZFAAA9_BUKgJ4766.png)

> 注意到这个bean的id，这个是要被引用的。

注释kahadb，启用持久化到MySQL配置

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0kpmALjQgAAAdUDnGAu0945.png)

> 实际中，我们会持久化到哪里呢？一般情况下，比如到kahadb，比如到leveldb，因为这些数据库的性能要较MySQL更高些，我们并不关心消息的“可视化”，更加关心的是消息在持久化的同时更加高效！

#### 八、整合SpringBoot

https://blog.csdn.net/liuyuanq123/article/details/79109218 这个博客里边有一个基本的Demo，在单机环境下自己测试玩玩挺有用的，记录下。

整合工程：

https://gitee.com/ao10001/mq

一些基本的：

添加依赖：

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
<!--启用连接池-->
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-pool</artifactId>
</dependency>
```

简单配置连接属性：

```
spring:
  activemq:
    user: admin
    password: admin
    broker-url: tcp://localhost:61616 # 提供服务的61616接口，不是8161。默认为61616
    pool:
      enabled: true
      max-connections: 10
```