---
layout: posts
title: 我永远学不会系列--分布式服务治理之Dubbo
date: 2019-02-01 00:38:36
categories: Dubbo
tags: [Dubbo]
top: false
---
做过互联网项目的我想应该都听过Dubbo的大名吧，但是这一段时间好像SpringCloud的应用要比Dubbo还火一点，奈何我只会用Dubbo啊，SpringCloud有时间我学学，毕竟自己用SpringBoot也这么长时间了，这俩配合起来相比也更得心应手。不过现在使用Dubbo的公司也不在少数，毕竟开始重新维护了呀。好了，我就来写写有关Dubbo的那些事吧。

<!--more--> 

#### 一、分布式简介

随着互联网的发展，网站应用的规模不断扩大，常规的垂直应用架构已无法应对，分布式服务架构以及流动计算架构势在必行，亟需一个治理系统确保架构有条不紊的演进。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx7UMmANWytAACEiypWzHQ877.jpg)

- 单一应用架构
  当网站流量很小时，只需一个应用，将所有功能都部署在一起，以减少部署节点和成本。
  此时，用于简化增删改查工作量的数据访问框架(ORM) 是关键。
- 垂直应用架构
  当访问量逐渐增大，单一应用增加机器带来的加速度越来越小，将应用拆成互不相干的几个应用，以提升效率。
  此时，用于加速前端页面开发的 Web框架(MVC) 是关键。
- 分布式服务架构
  当垂直应用越来越多，应用之间交互不可避免，将核心业务抽取出来，作为独立的服务，逐渐形成稳定的服务中心，使前端应用能更快速的响应多变的市场需求。
  此时，用于提高业务复用及整合的 分布式服务框架(RPC) 是关键。
- 流动计算架构
  当服务越来越多，容量的评估，小服务资源的浪费等问题逐渐显现，此时需增加一个调度中心基于访问压力实时管理集群容量，提高集群利用率。
  此时，用于提高机器利用率的 资源调度和治理中心(SOA) 是关键。

#### 二、啥是RPC？

`RPC`【Remote Procedure Call】是指`远程过程调用`，是一种进程间通信方式，它是一种技术的思想，而不是规范。它允许程序调用另一个地址空间（通常是共享网络的另一台机器上）的过程或函数，而不用程序员显式编码这个远程调用的细节。即程序员无论是调用本地的还是远程的函数，本质上编写的调用代码基本相同。

RPC的一个基本原理图：

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx85ueAU7p-AAIA1egev-w305.png)

一个实例：

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx8562AClC4AAJlQ3SBO4A470.png)

系统A要调用系统B上的`hi`方法，流程就是这个样子喽。

> 衡量一个RPC框架就有两个方面，一个是能否`快速建立连接`，二是`序列化与反序列化机制`。

#### 三、Dubbo架构构成

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx7Zs6AIoJQAAEzK3XNhpU977.png)

##### 1、节点角色

`Provider`：暴露服务的服务提供方。

`Consumer`：调用远程服务的服务消费方。

`Registry`：服务注册与发现的注册中心。

`Monitor`：统计服务的调用次调和调用时间的监控中心。

`Container`： 服务运行容器。

##### 2、调用关系

1. 服务容器负责启动，加载，运行服务提供者。
2. 服务提供者在启动时，向注册中心注册自己提供的服务。
3. 服务消费者在启动时，向注册中心订阅自己所需的服务。
4. 注册中心返回服务提供者地址列表给消费者，如果有变更，注册中心将基于长连接推送变更数据给消费者。
5. 服务消费者，从提供者地址列表中，基于软负载均衡算法，选一台提供者进行调用，如果调用失败，再选另一台调用。
6. 服务消费者和提供者，在内存中累计调用次数和调用时间，定时每分钟发送一次统计数据到监控中心。

##### 3、dubbo的特性

(1) 连通性：

注册中心负责服务地址的注册与查找，相当于目录服务，服务提供者和消费者只在启动时与注册中心交互，注册中心不转发请求，压力较小。监控中心负责统计各服务调用次数，调用时间等，统计先在内存汇总后每分钟一次发送到监控中心服务器，并以报表展示服务提供者向注册中心注册其提供的服务，并汇报调用时间到监控中心，此时间不包含网络开销服务消费者向注册中心获取服务提供者地址列表，并根据负载算法直接调用提供者，同时汇报调用时间到监控中心，此时间包含网络开销注册中心，服务提供者，服务消费者三者之间均为长连接，监控中心除外注册中心通过长连接感知服务提供者的存在，服务提供者宕机，注册中心将立即推送事件通知消费者注册中心和监控中心全部宕机，不影响已运行的提供者和消费者，消费者在本地缓存了提供者列表，注册中心和监控中心都是可选的，服务消费者可以直连服务提供者。

(2) 健状性：

监控中心宕掉不影响使用，只是丢失部分采样数据数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务注册中心对等集群，任意一台宕掉后，将自动切换到另一台注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯服务提供者无状态，任意一台宕掉后，不影响使用。服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复。

(3) 伸缩性：

注册中心为对等集群，可动态增加机器部署实例，所有客户端将自动发现新的注册中心 。
服务提供者无状态，可动态增加机器部署实例，注册中心将推送新的服务提供者信息给消费者。

(4) 升级性：

当服务集群规模进一步扩大，带动IT治理结构进一步升级，需要实现动态部署，进行流动计算，现有分布式服务架构不会带来阻力：

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx7aJeAXJXHAAEFuJC6Bhk112.jpg)

#### 三、dubbo支持的注册中心

Dubbo提供的注册中心有如下几种类型可供选择：

- Multicast注册中心
- Zookeeper注册中心
- Redis注册中心
- Simple注册中心

ZooKeeper是一个开源的分布式服务框架，它是Apache Hadoop项目的一个子项目，主要用来解决分布式应用场景中存在的一些问题，如：**统一命名服务、状态同步服务、集群管理、分布式应用配置管理等**，它支持Standalone模式和分布式模式，在分布式模式下，能够为分布式应用提供高性能和可靠地协调服务，而且使用ZooKeeper可以大大简化分布式协调服务的实现，为开发分布式应用极大地降低了成本。

ZooKeeper总体架构：

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx7aWOABuAAAAAnKgYVaE8192.png)

ZooKeeper集群由一组Server节点组成，这一组Server节点中存在一个角色为Leader的节点，其他节点都为Follower。当客户端Client连接到ZooKeeper集群，并且执行写请求时，这些请求会被发送到Leader节点上，然后Leader节点上数据变更会同步到集群中其他的Follower节点。

#### 四、SpringBoot整合Dubbo

一个整合实例

#### 五、有关Dubbo的相关配置

XML配置官方文档

schema配置官方文档

#### 六、Dubbo的高可用性

##### 1、zookeeper宕机与dubbo直连

zookeeper注册中心宕机，还可以消费dubbo暴露的服务。

> 监控中心宕掉不影响使用，只是丢失部分采样数据.
>
> 数据库宕掉后，注册中心仍能通过缓存提供服务列表查询，但不能注册新服务.
>
> 注册中心对等集群，任意一台宕掉后，将自动切换到另一台
>
> **注册中心全部宕掉后，服务提供者和服务消费者仍能通过本地缓存通讯**
>
> 服务提供者无状态，任意一台宕掉后，不影响使用
>
> 服务提供者全部宕掉后，服务消费者应用将无法使用，并无限次重连等待服务提供者恢复

##### 2、集群下Dubbo的负载均衡配置

官方文档

在集群负载均衡时，Dubbo 提供了多种均衡策略，缺省为 `random` 随机调用。

**负载均衡策略：**

**Random LoadBalance**

随机，按权重设置随机概率。

在一个截面上碰撞的概率高，但调用量越大分布越均匀，而且按概率使用权重后也比较均匀，有利于动态调整提供者权重。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx-KOyAEOjGAACOAJyxgRQ106.png)

**RoundRobin LoadBalance**

轮循，按公约后的权重设置轮循比率。

存在慢的提供者累积请求的问题，比如：第二台机器很慢，但没挂，当请求调到第二台时就卡在那，久而久之，所有请求都卡在调到第二台上。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx-KciAKFpaAABek33yfzI715.png)

**LeastActive LoadBalance**

最少活跃调用数，相同活跃数的随机，活跃数指调用前后计数差。

使慢的提供者收到更少请求，因为越慢的提供者的调用前后计数差会越大。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx-KjKATVzuAAA665HsbRU466.png)

**ConsistentHash LoadBalance**

一致性 Hash，相同参数的请求总是发到同一提供者。

当某一台提供者挂时，原本发往该提供者的请求，基于虚拟节点，平摊到其它提供者，不会引起剧烈变动。算法参见：http://en.wikipedia.org/wiki/Consistent_hashing

缺省只对第一个参数 Hash，如果要修改，请配置

```
<dubbo:parameter key="hash.arguments" value="0,1" />
```

缺省用160 份虚拟节点，如果要修改，请配置

```
<dubbo:parameter key="hash.nodes" value="320" />
```

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx-KluAaW_HAAC9eJTsnJI353.png)

##### 3、服务降级

当服务器压力剧增的情况下，根据实际业务情况及流量，对一些服务和页面有策略的不处理或换种简单的方式处理，从而释放服务器资源以保证核心交易正常运作或高效运作。

可以通过服务降级功能临时屏蔽某个出错的非关键服务，并定义降级后的返回策略。

- `mock=force:return+null` 表示消费方对该服务的方法调用都直接返回 null 值，不发起远程调用。用来屏蔽不重要服务不可用时对调用方的影响。
- `mock=fail:return+null` 表示消费方对该服务的方法调用在失败后，再返回 null 值，不抛异常。用来容忍不重要服务不稳定时对调用方的影响。

##### 4、集群容错

集群容错

在集群调用失败时，Dubbo 提供了多种容错方案，缺省为 `failover` 重试。

**集群容错模式**：

**Failover Cluster**

失败自动切换，当出现失败，重试其它服务器。通常用于读操作，但重试会带来更长延迟。可通过 `retries="2"` 来设置重试次数(不含第一次)。

重试次数配置如下：

```
<dubbo:service retries="2" />
或
<dubbo:reference retries="2" />
或
<dubbo:reference>
	<dubbo:method name="findFoo" retries="2" />
</dubbo:reference>
```

**Failfast Cluster**

快速失败，只发起一次调用，失败立即报错。**通常用于非幂等性的写操作，比如新增记录**。

**Failsafe Cluster**

失败安全，出现异常时，直接忽略。**通常用于写入审计日志等操作**。

**Failback Cluster**

失败自动恢复，后台记录失败请求，定时重发。**通常用于消息通知操作**。

**Forking Cluster**

并行调用多个服务器，只要一个成功即返回。通常用于实时性要求较高的读操作，但需要浪费更多服务资源。可通过 `forks="2"` 来设置最大并行数。

**Broadcast Cluster**

广播调用所有提供者，逐个调用，任意一台报错则报错。通常用于通知所有提供者更新缓存或日志等本地资源信息。

**集群模式配置**

按照以下示例在服务提供方和消费方配置集群模式：

```
<dubbo:service cluster="failsafe" />
或
<dubbo:reference cluster="failsafe" />
```

##### 5、整合hystrix

Hystrix 旨在通过控制那些访问远程系统、服务和第三方库的节点，从而对延迟和故障提供更强大的容错能力。Hystrix具备拥有回退机制和断路器功能的线程和信号隔离，请求缓存和请求打包，以及监控和配置等功能。

springboot官方提供了对hystrix的集成，直接在pom.xml里加入依赖

```
 <dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

然后在Application类上增加`@EnableHystrix`来启用hystrix starter：

```
@SpringBootApplication
@EnableHystrix
public class ProviderApplication {
	...
}
```

A.**配置Provider端**

在Dubbo的Provider上增加`@HystrixCommand`配置，这样调用就会经过Hystrix代理。

```
@Service(version = "1.0.0")
public class HelloServiceImpl implements HelloService {
    @HystrixCommand
    @Override
    public String sayHello(String name) {
        // System.out.println("async provider received: " + name);
        // return "annotation: hello, " + name;
        throw new RuntimeException("Exception to show hystrix enabled.");
    }
}
```

B.**配置Consumer端**

对于Consumer端，则可以增加一层method调用，并在method上配置`@HystrixCommand`。当调用出错时，会走到`fallbackMethod= "reliable"`的调用里。

```
@Reference(version = "1.0.0")
private HelloService demoService;

@HystrixCommand(fallbackMethod = "reliable")
public String doSayHello(String name) {
    return demoService.sayHello(name);
}
public String reliable(String name) {
    return "hystrix fallback value";
}
```