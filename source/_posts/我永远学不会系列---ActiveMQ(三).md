---
layout: posts
title: 我永远学不会系列---ActiveMQ(三)
date: 2018-08-18 00:38:36
categories: MQ
tags: [ActiveMQ]
top: false
---
主要是关于ActiveMQ集群，这里采用的方式是：Zookeeper+LevelDB+ActiveMQ。还有一些项目实际使用MQ遇到的一些问题。实际上再生产环境中不可能采用一个单机的ActiveMQ进行消息传输的，因为ActiveMQ不像是Kafka，天然就是分布式的，ActiveMQ的吞吐量和Kafka可不是一个数量级的，对于大的分布式系统来说，其实我认为采用ActiveMQ作为消息中间件是不成的，QPS一旦高了，就可能系统就瘫了。其实对于保险公司来说，并不存在说某一个时间段并发特别高，也就是每个月月末分公司冲冲业绩了，系统并发量会高点。用这个ActiveMQ集群足够了。

<!--more--> 

#### 一、利用Zookeeper实现ActiveMQ的高可用

ActiveMQ官方提供的架构图：

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0mweAD3cqAAAeTA7iJyg747.png)

> Master/Slave broker的信息要注册到ZK。
>
> 注意到只有Master对外提供了服务，Slave是待机状态。当Master出现故障，ZK内部的选举机制，会让一个Slave升级成Master对外提供服务。

既然要做到高可用，那么ZK也得是高可用的，所以这里的搭建方案是这样的：

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0m0iAayUDAAAmqIE04pc779.png)

##### **1：JDK环境**

保证这3台机器都安装了JDK，并配置了JAVA环境变量。

##### **2：配置Zookeeper**

![img](http://www.ao10001.wang/group1/M00/00/00/rB9p_Fx0m3-AF-fXAAAaPLps8kc907.png)

> 为什么要配置ZK环境变量呢？很简单，在命令行下直接使用ZK相关的命令，而不是进入到安装ZK目录下的bin，更不想用绝对路径。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx0m62AWavqAACIeDOk6QA838.png)

> 注意dataDir目录的指定；注意2181是外部访问ZK的端口；
>
> 2888:3888是ZK集群内部通信（比如ZK原子广播消息）的端口，注意`server.X`的定义，这是将ZK集群中的实例进行编号，实际上需要在dataDir目录中新建myid文件，并与之保持一致。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx0m9-AE06YAAAKirHuAMo666.png)

##### **3：启动ZK**

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx0nA-ASV3cAAB42oHrTgM273.png)

可以通过netstat命令查看2722进程，发现ZK的端口是2181，这和zoo.cfg的配置是一致的。让3台机器的ZK都启动起来。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx0nDyAMD8dAAAzgMbScpI816.png)

##### **4：ActiveMQ主从配置**

注意了，由于我将在3台物理机上搭建一台Master，2台Slave，因此我这边不需要对端口配置文件进行改动。比如WEB管控台的jetty.xml。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx0nGqAAMVXAAAVujVjAHE794.png)

> 3台机器应该对外只有一个统一的名称，就是这个brokerName。3台机器都修改成一个名称即可。

这里持久化，我将采用LevelDB，因此需要修改持久化配置：

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx0nJaAFDlvAAArXuTDeg0164.png)

注意bind地址，其实是ActiveMQ集群内部通信的TCP端口，和ActiveMQ对外提供的消息端口（默认61616）不要搞混了。

hostname即本机的主机名称。

给出ZK集群的列表以及zkPath。zkPath下面其实存放着ActiveMQ的节点，在后续你会看到。

启动3台机器上的ActiveMQ，然后利用ZooInspector你可以看到：

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx0nMOAB1WIAAAwYGFC3Yo645.png)

此时此刻，基于ZK的ActiveMQ的高可用方案就做好了。那么JAVA端访问ActiveMQ有什么变化么？其实就是在创建ConnectionFactory的时候给定的URL有变化：

> **failover:(tcp://192.168.99.121:61616,tcp://192.168.99.122:61616,tcp://192.168.99.123:61616)?Randomize=false**

就是一个失败转移协议！

上面只是做了一个ActiveMQ的高可用方案，那么ActiveMQ集群呢？其实所谓的ActiveMQ集群就是多个ActiveMQ高可用之间产生关联：

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx0nO6ARl25AAAZMNaUkHc464.png)

> 高可用的ActiveMQ-1,ActiveMQ-2,…,ActiveMQ-N就可以组建ActiveMQ集群
>
> 在配置上很简单，其实就是ActiveMQ-1要知道ActiveMQ-2的信息而已.

#### 二、遇到的一些问题

##### 1：重复消费导致数据库中出现脏数据

这里边有个概念就是幂等性，什么意思呢？就是一个消息，或者一个请求，给你重复来多次，你得保证对应的数据是不会改变的，不能出错。

在MQ中，常用的就是在消费了一条消息之后，可以把和消费过的消息放在一个地方，比如Redis、内存Set都可以，下次再进行消费的时候先去判断这条消息是否已经被消费过了。也可以在数据库中设置唯一键，也可以保证这种重复消费的情况，避免脏数据。

##### 2：持久化消息非常慢

默认的情况下，非持久化的消息是异步发送的，持久化的消息是同步发送的，遇到慢一点的硬盘，发送消息的速度是无法忍受的。但是在开启事务的情况下，消息都是异步发送的，效率会有2个数量级的提升。所以在发送持久化消息时，请务必开启事务模式。其实发送非持久化消息时也建议开启事务，因为根本不会影响性能。

##### 3：消息的不均匀消费

有时在发送一些消息之后，开启2个消费者去处理消息。会发现一个消费者处理了所有的消息，另一个消费者根本没收到消息。原因在于ActiveMQ的prefetch机制。当消费者去获取消息时，不会一条一条去获取，而是一次性获取一批，默认是1000条。这些预获取的消息，在还没确认消费之前，在管理控制台还是可以看见这些消息的，但是不会再分配给其他消费者，此时这些消息的状态应该算作“已分配未消费”，如果消息最后被消费，则会在服务器端被删除，如果消费者崩溃，则这些消息会被重新分配给新的消费者。但是如果消费者既不消费确认，又不崩溃，那这些消息就永远躺在消费者的缓存区里无法处理。更通常的情况是，消费这些消息非常耗时，你开了10个消费者去处理，结果发现只有一台机器吭哧吭哧处理，另外9台啥事不干。

解决方案：**将prefetch设为1**，每次处理1条消息，处理完再去取，这样也慢不了多少。

```
ActiveMQPrefetchPolicy p = new ActiveMQPrefetchPolicy();
p.setQueuePrefetch(1);// 设置prefetch 值(多个消费者有用)
// 实例化连接工厂
connectionFactory.setPrefetchPolicy(p);
```

##### 4：**死信队列**

消费消息有2种方法，一种是调用consumer.receive()方法，该方法将阻塞直到获得并返回一条消息。这种情况下，消息返回给方法调用者之后就自动被确认了。另一种方法是采用listener回调函数，在有消息到达时，会调用listener接口的onMessage方法。在这种情况下，在onMessage方法执行完毕后，消息才会被确认，此时只要在方法中抛出异常，该消息就不会被确认。那么问题来了，如果一条消息不能被处理，会被退回服务器重新分配，如果只有一个消费者，该消息又会重新被获取，重新抛异常。就算有多个消费者，往往在一个服务器上不能处理的消息，在另外的服务器上依然不能被处理。难道就这么退回–获取–报错死循环了吗？

在重试6次后，ActiveMQ认为这条消息是“有毒”的，将会把消息丢到死信队列里。如果消息不见了，去ActiveMQ.DLQ里找找，说不定就躺在那里。

好了，这个ActiveMQ写到这里了，我感觉这个MQ后期很有可能被废了，大数据时代了，Kafaka、RocketMQ还是记得去研究的呀。