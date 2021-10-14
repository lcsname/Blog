---
layout: posts
title: 有关Servlet的生命周期
date: 2018-08-21 00:38:36
categories: Servlet
tags: [Servlet]
top: false
---
以前开发一个小型的系统，都是基于JSP/Servlet，而且现在公司有些服务依然用的这种模式，以前08,09年那会来说，就很牛逼了，而且是对一个保险系统来说，Service的多线程保证数据一致性，真的是很强，但是对于10年后的今天来说，对于一个JavaWeb项目来说，还在用这种模式的是在太少见了，不过他的一些原理还是要懂得。比如Servlet的生命周期。你真的了解吗？？？

<!--more--> 

Servlet运行在Servlet容器中，其生命周期由容器来管理。Servlet的生命周期通过`javax.servlet.Servlet`接口中的`init()`、`service()`和`destroy()`方法来表示。

Servlet的生命周期包含了下面4个阶段：

**1.加载和实例化**

**2.初始化**

**3.请求处理**

**4.服务终止**

Web服务器在与客户端交互时Servlet的工作过程是:

- 在客户端对web服务器发出请求。
- web服务器接收到请求后将其发送给Servlet。
- Servlet容器为此产生一个实例对象并调用ServletAPI中相应的方法来对客户端HTTP请求进行处理,然后将处理的响应结果返回给WEB服务器。
- web服务器将从Servlet实例对象中收到的响应结构发送回客户端。

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx55LuANVtwAAA26kvu4CA257.jpg)

#### **Servlet的生命周期**

##### 一、加载和实例化

 Servlet容器负责加载和实例化Servlet。当Servlet容器启动时，或者在容器检测到需要这个Servlet来响应第一个请求时，创建Servlet实例。当Servlet容器启动后，它必须要知道所需的Servlet类在什么位置，Servlet容器可以从本地文件系统、远程文件系统或者其他的网络服务中通过类加载器加载Servlet类，成功加载后，容器创建Servlet的实例。因为容器是通过Java的反射API来创建Servlet实例，调用的是Servlet的默认构造方法（即不带参数的构造方法），所以我们在编写Servlet类的时候，不应该提供带参数的构造方法。

##### 二、初始化

 在Servlet实例化之后，容器将调用Servlet的`init()`方法初始化这个对象。初始化的目的是为了让Servlet对象在处理客户端请求前完成一些初始化的工作，如建立数据库的连接，获取配置信息等。对于每一个Servlet实例，init()方法只被调用一次。在初始化期间，Servlet实例可以使用容器为它准备的ServletConfig对象从Web应用程序的配置信息（在web.xml中配置）中获取初始化的参数信息。在初始化期间，如果发生错误，Servlet实例可以抛出ServletException异常或者UnavailableException异常来通知容器。ServletException异常用于指明一般的初始化失败，例如没有找到初始化参数；而UnavailableException异常用于通知容器该Servlet实例不可用。例如，数据库服务器没有启动，数据库连接无法建立，Servlet就可以抛出UnavailableException异常向容器指出它暂时或永久不可用。

1、如何配置Servlet的初始化参数？

在web.xml中该Servlet的定义标记中，比如：

```
<servlet>
    <servlet-name>TimeServlet</servlet-name>
    <servlet-class>com.yaoyao.servlet.basic.TimeServlet</servlet-class>
    <init-param>
        <param-name>user</param-name>
        <param-value>username</param-value>
    </init-param>
    <init-param>
        <param-name>blog</param-name>
        <param-value>http://。。。</param-value>
    </init-param>
</servlet>
```

> 配置了两个初始化参数user和blog它们的值分别为`username`和`http://。。。`， 这样以后要修改用户名和博客的地址不需要修改Servlet代码，只需修改配置文件即可。

2、如何读取Servlet的初始化参数？

ServletConfig中定义了如下的方法用来读取初始化参数的信息：

```
public String getInitParameter(String name)
```

> 参数：初始化参数的名称。
> 返回：初始化参数的值，如果没有配置，返回null。

3、init(ServletConfig)方法执行次数

在Servlet的生命周期中，该方法执行一次。

4、init(ServletConfig)方法与线程

该方法执行在单线程的环境下，因此开发者不用考虑线程安全的问题。

5、init(ServletConfig)方法与异常

 该方法在执行过程中可以抛出ServletException来通知Web服务器Servlet实例初始化失败。一旦ServletException抛出，Web服务器不会将客户端请求交给该Servlet实例来处理，而是报告初始化失败异常信息给客户端，该Servlet实例将被从内存中销毁。如果在来新的请求，Web服务器会创建新的Servlet实例，并执行新实例的初始化操作

##### 三、请求处理

　　Servlet容器调用Servlet的service()方法对请求进行处理。要注意的是，在service()方法调用之前，**init()方法必须成功执行**。在service()方法中，Servlet实例通过ServletRequest对象得到客户端的相关信息和请求信息，在对请求进行处理后，调用ServletResponse对象的方法设置响应信息。在service()方法执行期间，如果发生错误，Servlet实例可以抛出ServletException异常或者UnavailableException异常。如果UnavailableException异常指示了该实例永久不可用，Servlet容器将调用实例的destroy()方法，释放该实例。此后对该实例的任何请求，都将收到容器发送的HTTP 404（请求的资源不可用）响应。如果UnavailableException异常指示了该实例暂时不可用，那么在暂时不可用的时间段内，对该实例的任何请求，都将收到容器发送的HTTP 503（服务器暂时忙，不能处理请求）响应。

1、service()方法的职责

service()方法为Servlet的核心方法，客户端的业务逻辑应该在该方法内执行，典型的服务方法的开发流程为：

> 解析客户端请求-〉执行业务逻辑-〉输出响应页面到客户端

2、service()方法与线程

为了提高效率，Servlet规范要求一个Servlet实例必须能够同时服务于多个客户端请求，即service()方法运行在多线程的环境下，Servlet开发者必须保证该方法的线程安全性。

3、service()方法与异常

service()方法在执行的过程中可以抛出ServletException和IOException。其中ServletException可以在处理客户端请求的过程中抛出，比如请求的资源不可用、数据库不可用等。一旦该异常抛出，容器必须回收请求对象，并报告客户端该异常信息。IOException表示输入输出的错误，编程者不必关心该异常，直接由容器报告给客户端即可。

注意事项说明：

1) 当Server Thread线程执行Servlet实例的init()方法时，所有的Client Service Thread线程都不能执行该实例的service()方法，更没有线程能够执行该实例的destroy()方法，因此Servlet的init()方法是工作在单线程的环境下，开发者不必考虑任何线程安全的问题。

2) 当服务器接收到来自客户端的多个请求时，服务器会在单独的Client Service Thread线程中执行Servlet实例的service()方法服务于每个客户端。此时会有多个线程同时执行同一个Servlet实例的service()方法，因此必须考虑线程安全的问题。

3) 虽然service()方法运行在多线程的环境下，并不一定要同步该方法。而是要看这个方法在执行过程中访问的资源类型及对资源的访问方式。分析如下：

- 如果service()方法没有访问Servlet的成员变量也没有访问全局的资源比如静态变量、文件、数据库连接等，而是只使用了当前线程自己的资源，比如非指向全局资源的临时变量、request和response对象等。该方法本身就是线程安全的，不必进行任何的同步控制。
- 如果service()方法访问了Servlet的成员变量，但是对该变量的操作是只读操作，该方法本身就是线程安全的，不必进行任何的同步控制。
- 如果service()方法访问了Servlet的成员变量，并且对该变量的操作既有读又有写，通常需要加上同步控制语句。
- 如果service()方法访问了全局的静态变量，如果同一时刻系统中也可能有其它线程访问该静态变量，如果既有读也有写的操作，通常需要加上同步控制语句。
- 如果service()方法访问了全局的资源，比如文件、数据库连接等，通常需要加上同步控制语句。

##### 四、服务终止

　　当容器检测到一个Servlet实例应该从服务中被移除的时候，容器就会调用实例的destroy()方法，以便让该实例可以释放它所使用的资源，保存数据到持久存储设备中。当需要释放内存或者容器关闭时，容器就会调用Servlet实例的destroy()方法。在destroy()方法调用之后，容器会释放这个Servlet实例，该实例随后会被Java的垃圾收集器所回收。如果再次需要这个Servlet处理请求，Servlet容器会创建一个新的Servlet实例。

 在整个Servlet的生命周期过程中，创建Servlet实例、**调用实例的init()和destroy()方法都只进行一次**，当初始化完成后，Servlet容器会将该实例保存在内存中，通过调用它的service()方法，为接收到的请求服务。

#### 转发与重定向的一些认知

![img](http://www.ao10001.wang/group1/M00/00/01/rB9p_Fx56B-ABCjPAADdo9OYt0I113.png)

`HttpServletResponse.sendRedirect`方法对浏览器的请求直接作出响应，响应的结果就是告诉浏览器去重新发出对另外一个URL的 访问请求，这个过程好比有个绰号叫“浏览器”的人写信找张三借钱，张三回信说没有钱，让“浏览器”去找李四借，并将李四现在的通信地址告诉给了“浏览器”。于是，“浏览器”又按张三提供通信地址给李四写信借钱，李四收到信后就把钱汇给了“浏览器”。可见，“浏览器”一共发出了两封信和收到了两次回复， “浏览器”也知道他借到的钱出自李四之手。

`RequestDispatcher.forward`方法在服务器端内部将请求转发给另外一个资源，浏览器只知道发出了请求并得到了响应结果，并不知道在服务器程序内部发生了转发行为。这个过程好比绰号叫“浏览器”的人写信找张三借钱，张三没有钱，于是张三找李四借了一些钱，甚至还可以加上自己的一些钱，然后再将这些钱汇给了“浏览器”。可见，“浏览器”只发 出了一封信和收到了一次回复，他只知道从张三那里借到了钱，并不知道有一部分钱出自李四之手。

- 转发使用的是getRequestDispatcher()方法;重定向使用的是sendRedirect();
- 转发：浏览器URL的地址栏不变。重定向：浏览器URL的地址栏改变；
- 转发是服务器行为，重定向是客户端行为；
- 转发是浏览器只做了一次访问请求。重定向是浏览器做了至少两次的访问请求；
- 转发2次跳转之间传输的信息不会丢失，重定向2次跳转之间传输的信息会丢失（request范围）。