---
layout: posts
title: 我永远学不会系列---ActiveMQ(一)
date: 2018-08-16 00:38:36
categories: MQ
tags: [ActiveMQ]
top: false
---
这是关于消息中间件ActiveMQ的一个系列专题文章，将涵盖JMS、ActiveMQ的初步入门及API详细使用、两种经典的消息模式（PTP and Pub/Sub）、与SpringBoot整合、ActiveMQ集群、监控与配置优化等。话不多说，我们来一起瞧一瞧！我们生产环境就是用的这个MQ，最近好像这个MQ社区也不是很活跃了，在过几年也许这个MQ就淘汰也不定了呢，比如现在比较常用的Kafka、RocketMQ都是很强的消息中间件。以后有时间好好学学Kafka，没准我就去干大数据去了呢？？？

<!--more--> 

#### 一、有关JMS

首先来说较早以前，也就是没有JMS的那个时候，很多应用系统存在一些缺陷：

- 通信的同步性，client端发起调用后，必须等待server处理完成并返回结果后才能继续执行
- client 和 server 的生命周期耦合太高，client进程和server服务进程都必须可用，如果server出现问题或者网络故障，那么client端会收到异常
- 点对点通信，client端的一次调用只能发送给某一个单独的服务对象，无法一对多

JMS，即Java Message Service，通过面向消息中间件（MOM：Message Oriented Middleware）的方式很好的解决了上面的问题。大致的过程是这样的：发送者把消息发送给消息服务器，消息服务器将消息存放在若干队列/主题中，在合适的时候，消息服务器会将消息转发给接受者。在这个过程中，发送和接受是异步的，也就是发送无需等待，而且发送者和接受者的生命周期也没有必然关系；在pub/sub模式下，也可以完成一对多的通信，即让一个消息有多个接受者。

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0AG2AcqeMAAAYbhJIuss258.png)

JMS只给出接口，然后由具体的中间件去实现，比如ActiveMQ就是实现了JMS的一种Provider，还有阿里巴巴的RocketMQ。这些消息中间件都符合JMS规范。

`Provider/MessageProvider`：生产者

`Consumer/MessageConsumer`：消费者

`PTP`：Point To Point，点对点通信消息模型

`Pub/Sub`：Publish/Subscribe，发布订阅消息模型

`Queue`：队列，目标类型之一，和PTP结合

`Topic`：主题，目标类型之一，和Pub/Sub结合

`ConnectionFactory`：连接工厂，JMS用它创建连接

`Connnection`：JMS Client到JMS Provider的连接

`Destination`：消息目的地，由Session创建

`Session`：会话，由Connection创建，实质上就是发送、接受消息的一个线程，因此生产者、消费者都是Session创建的

#### 二、有关ActiveMQ

ActiveMQ是Apache出品的，非常流行的消息中间件，可以说要掌握消息中间件，需要从ActiveMQ开始，要掌握更加强大的RocketMQ，也需要ActiveMQ的基础。

一个windows版本的文件目录：

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_FxuEASAc-jEAACDZEqutrs138.png)

`bin` 下面存放的是ActiveMQ的启动脚本activemq.bat，注意分32、64位

`conf`里面是配置文件，重点关注的是activemq.xml、jetty.xml、jetty-realm.properties。在登录ActiveMQ Web控制台需要用户名、密码信息；在JMS CLIENT和ActiveMQ进行何种协议的连接、端口是什么等这些信息都在上面的配置文件中可以体现。

`data`目录下是ActiveMQ进行消息持久化存放的地方，默认采用的是kahadb，当然我们可以采用leveldb，或者采用JDBC存储到MySQL，或者干脆不使用持久化机制。

`webapps` ActiveMQ自带Jetty提供Web管控台

`lib` ActiveMQ为提供了分功能的JAR包

在JDK安装没有问题的情况下，直接activemq.bat启动它，并访问Web控制台！

> 访问ActiveMQ web控制台的用户名、密码在哪里配置的？URL当中的端口是在哪里配置的？

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0ANSAMgC-AAAK8Ag9Y4o558.png)

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0AQGAWHygAAAy1kywhNU974.png)

#### 三、HelloWorld Demo

来一个HelloWorld级别的例子，来感受下ActiveMQ。写一个生产者用于发送消息，一个消费者用于接收消息。实际上，JMS是有“套路”的。

##### 1：创建ConnectionFactory连接工厂

```
 //1、创建工厂连接对象，需要制定ip和端口号
ConnectionFactory connectionFactory = new ActiveMQConnectionFactory(
                ActiveMQConnectionFactory.DEFAULT_USER,
                ActiveMQConnectionFactory.DEFAULT_PASSWORD,
                ActiveMQConnectionFactory.DEFAULT_BROKER_BIND_URL);
```

> 实际上，这里是存在安全隐患的，也就是任何人一旦知道MQ的地址，就可以连接访问了，可以在activemq.xml中配置指定的用户、密码才能访问ActiveMQ。
>
> 关于broker_bind_url，默认就是tcp://localhost:61616，说明是采用TCP协议，61616端口。其实对于ActiveMQ不仅仅支持TCP协议，还有其他协议，开启了多个端口。

##### 2：创建Connection

```
//2、使用连接工厂创建一个连接对象
Connection connection = connectionFactory.createConnection();
//3、开启连接
connection.start();
```

> Connection就代表了应用程序和消息服务器之间的通信链路。获得了连接工厂后，就可以创建Connection。
>
> 事实上，ConnectionFactory存在重载方法：
>
> Connection createConnection(String username,String password)
>
> 也就是说我们也可以在这里指定用户名、密码进行验证。

##### 3：创建Session

```
//4、使用连接对象创建会话（session）对象
//前两步其实就是为了创建Session(上下文环境对象)，创建Session有一些配置参数，比如是否启用事务 签收模式
Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
```

> Session，用于发送和接受消息，而且是单线程的，支持事务的。如果Session开启事务支持，那么Session将保存一组信息，要么commit到MQ，要么回滚这些消息。Session可以创建MessageProducer/MessageConsumer。

##### 4：创建Destination

```
//5、使用会话对象创建目标对象，包含queue和topic（一对一和一对多）
Queue queue = session.createQueue("test-queue");
```

> 所谓消息目标，就是消息发送和接受的地点，要么queue，要么topic。

##### 5：创建MessageProducer

```
//6、使用会话对象创建生产者对象
MessageProducer producer = session.createProducer(queue);
```

##### 6：设置持久化方式

```
//设置持久化/非持久化特性，如果非持久，那么意味着MQ的重启会导致消息丢失。
//如果持久化到Kahdb/Leveldb/jdbc方式的话，意味着消息持久化
producer.setDeliveryDelay(DeliveryMode.NON_PERSISTENT);
```

##### 7：定义消息对象，并发送

```
//7、使用会话对象创建一个消息对象
TextMessage textMessage = session.createTextMessage("hello!test-queue");
//8、发送消息
producer.send(textMessage);
```

> 生产者和消费者之间传递的对象，由3个主要部分构成：
>
> 消息头（路由）+消息属性（消息选择器）+消息体（JMS规范的5种类型消息）
>
> `TextMessage`：字符串对象
>
> `MapMessage`： 一套名称-值 对
>
> `ObjectMessage`：一个序列化Java对象
>
> `BytesMessage`：一个未解释字节的数据流
>
> `StreamMessage`：Java原始值的数据流

##### 8：释放连接

```
//9、关闭资源
producer.close();
session.close();
connection.close();
```

> 必须close connection，只有这样ActiveMQ才会释放资源！
>
> 消费者的代码和上面非常类似，只不过就是创建MessageConsumer进行receive而已，注意receive()/receive(long)/receiveNoWait()，这些说明消费者可以采用阻塞模式、非阻塞模式接受消息。

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0igeAFbqmAABDnGqyVww604.png)

> Messages Enqueued：表示生产了多少条消息，记做P
>
> Messages Dequeued：表示消费了多少条消息，记做C
>
> Number Of Consumers：表示在该队列上还有多少消费者在等待接受消息
>
> Number Of Pending Messages：表示还有多少条消息没有被消费，实际上是表示消息的积压程度，就是P-C

#### 四、有关Session

在通过Connection创建Session的时候，需要设置2个参数，一个是否支持事务，另一个是签收的模式。有关这个签收模式：

什么是签收？通俗点说，就是消费者接受到消息后，需要告诉消息服务器，我收到消息了。当消息服务器收到回执后，本条消息将失效。因此签收将对PTP模式产生很大影响。如果消费者收到消息后，并不签收，那么本条消息继续有效，很可能会被其他消费者消费掉！

`AUTO_ACKNOWLEDGE`：表示在消费者receive消息的时候自动的签收

`CLIENT_ACKNOWLEDGE`：表示消费者receive消息后必须手动的调用acknowledge()方法进行签收

`DUPS_OK_ACKNOWLEDGE`：签不签收无所谓了，只要消费者能够容忍重复的消息接受，当然这样会降低Session的开销

在实际中，我们应该采用哪种签收模式呢？**CLIENT_ACKNOWLEDGE**，采用手动的方式较自动的方式可能更好些，因为接收到了消息，并不意味着成功的处理了消息，假设我们采用手动签收的方式，只有在消息成功处理的前提下才进行签收，那么只要消息处理失败，那么消息还有效，仍然会继续消费，直至成功处理！

#### 五、关于消息的Priority/TTL/DeliveryMode

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0i0-AEo-nAABNr7xVszU143.png)

> 在上面的code当中，创建生产者的时候，指定了Destination，设置了持久化方式，实际上这些都可以不必指定的，而是到send的时候指定。而且在实际业务开发中，往往根据各种判断，来决定将这条消息发往哪个Queue，因此往往不会在MessageProducer创建的时候指定Destination。
>
> `TTL`，消息的存活时间，一句话：生产者生产了消息，如果消费者不来消费，那么这条消息保持多久的有效期.
>
> `priority` 消息优先级，**0-9**。0-4是普通消息，5-9是加急消息，消息默认级别是 **4**。注意，消息优先级只是一个理论上的概念，并不能绝对保证优先级高的消息一定被消费者优先消费！也就是说ActiveMQ并不能保证消费的顺序性！
>
> `deliveryMode` 如果不指定，默认是持久化的消息。如果可以容忍消息的丢失，那么采用非持久化的方式，将会改善性能、减少存储的开销。