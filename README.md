# SSM高并发秒杀、红包API设计

## maven初始化项目

```bash
# 其中archetypeCatalog=internal表示不从远程获取archetype的分类
# groudId对应包名,artifactId对应项目名
# -X指定maven以DEBUG方式运行，输出详细信息。之前的没有加这个参数执行项目生成的时候输出Generating project in Interactive mode就卡在那里，加上-X参数后定位到原来是从远程获取archetypeCatalog了,-B表示静默
$ mvn archetype:generate -DarchetypeCatalog=internal -DgroudId=io.github.consoles -DartifactId=seckill -DarchetypeArtifactId=maven-archetype-webapp -X -B
```

上面的命令生成的webapp默认使用的是servlet2.3已经太老了,我们可以在tomcat的实例目录中拷贝一个`web.xml`的头部过来(tomcat目录下的`webapps/examples/WEB-INF/web.xml`)

##  秒杀业务分析

![秒杀业务分析](http://7xlan5.com1.z0.glb.clouddn.com/miaosha.png)

如上图所示,秒杀系统的核心其实是对库存的处理。当用户成功秒杀了一个商品的时候会做2件事:

1. 减库存
2. 记录购买明细

以上的2件事组成了一个完整的事务,需要准确的数据落地。以上之所以需要事务就是因为任何一个方面不一致就会出现超卖或者少卖的情况。用户的*购买行为*包括以下的3个方面:1.谁购买成功了;2.成功的时间和有效期;3.付款和发货信息。

**事务机制依然是当前最有效的数据落地方案。**

## 秒杀业务难点

当多个用户同时秒杀同一件商品的时候,就会出现*竞争*。反映到数据库就是事务和行级锁。

1. start transaction
2. update 库存数量(竞争出现在这个阶段)
3. insert 购买明细
4. commit

![使用行级锁保证事务](http://7xlan5.com1.z0.glb.clouddn.com/mysql-row-lock.png)

当一个人秒杀成功的时候其他人会等待,直到该事务commit。从上面可以看出秒杀的难点在于*如何高效处理竞争*。

## DAO层

接口设计 + SQL语句编写,代码和SQL进行分离,方便Review。逻辑程序应该放在Service层完成。

### DB设计与编码

在设计表的时候最好有一个类型为`timestamp`的`create_time`字段,用来表示该行数据的创建时间。Mybatis可以通过注解和xml编写sql,但是本质上注解是java源码,改动需要重新编译,建议使用*xml编写sql*。

使用mybatis只需要写dao接口,不写实现类。

### Junit单元测试

生成dao的测试用例。

`org.mybatis.spring.MyBatisSystemException: nested exception is org.apache.ibatis.binding.BindingException: Parameter 'offset' not found. Available parameters are [0, 1, param1, param2]`

```java
List<Seckill> queryAll(int offset, int limit);
```

以上的方法在运行的时候会变成下面的形式`queryAll(arg0,arg1);`当一个参数的时候没有问题,当有多个参数的时候需要高度mybatis哪个位置是什么参数,解决方法是修改dao接口,参数增加`@Param`注解。

## Service层

设计业务接口的时候应该站在使用者的角度设计接口。初学者总是关注接口如何实现,这样设计出来的接口往往非常冗余,别人用接口的时候非常不便。主要包含以下的3个方面:

1. 方法的定义粒度;
2. 参数,例如不应该传入Map或者一大串的参数;
3. return/异常

DTO的作用就是方面web层和service层的数据传输。Spring声明式事务只回滚运行时异常。

> 使用枚举表示常量数据字典更优雅。

### Spring声明式事务

使用`@Transactional`注解来声明事务,尽量避免一次配置永久生效,因为事务是一个特别精细的过程。使用注解有以下优点:

1. 开发团队达成一致约定,明确标注事务方法的编程风格;
2. 保证事务方法的执行时间尽可能短,不要穿插其他的网络操作,例如RPC/HTTP(Redis等),或者将它们剥离到事务方法外部;
3. 不是所有的方法都需要事务。例如只读操作不需要事务。

## Web层

RESTFul API被Rails发扬光大,强调资源的一种状态,所以URL中使用动词不优雅。DTO用于web和service传输数据。

### SpringMVC运行流程

![SpringMVC运行流程](http://7xlan5.com1.z0.glb.clouddn.com/spring-mvc.png)

如上图所示:

1. 用户所有的请求都会映射到`DispatcherServlet`,这是一个中央控制器的Servlet,会拦截所有请求。
2. `DispatcherServlet`默认会用到`DefaultAnnotationHandlerMapping`来将URL映射到具体的Handler。
3. 映射完之后会默认使用`DefaultAnnotationHandlerApdater`,用来做Handler适配,最终会衔接到我们自己开发的`SeckillController`。
4. `DefaultAnnotationHandlerApdater`的最终的产出是`ModelAndView`(可以用字符串表示,例如:seckill/list,表示seckill目录下的list.jsp),ModelAndView同时交付到`DispatcherServlet`。
5. 中央控制器发现我们引用的是一个`InternalResourceViewResolver`,会将Model填充到View中返回给用户。

实际需要我们开发的仅仅是上图蓝色部分的`SeckillController`,其他的部分使用注解配置即可。

### 详情页秒杀逻辑

![详情页秒杀逻辑](http://7xlan5.com1.z0.glb.clouddn.com/seckill-detail.png)

## 高并发优化

Q1:为什么要获取系统时间?
A1:静态页面(详情页和相关js)会被部署到cdn上,这样就无法拿到系统时间了。

Q2:获取系统时间的接口需要优化么?
A2:不需要。因为java访问一次内存大概需要10ns,1s可以执行1亿次`new Date();`这样的操作。

秒杀地址接口暴露无法使用CDN缓存,但是可以采用服务端缓存。单台redis可以抗十万DPS,而集群可以抗百万DPS。一致性维护成本非常低,可以这样实现:给redis设置一个超时,超时之后直接进行超时穿透,访问MySQL或者MySQL更新的时候写入redis。

秒杀操作主要有以下困难:无法使用CDN,后端无法缓存(缓存会导致数据一致性问题,必须通过MySQL的事务解决一致性问题);单行数据竞争问题(热点商品)

### 常用的秒杀架构

![秒杀架构](http://7xlan5.com1.z0.glb.clouddn.com/seckill-arch.png)

上面的架构可以抗住非常高的并发,如果redis做成集群的话(例如100个redis实例,每台10W,就可以抗住1000W的并发),分布式MQ可以支持几十万的QPS。但是有以下的痛点:

1. 运维成本高,NoSQL、MQ由于追求分布式,没有MySQL稳定;
2. 开发成本高,数据一致性,回滚方案等;
3. 难以保证幂等性(重复秒杀问题),一般的方案是需要再维护一个MQ用来记录用户操作;
4. 这是一个不适合新手的架构。

一条update语句的QPS大概是4W QPS,如下图:

![MySQL更新语句的QPS](http://i2.piimg.com/567571/9851d0027f49ee52.png)

由此可见不是MySQL本身的问题造成了性能瓶颈。

![java事务分析](http://i2.piimg.com/567571/05405348f8ca8409.png)

由上面的分析可知,并发被串行化了,很多的阻塞。

![性能瓶颈](http://i2.piimg.com/567571/aedd7ccb7652bc2a.png)

性能瓶颈不是java慢和MySQL慢,而是每个事务中的网络延迟和GC操作。如果以上的所有操作2ms完成,那么QPS最高只有500,对于秒杀系统来说还不够。

#### 网络延迟分析

![同城机房网络延迟](http://i2.piimg.com/567571/7c9b69a40fcbf5a0.png)

如果在异地机房,例如北京和上海的往返距离是2600km,光在真空中的速度是30Wkm/s,在玻璃中的速度是30*2/3 = 20Wkm/s,则往返时间为13ms(实际值在20ms左右,也就是说QPS最高50)

### 优化分析

优化方向是行级锁在commit之后释放,减少行级锁持有的时间。

我们在客户端通过以下的2个条件判断update更新库存成功:

1. update自身没报错。
2. 客户端确认update影响记录数。

一个简单的优化就是调整SQL语句的顺序,先insert购买明细然后update减库存,减少一部网络延迟和GC

优化思路:

把客户端逻辑放到MySQL服务器上,避免网络延迟和GC影响。有以下的2种解决方案:

1. 定制MySQL方案:`update /* + [auto_commit] */`,需修改MySQL源码。早起Alibaba方案,当update影响记录数为1自动commit,为0rollback。不给java客户端和MySQL之间的网络延迟。
2. 使用存储过程:整个事务在MySQL服务端完成。

### 优化总结

1. 前端控制:暴露接口,防止按钮重复点击
2. 动静分离:CDN缓存,后端缓存
3. 事务竞争优化:减少事务锁时间

## 使用redis优化地址暴露接口

> 在使用`redis-cli`命令进入到redis shell时可以使用`info`命令查看redis运行信息。`dbsize`命令可查看redis中数据的条数。

由于redis存储的是二进制数组,存储java对象需要进行序列化,[java各种序列化方案的性能分析](https://github.com/eishay/jvm-serializers/wiki),其中最优方案是Google的protostuff

## 附录

### osx下idea快捷键

- 针对接口生成测试类:command + shift + T
- 自动生成constructor,getter,setter,toString:command + N

### 常见问题
- Spring `@Autowired`注解报错:`Could not autowire.No beans if xxx type found.`,[解决方案](http://www.oschina.net/question/202626_181237?fromerr=BYQ08rsA)
