---
title: "Java知识点总结"
date: 2021-04-13T00:29:44+08:00
draft: true
Categories: ["java"]
Tags: ["java", "spring"]
---



## 1. Spring中Ioc和AOP

**IOC：**控制反转，其实是一种设计思想，指将原本程序中的对象之间的依赖关系交给IOC容器统一管理，并由其创建对象。IOC容器实际上就是一个Map（key，value），Map中存放的是各种对象。

![image-20210317000541983](http://cdn.bearkchan.top/image-20210317000541983.png)

### [11. Redis 是如何判断数据是否过期的呢？](https://snailclimb.gitee.io/javaguide/#/docs/database/Redis/redis-all?id=_11-redis-是如何判断数据是否过期的呢？)

Redis 通过一个叫做过期字典（可以看作是 hash 表）来保存数据过期的时间。过期字典的键指向 Redis 数据库中的某个 key(键)，过期字典的值是一个 long long 类型的整数，这个整数保存了 key 所指向的数据库键的过期时间（毫秒精度的 UNIX 时间戳）。

![redis过期字典](https://snailclimb.gitee.io/javaguide/docs/database/Redis/images/redis-all/redis%E8%BF%87%E6%9C%9F%E6%97%B6%E9%97%B4.png)

过期字典是存储在 redisDb 这个结构里的：

```c
typedef struct redisDb {
    ...

    dict *dict;     //数据库键空间,保存着数据库中所有键值对
    dict *expires   // 过期字典,保存着键的过期时间
    ...
} redisDb;Copy to clipboardErrorCopied
```

### [12. 过期的数据的删除策略了解么？](https://snailclimb.gitee.io/javaguide/#/docs/database/Redis/redis-all?id=_12-过期的数据的删除策略了解么？)

如果假设你设置了一批 key 只能存活 1 分钟，那么 1 分钟后，Redis 是怎么对这批 key 进行删除的呢？

常用的过期数据的删除策略就两个（重要！自己造缓存轮子的时候需要格外考虑的东西）：

1. **惰性删除** ：只会在取出 key 的时候才对数据进行过期检查。这样对 CPU 最友好，但是可能会造成太多过期 key 没有被删除。
2. **定期删除** ： 每隔一段时间抽取一批 key 执行删除过期 key 操作。并且，Redis 底层会通过限制删除操作执行的时长和频率来减少删除操作对 CPU 时间的影响。

定期删除对内存更加友好，惰性删除对 CPU 更加友好。两者各有千秋，所以 Redis 采用的是 **定期删除+惰性/懒汉式删除** 。

但是，仅仅通过给 key 设置过期时间还是有问题的。因为还是可能存在定期删除和惰性删除漏掉了很多过期 key 的情况。这样就导致大量过期 key 堆积在内存里，然后就 Out of memory 了。

怎么解决这个问题呢？答案就是： **Redis 内存淘汰机制。**

### [13. Redis 内存淘汰机制了解么？](https://snailclimb.gitee.io/javaguide/#/docs/database/Redis/redis-all?id=_13-redis-内存淘汰机制了解么？)

> 相关问题：MySQL 里有 2000w 数据，Redis 中只存 20w 的数据，如何保证 Redis 中的数据都是热点数据?

Redis 提供 6 种数据淘汰策略：

1. **volatile-lru（least recently used）**：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰
2. **volatile-ttl**：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰
3. **volatile-random**：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰
4. **allkeys-lru（least recently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的 key（这个是最常用的）
5. **allkeys-random**：从数据集（server.db[i].dict）中任意选择数据淘汰
6. **no-eviction**：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。这个应该没人使用吧！

4.0 版本后增加以下两种：

1. **volatile-lfu（least frequently used）**：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰
2. **allkeys-lfu（least frequently used）**：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的 key

### [14. Redis 持久化机制(怎么保证 Redis 挂掉之后再重启数据可以进行恢复)](https://snailclimb.gitee.io/javaguide/#/docs/database/Redis/redis-all?id=_14-redis-持久化机制怎么保证-redis-挂掉之后再重启数据可以进行恢复)

很多时候我们需要持久化数据也就是将内存中的数据写入到硬盘里面，大部分原因是为了之后重用数据（比如重启机器、机器故障之后恢复数据），或者是为了防止系统故障而将数据备份到一个远程位置。

Redis 不同于 Memcached 的很重要一点就是，Redis 支持持久化，而且支持两种不同的持久化操作。**Redis 的一种持久化方式叫快照（snapshotting，RDB），另一种方式是只追加文件（append-only file, AOF）**。这两种方法各有千秋，下面我会详细这两种持久化方法是什么，怎么用，如何选择适合自己的持久化方法。

**快照（snapshotting）持久化（RDB）**

Redis 可以通过创建快照来获得存储在内存里面的数据在某个时间点上的副本。Redis 创建快照之后，可以对快照进行备份，可以将快照复制到其他服务器从而创建具有相同数据的服务器副本（Redis 主从结构，主要用来提高 Redis 性能），还可以将快照留在原地以便重启服务器的时候使用。

快照持久化是 Redis 默认采用的持久化方式，在 Redis.conf 配置文件中默认有此下配置：

```conf
save 900 1           #在900秒(15分钟)之后，如果至少有1个key发生变化，Redis就会自动触发BGSAVE命令创建快照。

save 300 10          #在300秒(5分钟)之后，如果至少有10个key发生变化，Redis就会自动触发BGSAVE命令创建快照。

save 60 10000        #在60秒(1分钟)之后，如果至少有10000个key发生变化，Redis就会自动触发BGSAVE命令创建快照。Copy to clipboardErrorCopied
```

**AOF（append-only file）持久化**

与快照持久化相比，AOF 持久化 的实时性更好，因此已成为主流的持久化方案。默认情况下 Redis 没有开启 AOF（append only file）方式的持久化，可以通过 appendonly 参数开启：

```conf
appendonly yesCopy to clipboardErrorCopied
```

开启 AOF 持久化后每执行一条会更改 Redis 中的数据的命令，Redis 就会将该命令写入硬盘中的 AOF 文件。AOF 文件的保存位置和 RDB 文件的位置相同，都是通过 dir 参数设置的，默认的文件名是 appendonly.aof。

在 Redis 的配置文件中存在三种不同的 AOF 持久化方式，它们分别是：

```conf
appendfsync always    #每次有数据修改发生时都会写入AOF文件,这样会严重降低Redis的速度
appendfsync everysec  #每秒钟同步一次，显示地将多个写命令同步到硬盘
appendfsync no        #让操作系统决定何时进行同步Copy to clipboardErrorCopied
```

为了兼顾数据和写入性能，用户可以考虑 appendfsync everysec 选项 ，让 Redis 每秒同步一次 AOF 文件，Redis 性能几乎没受到任何影响。而且这样即使出现系统崩溃，用户最多只会丢失一秒之内产生的数据。当硬盘忙于执行写入操作的时候，Redis 还会优雅的放慢自己的速度以便适应硬盘的最大写入速度。

**相关 issue** ：[783：Redis 的 AOF 方式](https://github.com/Snailclimb/JavaGuide/issues/783)

**拓展：Redis 4.0 对于持久化机制的优化**

Redis 4.0 开始支持 RDB 和 AOF 的混合持久化（默认关闭，可以通过配置项 `aof-use-rdb-preamble` 开启）。

如果把混合持久化打开，AOF 重写的时候就直接把 RDB 的内容写到 AOF 文件开头。这样做的好处是可以结合 RDB 和 AOF 的优点, 快速加载同时避免丢失过多的数据。当然缺点也是有的， AOF 里面的 RDB 部分是压缩格式不再是 AOF 格式，可读性较差。

**补充内容：AOF 重写**

AOF 重写可以产生一个新的 AOF 文件，这个新的 AOF 文件和原有的 AOF 文件所保存的数据库状态一样，但体积更小。

AOF 重写是一个有歧义的名字，该功能是通过读取数据库中的键值对来实现的，程序无须对现有 AOF 文件进行任何读入、分析或者写入操作。

在执行 BGREWRITEAOF 命令时，Redis 服务器会维护一个 AOF 重写缓冲区，该缓冲区会在子进程创建新 AOF 文件期间，记录服务器执行的所有写命令。当子进程完成创建新 AOF 文件的工作之后，服务器会将重写缓冲区中的所有内容追加到新 AOF 文件的末尾，使得新旧两个 AOF 文件所保存的数据库状态一致。最后，服务器用新的 AOF 文件替换旧的 AOF 文件，以此来完成 AOF 文件重写操作





## 数组去重：

使用LinkedHashSet的方式进行数组数据的保存，最后通过toArray再进行

## Spring相关知识内容

### IOC

IOC容器的接口就是`ApplicationContext`：

```java
// 当前应用的xml配置文件在ClassPath下
ApplicationContext ioc = new ClassPathXmlApplicationContext("ioc.xml");
```

根据spring的配置文件得到IOC容器对象。

> src，源码包开始的路径，称为类路径的开始；
>
>    \*    所有源码包里面的东西都会被合并放在类路径里面；
>
>    \*    java：/bin/
>
>    \*    web：/WEB-INF/classes/

IOC的底层使用`Map<K,V>`的方式存放所有的Spring bean对象。

一般的工作流程是：

```
xml配置文件--->Resource解析--->BeanDefition---->BeanFactory生产对象。
```

**什么叫依赖倒置：**

其实所谓的依赖倒置就是将原有的高层建筑依赖底层建筑进行反转，实现高层建筑和底层建筑全部依赖于接口，并由底层去实现接口的实现类的方式。

**什么是依赖注入：**

通过将底层类对象作为参数传入上层类的方式，实现上层类对下层类的控制。

IOC Container工厂，隐藏了具体的创建实例的细节，使得用户不需要去理解底层的实例创建过程，直接拿来用就可以了。



### AOP

面向切面编程，基于OOP基础之上的新的编程思想；指在程序运行期间，`将某段代码动态的切入到指定方法的指定位置进行运行`的这种编程方式，面向切面编程。

AOP可以用来进行`日志功能、权限验证、事务控制、安全检查`等方面。

---

动态代理底层的原理：

```java
public class CalculatorProxy {

  /**
     * 为传入的参数对象创建一个动态代理对象
     * @param calculator
     * @return
     *
     * Calculator calculator:被代理对象；（宝宝）
     * 返回的：宋喆
     */
  public static Calculator getProxy(final Calculator calculator) {
    // TODO Auto-generated method stub

    //方法执行器。帮我们目标对象执行目标方法
    InvocationHandler h = new InvocationHandler() {
      /**
             * Object proxy：代理对象；给jdk使用，任何时候都不要动这个对象
             * Method method：当前将要执行的目标对象的方法
             * Object[] args：这个方法调用时外界传入的参数值
             */
      @Override
      public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable {

        //System.out.println("这是动态代理将要帮你执行方法...");
        Object result = null;
        try {
          LogUtils.logStart(method, args);

          // 利用反射执行目标方法
          //目标方法执行后的返回值
          result = method.invoke(calculator, args);
          LogUtils.logReturn(method, result);
        } catch (Exception e) {
          LogUtils.logException(method,e);
        }finally{
          LogUtils.logEnd(method);

        }

        //返回值必须返回出去外界才能拿到真正执行后的返回值
        return result;
      }
    };
    Class<?>[] interfaces = calculator.getClass().getInterfaces();
    ClassLoader loader = calculator.getClass().getClassLoader();

    //Proxy为目标对象创建代理对象；
    Object proxy = Proxy.newProxyInstance(loader, interfaces, h);
    return (Calculator) proxy;
  }

}
```

具体的使用细则：

通过代理类Proxy的newProxyInstance方法创建代理对象，并进行强转。参数包括类加载器，代理类所有的接口集合，以及集成Invocationhandler的实现类，重写其中的invoke方法，并在其中利用反射执行目标方法，

```java
result = method.invoke(代理对象，方法的所有参数args);
```

并且最后要将result返回值返回出去，只有这样外界才能拿到真正执行后的返回值。

> jdk默认的动态代理，如果目标对象没有实现任何接口，是无法为他创建代理对象的；

---

![image-20210319000241870](http://cdn.bearkchan.top/image-20210319000241870.png)

而在Spring中，使用切入点表达式简化动态代理流程。

1. 然后编写配置，将目标类和切面类（封装了通知方法（在目标方法执行前后执行的方法））加入到ioc容器中。
2. 还应该告诉Spring到底哪个是切面类@Aspect
3. 告诉Spring，切面类里面的每一个方法，都是何时何地运行；

```java
@Before("execution(public int com.atguigu.impl.MyMathCalculator.*(int, int))")
public static void logStart(){
  System.out.println("【xxx】方法开始执行，用的参数列表【xxx】");
}

//想在目标方法正常执行完成之后执行
@AfterReturning("execution(public int com.atguigu.impl.MyMathCalculator.*(int, int))")
public static void logReturn(){
  System.out.println("【xxxx】方法正常执行完成，计算结果是：");
}

//想在目标方法出现异常的时候执行
@AfterThrowing("execution(public int com.atguigu.impl.MyMathCalculator.*(int, int))")
public static void logException() {
  System.out.println("【xxxx】方法执行出现异常了，异常信息是：；这个异常已经通知测试小组进行排查");
}

//想在目标方法结束的时候执行
@After("execution(public int com.atguigu.impl.MyMathCalculator.*(int, int))")
public static void logEnd() {
  System.out.println("【xxx】方法最终结束了");
}
```



## spring生命周期的问题

1. 实例化
2. 属性赋值
3. 初始化
4. 销毁

## spring常用注解

1. @Component：使用无参构造器创建一个bean对象，默认id为类名首字母小写
2. @Controller：用于表现层的注解
3. @Service：用于服务层，与dao层交互的
4. @Repository：dao层，数据持久化
5. @Bean：
6. @Autowire
7. @Qualifier：用于和@Autowire搭配使用，用于指定bean的id
8. @Value：将外部的值动态注入到bean中，作为成员变量的默认值
9. @Scope：指定bean的作用范围
   - singleton：单例
   - prototype：多例
   - request
   - session
   - globalsession

## sql语法

union：用于两个select语句的结果集的合并

## object的常用方法

- clone()
- hashCode和equals()
- Wait()和notify()
- toString()
- getClass()
- Finalize()：用于垃圾回收调用

## 问了int和integer的区别、

## spring mvc中的注解

- @RestController
- @RequestMapping
- @PathVariable
- @RequestParam
- @RequestBody



## spring mvc如何将前端数据传到后端

@RequestBody

pojo数据绑定



## SpringMVC的工作原理

![image-20210319005123058](http://cdn.bearkchan.top/image-20210319005123058.png)

DispathcerServlet-->handleMapping获取handlerExecutionChain对象--->handleAdapter---->调用目标方法获得ModeAndView--->ViewResolver--->进行试图渲染--->返回前端View



## spring boot文件类型

- application.yml
- application.properties



## SpringBoot的自动装箱机制

@SpringBootApplication：@Configuration+@EnableAutoConfiguration+@ComponentScan

- SpringBoot先加载所有的自动配置类  xxxxxAutoConfiguration
- 每个自动配置类按照条件进行生效，默认都会绑定配置文件指定的值。xxxxProperties里面拿。xxxProperties和配置文件进行了绑定
- 生效的配置类就会给容器中装配很多组件
- 只要容器中有这些组件，相当于这些功能就有了
- 定制化配置

- - 用户直接自己@Bean替换底层的组件
  - 用户去看这个组件是获取的配置文件什么值就去修改。

**xxxxxAutoConfiguration ---> 组件  --->** **xxxxProperties里面拿值  ----> application.properties**

## mybatis及mybatis-plus如何导入

![image-20210319003951393](http://cdn.bearkchan.top/image-20210319003951393.png)

![image-20210319004002810](http://cdn.bearkchan.top/image-20210319004002810.png)



## mysql左连接和右边连接的区别

LIFT JOIN 左连接，以左表为主表， 不满足on 的条件留在左表，右边数据为NULL

RIGHT JOIN 右连接 以右表为主表，不满足on 条件的留在右表中，左表NULL

## linux看日志的命令

tail -f -n 10



## docker进入容器的命令

docker exec -it 775c7c9ee1e1 /bin/bash  



