---
layout: posts
title: SpringBoot的缓存注解
date: 2018-08-12 00:38:36
categories: SpringBoot
tags: [SpringBoot,Redis]
top: false
---
从3.1开始，Spring引入了对Cache的支持。其使用方法和原理都类似于Spring对事务管理的支持。Spring Cache是作用在方法或者类上的，其核心思想是这样的：在调用一个缓存方法时会把该方法参数和返回结果作为一个键值对存放在缓存中，等到下次利用同样的参数来调用该方法时将不再执行该方法，而是直接从缓存中获取结果进行返回。所以在使用Spring Cache的时候要保证缓存的方法对于相同的方法参数要有相同的返回结果。 和Spring对事务管理的支持一样，Spring对Cache的支持也有基于注解和基于XML配置两种方式。我经常使用注解的方式，因为很方便。下面就来看看基于注解的方式吧。

<!--more--> 

Spring提供了几个注解来支持Spring Cache。其核心主要是`@Cacheable`和`@CacheEvict`。使用`@Cacheable`标记的方法在执行后Spring Cache将缓存其返回结果，而使用`@CacheEvict`标记的方法会在方法执行前或者执行后移除Spring Cache中的某些元素。

#### 一、@Cacheable

`@Cacheable`可以标记在一个方法上，也可以标记在一个类上。当标记在一个方法上时表示该方法是支持缓存的，当标记在一个类上时则表示该类所有的方法都是支持缓存的。

对于一个支持缓存的方法，Spring会在其被调用后将其返回值缓存起来，以保证下次利用同样的参数来执行该方法时可以直接从缓存中获取结果，而不需要再次执行该方法。Spring在缓存方法的返回值时是以键值对进行缓存的，值就是方法的返回结果，至于键的话，Spring又支持两种策略，默认策略和自定义策略，这个稍后会进行说明。需要注意的是**当一个支持缓存的方法在对象内部被调用时是不会触发缓存功能的**。

来看看`Cacheable`源码

```
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Cacheable {
    @AliasFor("cacheNames")
    String[] value() default {};
    @AliasFor("value")
    String[] cacheNames() default {};
    String key() default "";
    String keyGenerator() default "";
    String cacheManager() default "";
    String cacheResolver() default "";
    String condition() default "";
    String unless() default "";
    boolean sync() default false;
}
```

##### （一）Value属性

value属性指定Cache名称，这个是必须指定的，其表示当前方法的返回值是会被缓存在哪个Cache上的， 对应Cache的名称。其可以是一个Cache也可以是多个Cache，当需要指定多个Cache时其是一个**数组**。

例如这样：

```
@Cacheable("cache1")//Cache是发生在cache1上的
public User find(Integer id) {
    return null;
}
@Cacheable({"cache1", "cache2"})//Cache是发生在cache1和cache2上的
public User find(Integer id) {
    return null;
}
```

##### （二）Key属性

key属性是用来指定Spring缓存方法的返回结果时对应的key的 ，该属性支持SpringEL表达式。在没有指定该属性时，Spring将使用默认策略生成key。

自定义策略是指可以通过Spring的EL表达式来指定key ，这里的EL表达式可以使用方法参数及它们对应的属性。 使用方法参数时可以直接使用`#参数名`或者`#p参数index`

例如这样：

```
//将ID作为key
@Cacheable(value="users", key="#id")
public User find(Integer id) {
    return null;
}
//将第一参数作为key index从0开始计数
@Cacheable(value="users", key="#p0")
public User find(Integer id) {
    return null;
}
//基于对象的方式获取key
@Cacheable(value="users", key="#user.id")
public User find(User user) {
    return null;
}
//效果合上变的是一样的
@Cacheable(value="users", key="#p0.id")
public User find(User user) {
    return null;
}
```

除了上述使用方法参数作为key之外，Spring还提供了一个root对象可以用来生成key。通过该root对象可以获取到以下信息

| 属性名称    | 描述                        | 示例                 |
| ----------- | --------------------------- | -------------------- |
| methodName  | 当前方法名                  | #root.methodName     |
| method      | 当前方法                    | #root.method.name    |
| target      | 当前被调用的对象            | #root.target         |
| targetClass | 当前被调用的对象的class     | #root.targetClass    |
| args        | 当前方法参数组成的数组      | #root.args[0]        |
| caches      | 当前被调用的方法使用的Cache | #root.caches[0].name |

要使用root对象的属性作为key时也可以将`#root`省略，因为Spring默认使用的就是root对象的属性。

例如这样：

```
@Cacheable(value={"users", "xxx"}, key="caches[1].name")
public User find(User user) {
    return null;
}
```

##### （三）Condition属性

有的时候可能并不希望缓存一个方法所有的返回结果。通过condition属性可以实现这一功能。condition属性默认为空，表示将缓存所有的调用情形。其值是通过SpringEL表达式来指定的，当为true时表示进行缓存处理；当为false时表示不进行缓存处理，即每次调用该方法时该方法都会执行一次。

例如这样：表示只有当user的id为偶数时才会进行缓存

```
@Cacheable(value={"users"}, key="#user.id", condition="#user.id%2==0")
public User find(User user) {
    System.out.println("find user by user " + user);
    return user;
}
```

##### （四）Unless属性

这个属性其实和Condition属性取反，我认为应该做排除讲更合适些。

例如这样：`#result` 代表返回的结果集，这里意思是缓存所有的结果，排除返回结果为空的情况

```
@Cacheable(value = "CHAPTER", key = "#p0", unless = "#result == null")
public List<Chapter> getPartChapter(String book_id) {
    return null;
}
```

基本上常用的方法就这么多了，关于`keyGenerator` 属性，我没怎么用过，以后万一用到了再补充。

#### 二、@CachePut

在支持Spring Cache的环境下，对于使用@Cacheable标注的方法，Spring在每次执行前都会检查Cache中是否存在相同key的缓存元素，如果存在就不再执行该方法，而是直接从缓存中获取结果进行返回，否则才会执行并将返回结果存入指定的缓存中。@CachePut也可以声明一个方法支持缓存功能。与@Cacheable不同的是使用@CachePut标注的方法在执行前不会去检查缓存中是否存在之前执行过的结果，而是每次都会执行该方法，并将执行结果以键值对的形式存入指定的缓存中。

@CachePut也可以标注在类上和方法上。使用@CachePut时可以指定的属性跟@Cacheable是一样的。

#### 三、@CacheEvict

@CacheEvict是用来标注在需要清除缓存元素的方法或类上的。

当标记在一个类上时表示其中所有的方法的执行都会触发缓存的清除操作。@CacheEvict可以指定的属性有`value`、`key`、`condition`、`allEntries`和`beforeInvocation`。其中`value`、`key`和`condition`的语义与@Cacheable对应的属性类似。

来看看他的源码：

```
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface CacheEvict {
    @AliasFor("cacheNames")
    String[] value() default {};
    @AliasFor("value")
    String[] cacheNames() default {};
    String key() default "";
    String keyGenerator() default "";
    String cacheManager() default "";
    String cacheResolver() default "";
    String condition() default "";
    boolean allEntries() default false;
    boolean beforeInvocation() default false;
}
```

##### （一）、allEntries属性

`allEntries`是boolean类型，表示是否需要清除缓存中的所有元素。默认为`false`，表示不需要。当指定了`allEntries`为`true`时，Spring Cache将忽略指定的key。有的时候需要Cache一下清除所有的元素，这比一个一个清除元素更有效率。

例如这样：

```
//清除整个users缓存
@CacheEvict(value="users", allEntries=true) 
public void delete(Integer id) {
    System.out.println("delete user by id: " + id);
}
```

##### （二）、beforeInvocation属性

清除操作默认是在对应方法成功执行之后触发的，即方法如果因为抛出异常而未能成功返回时也不会触发清除操作。使用beforeInvocation可以改变触发清除操作的时间，当指定该属性值为true时，Spring会在调用该方法之前清除缓存中的指定元素。

例如这样：

```
//即使方法出现了异常情况，缓存也会被清除
@CacheEvict(value="users", beforeInvocation=true)
public void delete(Integer id) {
    int a = 1/0;
    System.out.println("delete user by id: " + id);
}
```

#### 四、@Caching

@Caching注解可以在一个方法或者类上同时指定多个Spring Cache相关的注解。其拥有三个属性：`cacheable`、`put`和`evict`，分别用于指定@Cacheable、@CachePut和@CacheEvict。

例如这样：

```
//这个方法表示 执行find方法会将结果缓存到users，同时清空cache2缓存中key为id的缓存，同时清空cache3缓存
@Caching(cacheable = @Cacheable("users"), 
         evict = { @CacheEvict("cache2"),@CacheEvict(value = "cache3", allEntries = true) }
        )
public User find(Integer id) {
   return null;
}
```

#### 五、使用自定义注解

Spring允许在配置可缓存的方法时使用自定义的注解，前提是自定义的注解上必须使用对应的注解进行标注。如有如下这么一个使用@Cacheable进行标注的自定义注解。

例如这样：

```
@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Cacheable(value="users")
public @interface MyCacheable {
}
```

那么在需要缓存的方法上使用@MyCacheable进行标注也可以达到同样的效果。

例如这样：

```
@MyCacheable
public User findById(Integer id) {
    System.out.println("find user by id: " + id);
    User user = new User();
    user.setId(id);
    user.setName("Name" + id);
    return user;
}
```

#### 六、配置SpringBoot对Cache的支持

我经常使用Redis作为缓存工具。所以我就拿Redis来说吧

首先得引入Redis依赖

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

配置类RedisConfig.java

```
@Configuration
@EnableCaching
public class RedisConfig extends CachingConfigurerSupport {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory factory) {
        RedisSerializer<String> redisSerializer = new StringRedisSerializer();
        Jackson2JsonRedisSerializer jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer(Object.class);

        //解决查询缓存转换异常的问题
        ObjectMapper om = new ObjectMapper();
        om.setVisibility(PropertyAccessor.ALL, JsonAutoDetect.Visibility.ANY);
        om.enableDefaultTyping(ObjectMapper.DefaultTyping.NON_FINAL);
        jackson2JsonRedisSerializer.setObjectMapper(om);

        // 配置序列化（解决乱码的问题）,过期时间10小时
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofSeconds(10 * 3600))
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(redisSerializer))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(jackson2JsonRedisSerializer))
                .disableCachingNullValues();

        return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .build();
    }
}
```

#### 七、关于在同一个类中调用缓存方法

当一个支持缓存的方法在对象内部被调用时是不会触发缓存功能的，这个也说过了，为了解决这个问题，要把缓存方法抽象出去，做成Service的方式，使用`@Service` 注解就成，然后在使用缓存方法的类中引入就完了。

嗯，也就这样了。