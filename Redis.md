
# 点评项目

> mysql密码：1234
>
> redis密码：123456
>
> 
>
> **1.本地redis使用：**
>
> 1.在这个界面打开cmd
>
> <img src="assets/image-20250116101445029.png" alt="image-20250116101445029" style="zoom: 50%;" />
>
> 2.redis-server.exe redis.windows.conf 启动redis
>
> 3.客户端工具
>
> 
>
> **2.项目介绍：**
>
> 1.前后端分离，单体项目
>
> 2.后端部署在tomcat服务器上，前端部署在nginx上，前端向nginx请求页面（静态资源），页面通过Ajax请求向tomcat服务器请求数据（mysql,redis），返回给前端
>
> <img src="assets/image-20250116104214772.png" alt="image-20250116104214772" style="zoom:50%;" />
>
> 3.考虑到后续水平扩展问题，tomcat可以集群部署，要考虑到并发，数据共享等等问题
>
> **注意：**
>
> 使用redis存储数据一定要注意设置过期时间，数据数据结构，key的唯一性，类型，存储粒度（不用全部信息）
>
> 
>
> **JSON对象反序列化为Java对象：**
>
> - JSONUtil.toBean(shopJson, Shop.class)
>
> **Java对象序列化为JSON：**
>
> - String shopJSON = JSONUtil.toJsonStr(shop);

## 1.短信登陆

> redis的共享session应用

#### 1.短信登陆的业务流程

<img src="assets/image-20250116143613907.png" alt="image-20250116143613907" style="zoom:50%;" />

注册，登录完之后会再发一个请求，利用session查询当前用户是否存在，然后存在本地（ThreadLocal）

#### 2.发送验证码

1.@slf4j注解：记录日志

2.使用sssion保存，调用第三方api完成发送功能



#### 3.短信验证码登录和注册

1.if反向验证，避免验证码嵌套

2.使用mybatis-plus进行数据库查询操作（srvice接口里面的lambda方法）

```
User user = lambdaQuery().eq(User::getPhone, phone).one();
```



#### 4.检验登录状态

1.拦截器进行登录状态的拦截（不能每一个controller都进行一次拦截），拦截器进行全局拦截（进入controller之前进行拦截），将拦截到的信息保存到ThreadLocal(线程安全，每个线程相互不干扰)

2.session是保存在tomcat服务器中的，为了使得服务器的压力尽可能的小，使用UserDTO对象

<img src="assets/image-20250219171037783.png" alt="image-20250219171037783" style="zoom:50%;" />





#### 5.集群session的共享问题

**问题：**

多台tomcat服务器之间的并不共享session存储空间，切换到不同的服务器数据丢失

**替代方案应该满足的：**

1.数据共享

2.内存存储

3.key,value结构（session和redis都是键值对结构）



#### 6.redis代替session

<img src="assets/image-20250118105948128.png" alt="image-20250118105948128" style="zoom:50%;" />

<img src="assets/image-20250118110028752.png" alt="image-20250118110028752" style="zoom:50%;" />

1.每一个不同的手机号要有自己的key（手机号作为key）

2.redis中存储对象使用Hash，更方便，内存占用也更少

3.验证码的key是手机号，用户信息的key是一个随机的token，会返回给客户端

4.token放在sessionStorage，前端将token作为请求头authorization放入请求中，以后所有请求都有token（所以token不要使用手机号）

<img src="assets/image-20250118110623535.png" alt="image-20250118110623535" style="zoom:50%;" />

5.检验登录状态发现用户一直活跃，要去更新token过期时间

#### 7.基于redis实现共享session登录

1.key前面拼接上业务前缀，区分方便,常量类，设置过期时间

```java
stringRedisTemplate.opsForValue().set(LOGIN_CODE_KEY + phone,code,LOGIN_CODE_TTL, TimeUnit.MINUTES)
```

2.生成的token使用hutool包下的UUID，对象使用Hash，直接使用putAll方法，对象转为HashMap

3.LoginInterceptor类的对象是我们自己手动new出来的（不是通过@Component之类的注解交给了IOC容器进行管理），所以在LoginInterceptor类中的stringRedisTemplate只能通过构造函数注入（不是Spring创建的对象）

4.取出对应key的所有哈希：stringRedisTemplate.opsForHash().entries(token)

5.BeanUtil的copy方法以及转换为Map,以及Map转换回去的方法

6.spring3.0以上的版本，redis的配置： spring:data:redis

<img src="assets/image-20250118205913776.png" alt="image-20250118205913776" style="zoom:50%;" />

7.StringRedisTemplate要求存到redis的数据都是<String,String>类型的，哈希当中的键值对也是，都是字符串

8.拦截器优化：

访问不需要登录就可以看的页面，过期时间不会刷新的，也会导致登录失效，再加拦截器（只要你登陆了，看不用登录的网页，也会给你刷新时间，因为会拦截一切），拦截一切路径，查询用户，保存threadLocal，刷新有效期，第二个拦截器只需要查询threadLocal有没有就行

9.order是拦截器执行顺序，越大执行优先级越低

```java
registry.addInterceptor(new RefreshInterceptor(stringRedisTemplate)).addPathPatterns("/**").order(0);
```



## 2.商户查询缓存

### 1.缓存

1.数据交换的缓冲区，读写性能比较高（例如CPU的缓冲区）

2.web应用的流程：

<img src="assets/image-20250119101323733.png" alt="image-20250119101323733" style="zoom:50%;" />

3.作用：

- 降低后端负载
- 提高读写效率，降低响应时间

  成本：

- 数据一致性成本（缓存与数据库一致性）
- 代码维护成本
- 运维成本



### 2.redis缓存

1.缓存工作模型：

<img src="assets/image-20250119102432455.png" alt="image-20250119102432455" style="zoom:50%;" />

2.

**判断为空：**

- s.isEmpty() 可能会产生空指针异常
- strUtil.isblank(s)

**JSON对象反序列化为Java对象：**

- JSONUtil.toBean(shopJson, Shop.class)

**Java对象序列化为JSON：**

- String shopJSON = JSONUtil.toJsonStr(shop);



3.若Service层继承了IService接口，则在Impl层中，使用IService接口的方法，例如query(),getById()之类的方法不用调用者，直接使用

```java
Shop shop = getById(id);//使用IService接口的方法
```



4.**TODO**

stringRedisTemplate取出数据与存入数据的类型设置

存入：string，string

取出：以string为key，取出的数据不一定是string(看存入的数据是什么，函数是什么)

- List集合：List集合泛型一定要是**String**才能存入

  ```java
  List<String> typeList = new ArrayList<>();
          for (ShopType shoptype : shopTypes) {
              typeList.add(JSONUtil.toJsonStr(shoptype));
          }
          stringRedisTemplate.opsForList().leftPushAll(key,typeList); //存入redis
  ```

  ```java
  List<String> types = stringRedisTemplate.opsForList().range(key, 0, -1);
  List<ShopType> shopTypeList = new ArrayList<>();
              for (String type : types) {
                  shopTypeList.add(JSONUtil.toBean(type,ShopType.class));  //取出进行反序列化
              }
  ```



### 3.缓存更新策略

> 缓存与数据库的一致性

<img src="assets/image-20250120094323983.png" alt="image-20250120094323983" style="zoom:50%;" />



**主动更新：**

<img src="assets/image-20250120094857993.png" alt="image-20250120094857993" style="zoom:50%;" />

三种方法，以第一种为主，调用者自己写代码

- 选择删除缓存，更新缓存会产生无效更新，并且存在较大的线程安全问题
- 确保缓存与数据库操作同时成功或者失败，单体系统是将缓存与数据库操作放在一个**事务**；分布式系统是利用TCC等**分布式事务方案**
- 先删除缓存，后更新数据库
  - 会出现不一致性的情况（线程交叉执行产生）
    - A删除了缓存，还没来得及更新数据库，B线程读取缓存发现被删了，就去读取数据库的旧信息，并将旧数据重新写入缓存
  - 出现可能性会稍微**高**一些
  - 延时双删操作
- 先更新数据库，后删除缓存
  - 会出现不一致性的情况（线程交叉执行产生）
    - 假设有两个线程 A 和 B 同时对同一数据进行更新操作。线程 A 先更新数据库，然后删除缓存；线程 B 后更新数据库，但由于一些原因（如线程调度），线程 B 删除缓存的操作先于线程 A 完成。当线程 A 再去删除缓存时，实际上删除的是已经被线程 B 更新后又删除过的缓存，这可能会导致后续的请求读取到错误的缓存数据，造成数据不一致
    - 删除缓存失败（消息队列进行重试）
  - 出现可能性会更加**低**
  - 利用消息队列进行删除的补偿
  
- <img src="assets/image-20250120100234348.png" alt="image-20250120100234348" style="zoom:50%;" />



### 4.缓存穿透

> 客户端请求的数据在缓存和数据库中都不存在，缓存永远不会生效，请求都会被打到数据库

<img src="assets/image-20250120163440580.png" alt="image-20250120163440580" style="zoom:50%;" />

1.布隆过滤器

二进制位（基于某种哈希算法计算出来的，大概率可以用于判断是否存在）保存在过滤器中，大概率可以拦截，少部分可能还是会穿透



2.缓存空对象

需要设置TTL（时间短的），即使过期，不占用过多的内存空间

短期不一致性：某个id在redis中刚缓存空对象，结果数据库就保存了这个id，这样就会短期不一致



3.查询商铺添加缓存穿透机制

<img src="assets/image-20250120164345439.png" alt="image-20250120164345439" style="zoom:50%;" />

4.总结

<img src="assets/image-20250120170129743.png" alt="image-20250120170129743" style="zoom:50%;" />



### 5.缓存雪崩

> 同一时段大量的缓存key同时失效或者redis服务宕机，导致大量请求到达数据库，压力巨大

<img src="assets/image-20250120171116402.png" alt="image-20250120171116402" style="zoom:50%;" />

key的过期时间添加随机数，不会出现大量key同一时间同时失效的事



### 6.缓存击穿

> 热点key问题，一个被**高并发访问**并且**缓存重建业务较复杂**的key突然失效了，无数的请求访问会在瞬间给数据库带来巨大的冲击

1.解决方案：

- 互斥锁（没有就一直等，等到有为止）

  - 选择了一致性

  <img src="assets/image-20250121114753987.png" alt="image-20250121114753987" style="zoom:50%;" />

- 逻辑过期（没有就用旧数据，新线程去写入缓存新数据）

  - 选择了可用性

  <img src="assets/image-20250121115103408.png" alt="image-20250121115103408" style="zoom:50%;" />



2.对比

<img src="assets/image-20250121115619883.png" alt="image-20250121115619883" style="zoom:50%;" />

 

3.使用互斥锁解决商铺查询问题

<img src="assets/image-20250121120014514.png" alt="image-20250121120014514" style="zoom:50%;" />



（1）sychronized和Lock都是java中的单机锁，拿到锁执行，没拿到锁等待；现在拿到锁和没拿到锁是要自定义的

（2）setnx锁是针对String类型的

（3）自动拆箱前要判断包装类是否为null，否则可能会产生空指针异常

```java
//获取锁
    private boolean tryLock(String key){
        Boolean flag = stringRedisTemplate.opsForValue().setIfAbsent(key,"1",10,TimeUnit.SECONDS);
        return BooleanUtil.isTrue(flag); //手动拆箱，进行判断，将包装类变为基本数据类型

    }
//释放锁
    private void unlock(String key){
        stringRedisTemplate.delete(key);
    }
```

 

4.使用逻辑过期解决缓存击穿

<img src="assets/image-20250122102211036.png" alt="image-20250122102211036" style="zoom:50%;" />

未命中证明不是热点信息，直接返回空就行；命中判断是否逻辑过期

(1)为储存的商户信息添加逻辑过期时间字段，直接在Shop类中添加字段修改了原代码，不太好，新创一个类，包含Shop对象

(2)单元测试完成缓存预热

```java
public void saveShopToRedis(Long id,Long expireSeconds){
        //1.查询出要存的热点数据
        Shop shop = getById(id);

        //2.封装逻辑过期时间
        RedisData redisData = new RedisData();
        redisData.setData(shop);
        redisData.setExpireTime(LocalDateTime.now().plusSeconds(expireSeconds));

        //3.存入redis
        stringRedisTemplate.opsForValue().set(CACHE_SHOP_KEY + id,JSONUtil.toJsonStr(redisData));

    }
```

(3)类经过JSON反序列化出来的结果是JSONObject类型，使用toBean方法

```java
RedisData redisData = JSONUtil.toBean(shopJson, RedisData.class);
        JSONObject data = (JSONObject)redisData.getData();
        Shop shop = JSONUtil.toBean(data, Shop.class);
```

```java
@Data
public class RedisData {
    private LocalDateTime expireTime;
    private Object data;
}
```

(4)创建线程池

```java
private static final ExecutorService CACHE_REBUILD_EXECUTOR = Executors.newFixedThreadPool(10);
```

(4)缓存重建的消耗是线程睡眠



### 7.工具类封装

<img src="assets/image-20250122173054481.png" alt="image-20250122173054481" style="zoom:50%;" />



缓存穿透的工具类封装：

- 泛型两个，一个R类型，一个ID类型，函数最后返回的是R类型
- id不一定是Long类型
- JSON反序列化需要返回的数据的类型，即Class<R> type
- 去数据库查询数据的逻辑工具类无法编写，需要函数式编程，传递函数进来，参数是ID类型，返回值是R类型，apply方法直接使用（具体的逻辑是调用者写的），后面是TTL时间

<img src="assets/image-20250122175351779.png" alt="image-20250122175351779" style="zoom:50%;" />

<img src="assets/image-20250122175450573.png" alt="image-20250122175450573" style="zoom:50%;" />



**原始代码使用工具类改装：**

<img src="assets/image-20250122175719909.png" alt="image-20250122175719909" style="zoom:50%;" />





缓存击穿：

- 逻辑过期
- 函数式编程，查询数据库



<img src="assets/image-20250122180328621.png" alt="image-20250122180328621" style="zoom:50%;" />

<img src="assets/image-20250122180413538.png" alt="image-20250122180413538" style="zoom:50%;" />

return r;



**原始代码使用工具类改装：**

<img src="assets/image-20250122180545324.png" alt="image-20250122180545324" style="zoom:50%;" />

缓存预热：

<img src="assets/image-20250122180822108.png" alt="image-20250122180822108" style="zoom:50%;" />



> **总结：**
>
> - 泛型的使用
> - 函数式编程的使用
> - 抽出逻辑写成工具类，方便调用（这个可以说是开源项目自己的优化）



## 3.优惠券秒杀

### 1.全局唯一ID

> 订单（优惠券）id没有使用自增长ID
>
> - id规律性太明显了
> - 受表单数据量限制（单表无法存储太多的订单量），分散表存储每个表单独计算自增，可能会重复

**全局ID生成器：**

<img src="assets/image-20250123113656863.png" alt="image-20250123113656863" style="zoom:50%;" />

redis自增 incr

为了安全性，我们可以不直接使用Redis自增的数值，而是拼接一些其他信息

具体结构：

<img src="assets/image-20250123114146676.png" alt="image-20250123114146676" style="zoom:50%;" />

（1）redis单个key的自增长是有一个上限的，2的64次方；并且32位序列号可能会超出，所以就算是同一个业务也不能使用同一个key

（2）每一天下的单采用一个相同的key，便于统计这一天或者这一个月或者这一年的订单量

（3）id的拼接不能直接使用String类型，因为需要返回的id是long型，需要位运算进行拼接

（4）使用lambda表达式表示Runnable接口

```java
//线程池
    private ExecutorService es = Executors.newFixedThreadPool(500);

    @Test
    void testIdWorker(){
        Runnable task = () -> {  //线程任务，使用lambda表达式
            for (int i = 0; i < 100; i++) {
                long id = redisIdWorker.nextId("order");
                System.out.println("id = " + id );
            }
        };
        for (int i = 0; i < 300; i++) {
            es.submit(task); //创建30000个id，放到线程池执行
        }

    }
```

（5）线程池是异步的，要计时的话需要等到完全结束再去计时

CountDownLatch类：让一个或多个线程等待其他线程完成操作后再继续执行

任务执行完成后调用`latch.countDown()`将计数器减 1

主线程调用`latch.await()`方法进入等待状态，直到计数器的值变为 0

```java
@Test
    void testIdWorker() throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(300); //300个线程执行
        Runnable task = () -> {
            for (int i = 0; i < 100; i++) {
                long id = redisIdWorker.nextId("order");
                System.out.println("id = " + id );
            }
            countDownLatch.countDown();//任务执行完成后调用latch.countDown()将计数器减 1
        };
        long begin = System.currentTimeMillis();
        for (int i = 0; i < 300; i++) {
            es.submit(task); //创建30000个id，放到线程池执行
        }
        countDownLatch.await();
        long end = System.currentTimeMillis();
        System.out.println("time: " + (end - begin));


    }
```



**总结：**

<img src="assets/image-20250123124445542.png" alt="image-20250123124445542" style="zoom:50%;" />



### 2.优惠券秒杀下单

> 特价券需要秒杀下单
>
> voucher表：存放所有的券
>
> seckill_voucher表：只存放特价券
>
> 下单，扣库存

<img src="assets/image-20250123164917515.png" alt="image-20250123164917515" style="zoom:50%;" />

（1）多张表的操作加上事务注解



### 3.超卖问题

**问题：**

<img src="assets/image-20250124114208906.png" alt="image-20250124114208906" style="zoom:50%;" />

开始时间和结束时间24点不能写成00：00：00这种；23：59：59

<img src="assets/image-20250124114859757.png" alt="image-20250124114859757" style="zoom:50%;" />

多个线程操作共享的资源，并发问题



**解决思想：**

加锁

<img src="assets/image-20250124115707118.png" alt="image-20250124115707118" style="zoom:50%;" />

单机锁：synchronized（悲观）,Lock（悲观）

分布式锁：setnx（乐观），zookeeper

CAS：乐观锁（CPU同步原语，硬件）



（1）乐观锁的版本号法

<img src="assets/image-20250124121012660.png" alt="image-20250124121012660" style="zoom:50%;" />

加上版本号字段，每次修改数据库前查询其值，修改数据库时进行判断值是否改变，并且对其值进行加一操作



（2）CAS法

判断要修改的变量值是否等于旧值，等于旧值则原子操作更新为新值

<img src="assets/image-20250124121734118.png" alt="image-20250124121734118" style="zoom:50%;" />

对乐观锁改造，只要库存大于零就可以（否则成功率太低了），不在判断是否一致。分段锁也可以解决。

数据库的update操作是原子的

真正的秒杀是不能访问数据库的，高并发

**TODO：Mysql锁的相关知识看一下**



### 4.一人一单

> 同一张优惠券，一个用户只能下一单

<img src="assets/image-20250124160702940.png" alt="image-20250124160702940" style="zoom:50%;" />

**读与写产生的多线程并发安全问题**

> 乐观锁用在更新数据（数据之前要有），悲观锁可以用在插入数据的情况下
>
> 利用用户作为锁对象，一个用户一个锁，锁的是当前用户

```java
synchronized (userId.toString().intern()) { //保证用户id的值一样，锁就一样，锁的是当前用户
            //为了防止事务失效，使用代理(事务)，因为之前createVoucherOrder使用的是this调用
            IVoucherOrderService proxy = (IVoucherOrderService)AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId);//先加锁，在执行事务，保证只有在事务提交之后才会释放锁
        }
@Transactional
    public Result createVoucherOrder(Long voucherId){...}
```

- 以 userId转换后的字符串在常量池中的引用作为锁对象，来实现同步机制。也就是说，同一时刻只有一个线程可以获取到这个锁并执行 synchronized 代码块中的内容。这种用法通常用于根据用户 ID 进行细粒度的同步控制。例如，在多线程环境下处理用户相关的业务逻辑时，为了保证同一个用户的操作是线程安全的，可以使用用户 ID 作为锁对象。



（1）动态代理（事务是否生效）

```xml
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
</dependency>
```

动态代理的模式



（2）释放锁的时机

先加锁，再去执行事务，提交事务，在解锁



**问题：**

集群部署意味着有着多台全新的Tomcat，多个全新的JVM，有自己全新的锁监视器，在当前JVM的内部可以监视线程实现互斥，但多个无法监视

`让多个JVM共用同一把锁`

<img src="assets/image-20250124210209553.png" alt="image-20250124210209553" style="zoom:50%;" />



## 4.分布式锁

> 满足分布式系统或者集群模式下多进程可见并且互斥的锁
>
> 高可用，高性能，安全性

<img src="assets/image-20250124211616224.png" alt="image-20250124211616224" style="zoom:50%;" />

基于Redis分布式锁的流程

<img src="assets/image-20250124214802138.png" alt="image-20250124214802138" style="zoom:50%;" />



### 1.使用redis实现分布式锁的功能



```java
public boolean tryLock(long timeoutSec) {
        //锁的value是当前线程的标识
        long id = Thread.currentThread().getId();
        Boolean success = stringRedisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, id + "", timeoutSec, TimeUnit.SECONDS);
//        return Boolean.TRUE.equals(success);
        return BooleanUtil.isTrue(success);
    }

    @Override
    public void unlock() {

        stringRedisTemplate.delete(KEY_PREFIX + name);
    }
```

1.自动拆箱的判断（防止空）

2.针对之前的一人一单，sychronized锁无法保证（单机锁），使用分布式锁

```java
Long userId = UserHolder.getUser().getId();
        //使用分布式锁
        //创建锁对象
        SimpleRedisLock simpleRedisLock = new SimpleRedisLock("order:" + userId,stringRedisTemplate);
        //获取锁
        boolean isLock = simpleRedisLock.tryLock(1200);
        //判断是否成功
        if(!isLock){
            //获取锁失败
            return Result.fail("不允许重复下单");
        }
        try{
            //为了防止事务失效，使用代理(事务)，因为之前createVoucherOrder使用的是this调用
            IVoucherOrderService proxy = (IVoucherOrderService)AopContext.currentProxy();
            return proxy.createVoucherOrder(voucherId);//先加锁，在执行事务，保证只有在事务提交之后才会释放锁
        }finally {
            //手动释放，不再是自动释放
            simpleRedisLock.unlock();
        }
    }
```



### 2.基于Redis的分布式锁

> 极端情况下多个线程都可以获取锁

**原因：**

1.业务时间不确定

2.释放了线程的锁

<img src="assets/image-20250215210110013.png" alt="image-20250215210110013" style="zoom: 50%;" />



**改进：**

1.获取锁的时候存入线程标识（不能使用线程的id，集群模式下线程的id可能会重复，使用UUID）

2.释放锁时先获取锁中的线程标识



通过添加线程标识的方式解决了分布式锁的误删问题，让分布式锁更加健壮

**问题：**判断锁标识和释放锁中间JVM垃圾回收阻塞，导致判断成功，但是没有释放锁，阻塞时间过长，锁自动释放，线程二恰巧此时获取了锁，去执行业务，但线程一的阻塞结束了，直接释放锁

<img src="assets/image-20250215214540641.png" alt="image-20250215214540641" style="zoom:50%;" />

所以必须判断锁标识的动作和释放锁是原子的



**改进：**

1.可以使用redis乐观锁+事务进行限制（比较麻烦）

2.Lua脚本，一个脚本中编写多条redis命令，一起执行



Lua脚本使用语法：

<img src="assets/image-20250216095910524.png" alt="image-20250216095910524" style="zoom:67%;" />

- 参数有两种，一种是key类型的参数，一种是其他参数
- Lua语言数组角标是从1开始的

<img src="assets/image-20250216100458893.png" alt="image-20250216100458893" style="zoom: 67%;" />



分布式锁Lua脚本：（原子性）

<img src="assets/image-20250216102114827.png" alt="image-20250216102114827" style="zoom:67%;" />

```lua
--比较线程标识与锁中的标识是否一致(锁的KEY和线程标识)
if(redis.call('get',KEYS[1] == ARGV[1]) then
    --释放锁
    return redis.call('del',KEYS[1])
end
return 0
```



java代码调用Lua脚本：（与EVAL命令的对比）

<img src="assets/image-20250216102444648.png" alt="image-20250216102444648" style="zoom:50%;" />

```java
//脚本提前准备好,Long是返回值 DefaultRedisScript是实现类
    private static final DefaultRedisScript<Long> UNLOCK_SCRIPT;
    static {
        UNLOCK_SCRIPT = new DefaultRedisScript<>();
        UNLOCK_SCRIPT.setLocation(new ClassPathResource("unlock.lua"));
        UNLOCK_SCRIPT.setResultType(Long.class);
    }

//调用Lua脚本去执行判断以及释放锁的操作(原子性)
    @Override
    public void unlock() {
        stringRedisTemplate.execute(
                UNLOCK_SCRIPT,
                Collections.singletonList(KEY_PREFIX + name),
                ID_PREFIX + Thread.currentThread().getId());
    }
```



**以上基于Lua脚本实现的redis分布式锁已经比较完善了**

<img src="assets/image-20250216113117780.png" alt="image-20250216113117780" style="zoom:50%;" />

还存在一些问题：

<img src="assets/image-20250216141224625.png" alt="image-20250216141224625" style="zoom:50%;" />



### 3.Redisson

对上述问题的优化以及拓展，提供了许多分布式的服务

#### 1.快速入门

（1）引入依赖

（2）配置Redisson客户端

（3）使用

<img src="assets/image-20250216141743348.png" alt="image-20250216141743348" style="zoom:50%;" />

<img src="assets/image-20250216142050803.png" alt="image-20250216142050803" style="zoom:50%;" />



#### 2.可重入实现

**问题：**

一个线程中method1先获取了锁，然后调用method2，但在method2中也会去获取锁，这时method2会获取锁失败，线程卡死

**解决方法：**

获取锁发现以及被获取了的时候，会自动去判断是不是自己获取的，如果是的话，可以获取，会用计数器去记录，释放锁的时候，次数会减一

使用Hash数据结果来存储可重入锁的结果<img src="assets/image-20250216150022209.png" alt="image-20250216150022209" style="zoom:50%;" />，当value值为0的时候，证明以及到了最后一步，这个时候就可以释放锁了（之前的释放只是次数减一）

<img src="assets/image-20250216151109962.png" alt="image-20250216151109962" style="zoom: 67%;" />

使用Lua脚本去实现上述流程，确保原子性

- 获取锁的Lua脚本

<img src="assets/image-20250216151432879.png" alt="image-20250216151432879" style="zoom:50%;" />

- 释放锁的Lua脚本

<img src="assets/image-20250216151531256.png" alt="image-20250216151531256" style="zoom:50%;" />

跟踪redisson的Lua脚本：

<img src="assets/image-20250216151843175.png" alt="image-20250216151843175" style="zoom:50%;" />

<img src="assets/image-20250216151858877.png" alt="image-20250216151858877" style="zoom:50%;" />

#### 3.重试机制实现

1.默认锁超时时间是30s

2.获取锁失败后，判断还存在等待时间的话，会订阅释放锁的信号（publish出来的）  释放了，再去尝试，对CPU友好





#### 4.解决超时释放的问题

> 要确保锁是业务执行完之后去释放的，而不是超时使得锁释放

当使用 Redisson 获取锁时，如果没有指定锁的过期时间，Redisson 会默认给锁设置一个过期时间（默认 30 秒），并且会启动一个定时任务（看门狗）来不断地为锁续期，只要持有锁的客户端没有释放锁，看门狗就会一直运行，不断延长锁的过期时间，避免锁因超时而自动释放。

- 当手动指定锁的过期时间时，看门狗机制将不会生效，需要自己确保业务逻辑在锁的过期时间内执行完毕。例如：`lock.lock(10, TimeUnit.SECONDS);` 表示手动指定锁的过期时间为 10 秒。
- 看门狗的默认续期时间为 30 秒，可以通过 `Config` 类的 `lockWatchdogTimeout` 属性进行修改。例如：`config.setLockWatchdogTimeout(60000);` 表示将看门狗的续期时间修改为 60 秒。



### 5.秒杀优化

**问题：**

之前的秒杀下单业务顺序执行，且多次查询数据库，效率比较低

**优化：**

> 异步下单，减少业务耗时
>
> 将秒杀业务和下单业务的耦合解开

- 将任务分成两个部分，分开完成
- 在redis中完成资格的校验操作，将信息保存到阻塞队列（在redis中对库存信息以及对优惠券与购买过的用户信息进行缓存，选择String:String ，String:Set的结构）
- 异步开启一个线程读取队列中的信息，完成下单操作（将同步的数据库写操作转变为异步的操作，大大减轻了数据库的压力，）

<img src="assets/image-20250216163103832.png" alt="image-20250216163103832" style="zoom: 50%;" />

<img src="assets/image-20250216164441787.png" alt="image-20250216164441787" style="zoom:50%;" />

- 左边这个扣减库存是在redis中扣减，不涉及数据库操作
- 返回订单id后用户就已经可以进行后续的操作



**步骤：**

<img src="assets/image-20250216164738007.png" alt="image-20250216164738007" style="zoom:50%;" />

- 新增优惠券的时候就将库存信息保存到reids中，优惠券订单信息在秒杀过程中添加
- Lua脚本中完成资格的校验

```lua
--2.订单key
local orderKey = 'seckill:order:' .. voucherId;

--脚本业务
--1.判断库存
if(tonumber(redis.call('get',stockKey)) <= 0) then
    --库存不足
    return 1
end

--2.判断用户是否下单 SISMEMBER orderKey userId
if(redis.call('SISMEMBER',orderKey,userId) == 1) then
    --存在，重复下单
    return 2
end

--3.扣库存，保存用户 incrby stockKey -1;'SADD',orderKey,userId
redis.call('incrby',stockKey,-1)
redis.call('SADD',orderKey,userId)
return 0
```

- 如果抢购成功，将下单信息封装到阻塞队列中，实现异步下单，不会影响到业务的执行时间

```java
//阻塞队列，当一个线程获取队列中的元素时，若队列中没有该元素，则线程会阻塞，直到有了之后被唤醒
    private BlockingQueue<VoucherOrder> orderTasks = new ArrayBlockingQueue<>(1024 * 1024);
```

- 线程池，线程任务
  - 线程池采用单线程池，只有一个线程
  - 线程任务有两种方式，一种是创建Thread的子类并重写run方法；一种是实现Runnable接口，重写run方法；或者使用lambda表达式

```java
Runnable task = () -> {  //线程任务，使用lambda表达式
            for (int i = 0; i < 100; i++) {
                long id = redisIdWorker.nextId("order");
                System.out.println("id = " + id );
            }
        };
        for (int i = 0; i < 300; i++) {
            es.submit(task); //创建30000个id，放到线程池执行
        }

//线程任务
    private class VoucherOrderHandler implements Runnable{

        @Override
        public void run() {
            
        }
    }
//要执行，需要  线程池.submit(线程任务类对象)
```



基于阻塞队列的异步秒杀问题：

- 阻塞队列占用JVM内存，内存有限
- 数据安全，信息在内存中，服务宕机信息丢失或者redis内存更新了，但故障，数据库没更新，不一致问题



## 5.消息队列

<img src="assets/image-20250217185527302.png" alt="image-20250217185527302" style="zoom:50%;" />

消息队列相比起阻塞队列：

- 是独立于JVM的服务，不占用JVM的内存
- 消息队列有数据持久化操作，同时会让消费者确认数据是否被消费（确认安全）



小型企业对消息队列的要求不高的话，可以直接使用redis的消息队列

<img src="assets/image-20250217190059067.png" alt="image-20250217190059067" style="zoom:50%;" />

### 1.使用List模拟消息队列

> Redis的List结构是双向链表

<img src="assets/image-20250217190408437.png" alt="image-20250217190408437" style="zoom:50%;" />

取出消息没处理然后挂了，就会导致消息丢失

**优缺点：**

<img src="assets/image-20250217190757843.png" alt="image-20250217190757843" style="zoom:50%;" />



### 2.基于PubSub的消息队列

> redis2.0之后引入的，可以支持多个消费者

<img src="assets/image-20250217191348789.png" alt="image-20250217191348789" style="zoom:50%;" />

- 订阅命令天生就是阻塞式的

- 消费者有一个缓冲区用来存储消息，若缓冲区满了消息就会丢失

- 可靠性不是很高



**优缺点：**

<img src="assets/image-20250217191946796.png" alt="image-20250217191946796" style="zoom:50%;" />



### 3.基于Stream的消息队列

> redis5.0之后引入的一种新的数据类型（支持数据的持久化的）

**添加消息：**

<img src="assets/image-20250217192555287.png" alt="image-20250217192555287" style="zoom:50%;" />

**读取消息：**

<img src="assets/image-20250217192831525.png" alt="image-20250217192831525" style="zoom:50%;" />

消息读取完之后不会删除

**问题：**

<img src="assets/image-20250217193318054.png" alt="image-20250217193318054" style="zoom:50%;" />



**优缺点：**

<img src="assets/image-20250217193612661.png" alt="image-20250217193612661" style="zoom:50%;" />



### 4.基于Stream的消息队列（消费者组）

<img src="assets/image-20250217201240451.png" alt="image-20250217201240451" style="zoom:50%;" />

<img src="assets/image-20250217201438982.png" alt="image-20250217201438982" style="zoom:50%;" />

<img src="assets/image-20250217201901726.png" alt="image-20250217201901726" style="zoom:50%;" />

正常读取消息ID是使用“>”，但是一旦出现了异常（即没有确认），就会进入Pending，捕捉到这个异常之后，ID不能在使用“>”，换成其他去继续处理

**确认命令：**

<img src="assets/image-20250217202229232.png" alt="image-20250217202229232" style="zoom:50%;" />

如果获取了消息，但没有执行确认XACK，会进入Pending队列,查看pending队列中的消息<img src="assets/image-20250217202716574.png" alt="image-20250217202716574" style="zoom: 33%;" />，



**消费者监听消息的基本思路：**

<img src="assets/image-20250217203535377.png" alt="image-20250217203535377" style="zoom: 50%;" />



**优缺点：**

<img src="assets/image-20250217203714421.png" alt="image-20250217203714421" style="zoom:50%;" />

取出消息之后没有进行处理就是消息丢失



### 消息队列总结

<img src="assets/image-20250217204051709.png" alt="image-20250217204051709" style="zoom:50%;" />

还有许多问题，只支持消费者的确认模式，不支持生产者的等等问题



### 基于消息队列实现异步秒杀下单

<img src="assets/image-20250217205206952.png" alt="image-20250217205206952" style="zoom:50%;" />

- redis命令行去创建消息队列
- 修改Lua脚本，增加一个参数orderId,在Lua脚本最后增加将参数发送到消息队列的操作
- <img src="assets/image-20250217205944183.png" alt="image-20250217205944183" style="zoom:50%;" />
- 修改代码，执行完Lua脚本后业务结束
- 开启新线程去获取信息，完成下单
- ![image-20250217211511414](assets/image-20250217211511414.png)

- ![image-20250217211703389](assets/image-20250217211703389.png)

- ![image-20250217211902580](assets/image-20250217211902580.png)

具体下单步骤：

1.从消息队列中读取消息

2.不存在，继续读取；存在，解析消息队列中的订单

3.下单

4.ACK确认

5.未确认的异常订单被捕获后读取出来（0），再次解析，进行处理，再次确认



redis在秒杀场景下的应用：

<img src="assets/image-20250217212729916.png" alt="image-20250217212729916" style="zoom:50%;" />



## 6.达人探店

### 1.发布探店笔记

- 上传图片的接口和发布笔记的接口不是一个，上传图片是一个独立的功能，会返回图片的地址，点击发布会将图片地址发布
- 为了简化操作，将文件直接保存到nginx服务器上（本地中）正常应该上传到文件服务器上



### 2.完善点赞的功能

<img src="assets/image-20250218135345984.png" alt="image-20250218135345984" style="zoom:50%;" />

- 利用数据库去记录笔记的点赞Id对数据库压力太大，太重

  ```java
      /**
       * 点赞功能
       * @param id
       * @return
       */
      @Override
      public Result likeBlog(Long id) {
          //1.获取登录用户
          Long userId = UserHolder.getUser().getId();
          //2.判断是否点赞
          String key = "blog:liked:" + id;
          Boolean isMember = stringRedisTemplate.opsForSet().isMember(key, userId.toString());
          if(BooleanUtil.isFalse(isMember)){
              //3.若未点赞，数据库点赞数加一，更新redis的set集合
              boolean isSuccess = update().setSql("liked = liked + 1").eq("id", id).update();
              if(isSuccess){
                  stringRedisTemplate.opsForSet().add(key,userId.toString()); 
              }
  
          }else {
              //4.若已经点赞，数据库点赞数减一，set集合中移除
              boolean isSuccess = update().setSql("liked = liked - 1").eq("id", id).update();
              if(isSuccess){
                  stringRedisTemplate.opsForZSet().remove(key,userId.toString());
              }
  
          }
          return Result.ok();
      }
  ```



### 3.点赞排行榜

<img src="assets/image-20250218153214584.png" alt="image-20250218153214584" style="zoom:50%;" />

- 数据结构使用SortedSet,命令使用ZADD,ZSCORE,ZRANGE，使用时间戳作为分数，按照分数进行排序，点赞列表展示前五位点赞的用户

- ZSet默认排序是按照分数升序排序
- 流的map方法：接收一个函数作为参数，函数会被应用到流中每个元素上，并将操作后的结果生成一个新的流
- 首页用户未登录，无需查询是否点赞



**问题：**

1.数据库用IN语句不一定按照传入的参数的顺序展示数据,需要加上order by field(id,5,1)，展示的顺序才会是5，1

<img src="assets/image-20250218163354914.png" alt="image-20250218163354914" style="zoom:33%;" />

需要加上orderBy语句

```java
public Result queryBlogLikes(Long id) {

        //1.按分数查询ZSet成员（点赞用户）前五名
        String key = BLOG_LIKED_KEY + id;
        Set<String> top5 = stringRedisTemplate.opsForZSet().range(key, 0, 4);
        if(top5.isEmpty() || top5 == null){
            return Result.ok(Collections.emptyList());
        }
        List<Long> ids = top5.stream().map(Long::valueOf).collect(Collectors.toList());//map是进行加工(里面是操作)，并且返回新流
        //2.获取用户
//        List<User> users = userService.listByIds(ids);
//        List<UserDTO> userDTOS = new ArrayList<>();
//        BeanUtil.copyProperties(users,userDTOS);
        String idStr = StrUtil.join(",",ids); //拼接的用户id顺序

        //拼接上orderBY语句
        List<UserDTO> userDTOS = userService.query()
                .in("id",ids).last("order by field(id," + idStr + ")" ).list()
                .stream().map(user -> BeanUtil.copyProperties(user, UserDTO.class))
                .collect(Collectors.toList());

        //3.返回
        return Result.ok(userDTOS);
    }
```



## 7.好友关注

实现关注和取关功能

- 关注和取关接口
- 判断是否关注的接口

MP的使用小案例：

```java
//在实现方式上，第一个语句显式地创建了 QueryWrapper 对象，而第二个语句使用了 query() 快捷方法来创建查询条件构造器，代码更加简洁
remove(new QueryWrapper<Follow>().eq("user_id", userId).eq("follow_user_id", followUserId));

remove(query().eq("user_id", userId).eq("follow_user_id", followUserId));
```

query()方法及其后面那一堆都只是条件构造方法，构造出查询条件QueryChainWrapper,需要调用终结方法运用这个查询条件，例如list(),count()等等

像sava(),remove()可以直接使用，只需构造参数，传参就行



**共同关注：**

利用set集合的求交集的命令，sinter

```java
/**
     * 共同关注功能
     * @param id
     * @return
     */
    @Override
    public Result followCommons(Long id) {
        Long userId = UserHolder.getUser().getId();
        String key1 = "follows:" + userId;
        String key2 = "follows:" + id;
        Set<String> commonFollows = stringRedisTemplate.opsForSet().intersect(key1,key2);
        if(commonFollows == null || commonFollows.isEmpty()){
            //无交集
            return Result.ok();
        }
        //解析id集合，转换为Long型
        List<Long> ids = commonFollows.stream().map(Long::valueOf).collect(Collectors.toList());
        //查询用户
        List<UserDTO> userDTOS = userService.listByIds(ids)
                .stream().map(user -> BeanUtil.copyProperties(user, UserDTO.class)).collect(Collectors.toList());
        return Result.ok(userDTOS);
    }
```

- BeanUtil.copyProperties()的返回值是UserDTO集合

--stream流与MP的使用--



1.项目问答 + 根据简历总结一下，去背八股--> JAVA GUIDE后续

2.代码随想录刷算法（上午）

