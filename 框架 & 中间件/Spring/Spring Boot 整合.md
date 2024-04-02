# SpringBoot  整合

[TOC]



## JUnit

JUnit 是一个 Java 单元测试框架。JUnit 5 由 3 个模块构成，分别是

- JUnit Platform ：基于 JVM 上启动测试框架的基础
- JUnit Jupiter ：JUnit 5 的核心，提供 JUnit 5 的新的编程模型，内部包含一个测试引擎，该测试引擎会基于 JUnit Platform 运行；
- JUnit Vintage ：兼容 JUnit 4 、JUnit 3 支持的测试引擎



|              | 意义                             |
| ------------ | -------------------------------- |
| @Test        | 标注一个测试方法                 |
| @BeforeEach  | 在每个测试方法前执行             |
| @AfterEach   | 在每个测试方法后执行             |
| @BeforeAll   | 在当前类中的所有测试方法之前执行 |
| @AfterAll    | 在当前类中的所有测试方法之后执行 |
| @Disabled    | 禁用测试方法/类                  |
| @Tag         | 标记和过滤                       |
| @TestFactory | 声明测试工厂进行动态测试         |
| @Nested      | 嵌套测试                         |
| @ExtendWith  | 注册自定义扩展                   |



Maven依赖

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
~~~

测试类的编写，需要`@SpringBootTest`注解，保证驱动SpringBoot

~~~java
@SpringBootTest
class SpringBootDemoApplicationTests {
    @Test
    void contextLoads() {
    }
}
~~~



断言的使用：

~~~java
// 数值断言
int num = 3 + 5;
Assertions.assertEquals(num, 8);

// Boolean 断言
int num = 3 + 5;
Assumptions.assumeTrue(num < 10);

// 浮点断言，可以指定浮动值
double result = 10.0 / 3;
Assertions.assertEquals(result, 3, 0.5);

// 可以自定义错误提示信息
Assertions.assertEquals(result, 3, 0.2, "计算数值偏差较大！");

// 断言两个对象是否是同一个
Object o1 = new Object();
Object o2 = o1;
Object o3 = new Object();
Assertions.assertSame(o1, o2);
Assertions.assertSame(o1, o3);

// 断言两个数组的元素是否完全相同
Assertions.assertArrayEquals(arr1, arr3);

// 断言是否抛出异常
Assertions.assertThrows(ArithmeticException.class, () -> {
    int i = 1 / 0;
});

// 断言是否超时
Assertions.assertTimeout(Duration.ofMillis(500), () -> {
    System.out.println("testTimeout run ......");
    TimeUnit.SECONDS.sleep(1);
    System.out.println("testTimeout finished ......");
});

// 断言强制失败
Assertions.fail();



~~~

组合条件断言，要求这些断言必须同时全部通过

~~~java

Assertions.assertAll(
    () -> {
        int num = 3 + 5;
        Assertions.assertEquals(num, 8);
    },
    () -> {
        String[] arr1 = {"aa", "bb"};
        String[] arr2 = {"bb", "aa"};
        Assertions.assertArrayEquals(arr1, arr2);
    }
);

~~~



嵌套测试：

~~~java
public class TestingAStackDemo {
    Stack<Object> stack;
    
    @Test
    void isInstantiatedWithNew() {
        new Stack<>();
    }
    
    // 在单元测试类中，可以编写内部测试类，这要求标注 @Nested 注解
    // 内部的单元测试类可以直接使用外部的成员属性
    @Nested
    class WhenNew {
        @BeforeEach
        void createNewStack() {
            stack = new Stack<>();
        }
        
        @Test
        void isEmpty() {
            assertTrue(stack.isEmpty());
        }
    }
}
~~~





参数化测试：

- 手动指明需要测试的值：
  ~~~java
  @ParameterizedTest
  @ValueSource(strings = {"aa", "bb", "cc"})
  // 声明了 3 个需要测试的值
  public void testSimpleParameterized(String value) throws Exception {
      System.out.println(value);
      Assertions.assertTrue(value.length() < 3);
  }
  ~~~

-  通过静态方法来提供测试数据

  ~~~java
  @ParameterizedTest
  @MethodSource("dataProvider")
  public void testDataStreamParameterized(Integer value) throws Exception {
      System.out.println(value);
      Assertions.assertTrue(value < 10);
  }
  
  private static Stream<Integer> dataProvider() {
      return Stream.of(1, 2, 3, 4, 5);
  }
  ~~~

## Mybatis

~~~xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.2.0</version>
</dependency>
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
</dependency>
~~~



为 Mapper 打上注解 @Mapper

~~~java
@Mapper
@Repository
public interface UserMapper {
    void save(User user);
    List<User> findAll();
}
~~~

直接通过依赖注入来获取：

~~~java
@Service
public class UserService {
    @Autowired
    private UserMapper userMapper;
    
    public void test() {
        User user = new User();
        user.setName("test mybatis");
        user.setTel("1234567");
        userMapper.save(user);
        
        List<User> userList = userMapper.findAll();
        userList.forEach(System.out::println);
    }
}
~~~

## Redis

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
~~~

在默认情况下 SpringDataRedis 会实用 Lettuce 作为底层与 Redis 交互的基础 API 实现。可以手动排除 Lettuce 的依赖，并引入 Jedis

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <exclusions>
        <exclusion>
            <groupId>io.lettuce</groupId>
            <artifactId>lettuce-core</artifactId>
        </exclusion>
    </exclusions>
</dependency>
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
</dependency>
~~~

SpringBootTest 提供了针对 SpringDataRedis 的场景测试器，所以可以直接用 `@DataRedisTest` 而不是 `@SpringBootTest`

~~~java
@DataRedisTest
public class RedisTest {
    @Autowired
    private RedisTemplate<Object, Object> redisTemplate;
    
    @Autowired
    private StringRedisTemplate stringRedisTemplate;
}
~~~



key-value 的操作：

~~~java
ValueOperations<Object, Object> valueOperations = redisTemplate.opsForValue();

// 存入指定值
valueOperations.set("abc", "def");
valueOperations.setIfAbsent("qaz", 123);
valueOperations.setIfAbsent("qaz", 456); // 这次不会生效

 // 存入指定值，并设置过期时间为1分钟（三种方式均可）
valueOperations.set("qqq", 333, 1, TimeUnit.MINUTES);
valueOperations.set("aaa", 444, TimeUnit.MINUTES.toMillis(1));
valueOperations.set("zzz", 555, Duration.ofMinutes(1));

// 取出指定值
valueOperations.get("abc")
valueOperations.get("qaz")
    
// 获取指定key的过期时间
redisTemplate.getExpire("qqq")
    
// 删除指定值，多个值
redisTemplate.delete("abc");
redisTemplate.delete(Arrays.asList("qqq", "aaa", "zzz"));

// 检查某个key是否存在
redisTemplate.hasKey("qaz")
~~~



List列表的操作

~~~java
ListOperations<Object, Object> listOperations = redisTemplate.opsForList();

// 向一个列表的左侧压入数据
listOperations.leftPush("leftList", "aaa");
// 一次性向左侧压入多个数据
listOperations.leftPushAll("leftList", 123, 456, 789);
// 获取列表中的数据
List<Object> leftList = listOperations.range("leftList", 0, 100);

// 替换列表中的一个数据
listOperations.set("leftList", 0, 999);

// 根据索引获取数据
listOperations.index("leftList", 1)
    
// 获取数据对应的索引
listOperations.indexOf("leftList", 456) 
    
// 向左弹出一个数据
listOperations.leftPop("leftList")

// 删除指定元素
// 如果第二个参数不为0，则表示将要删除的元素所在的位置，例如传入-1表示删除从列表末尾开始的第一个值为111的元素，并传入1则表示删除从列表开头计数的第一个值为111的元素。 为0，表示删除所有
listOperations.remove("rightList", 0, 111);
~~~



对于Set的操作、Hash的操作、ZSet等操作，回来自行查阅补充

## MongoDB

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
~~~

~~~yaml
spring.data.mongodb.host= 192.168.217.128
spring.data.mongodb.port= 27017
spring.data.mongodb.database= spring-data
spring.data.mogodb.uri= mongodb://180.76.159.126:27017,180.76.159.126:27018,180.76.159.126:27019/articledb connect=replicaSet&slaveOk=true&replicaSet=myrs
~~~

uri 必须包括副本集中所有的主机，包括主节点、副本节点、仲裁节点。

可以使用 SpringDataMongoDB 来操纵 MongoDB，但这里我们介绍 MongoTemplate。

插入数据

~~~java
@Autowired
MongoTemplate mongoTemplate;

public void foo() {
    User user = new User();
    mongoTemplate.save(user);
    
   
    // 返回一个对象，可以获取自增主键
    User userWithId = mongoTemplate.insert(User.class).one(user);
    // 这里 user 与 userWithId 是同一个对象
}
~~~

更新数据：

~~~java
User replace = new User();
    replace.setAge(100);

// 先通过 matching 指定要更新的对象，然后调用 replaceWith() 开始替换
mongoTemplate.update(User.class).matching(Criteria.where("name").is("mongo"))
            .replaceWith(replace).as(User.class).findAndReplaceValue();
~~~

这种更新是覆盖修改，我们通过以下方式实现局部修改：

~~~java
mongoTemplate.updateFirst(
    Query.query(Criteria.where("name").is("mongo")), 
    Update.update("age", 200), 
    User.class);
~~~



删除数据：

~~~java
mongoTemplate.remove(Query.query(Criteria.where("age").gt(30)), User.class);
~~~



查询数据：

~~~java
mongoTemplate.find(Query.query(Criteria.where("age").lt(18)), User.class)
    mongoTemplate.findById("62f0f24e618cfd7eb146b14c", User.class)
~~~

## ElasticSearch

maven依赖：

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
</dependency>
~~~



~~~java
@Autowired
private ElasticsearchRestTemplate elasticsearchRestTemplate;

@Test
public void testTemplate() throws Exception {
	SearchHits<GraphicsCard> hits = elasticsearchRestTemplate.search(
        new CriteriaQuery(Criteria
            .where("name")
            .contains("ROG")), 
        GraphicsCard.class);
}
~~~

## Cache

 Java 中的缓存模型规范：**JSR-107** 规范

- CachingProvider：它可以生产多个 `CacheManager`

- CacheManager：创建和管理 `Cache` 对象

  ~~~java
  public interface CacheManager {
      // 获取特定的 Cache
      Cache getCache(String name);
      // 获取所有的 Cache 名
      Collection<String> getCacheNames();
  }
  ~~~

- Cache 是具体的缓存器，它的设计非常类似于 `Map` 

  ~~~java
  public interface Cache {
  	String getName();
  	@Nullable
  	ValueWrapper get(Object key);
  	@Nullable
  	<T> T get(Object key, @Nullable Class<T> type);
  	@Nullable
  	<T> T get(Object key, Callable<T> valueLoader);
  	void put(Object key, @Nullable Object value);
  }
  ~~~

- Entry

- Expiry

Maven依赖：

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
~~~



使用示例：

~~~java
 @Autowired
private CacheManager cacheManager;

 public User get(Integer id) {
     // 1. 通过 CacheManager 拿到名为 user 的缓存对象 Cache
     Cache cache = cacheManager.getCache("user");
     
     // 2. 从 Cache 中尝试获取一个指定 id 的 User 类型的对象
     User user = cache.get(id, User.class);
     
     // 3. 如果对象数据存在，则直接返回
     if (user != null) {
         return user;
     }
     
     // 4. 如果数据不存在，则需要查询数据库，并将查询的结果放入Cache中
     User userFromDatabase = userMapper.get(id);
     cache.put(id, userFromDatabase);
     return userFromDatabase;
 }
~~~

要在主启动类上标注 `@EnableCaching` 注解

~~~java
@EnableCaching
@SpringBootApplication
public class SpringBootCacheApplication
~~~



可以在 Bean 初始化时就获取 `Cache` 对象，减少一部分重复代码。

~~~java
@Service
public class CachedUserService implements InitializingBean {;
    @Autowired
    private CacheManager cacheManager;
    private Cache cache;
    
    @Override
    public void afterPropertiesSet() throws Exception {
        this.cache = cacheManager.getCache("user");
    }

~~~



我们可以通过 @Cacheable 来简单快速地使用缓存：

~~~Java
@Service
public class AnnotationUserService {
    @Cacheable("user.get")
    // value代表缓存的名字，这个参数必填
    public User get(Integer id) {
        // 这里缓存映射关系是 id → User
        return ...;
    }
}
~~~

~~~java
@Autowired
private AnnotationUserService annotationUserService;

@Test
public void test3() throws Exception {
    User user1 = annotationUserService.get(1);
    User user2 = annotationUserService.get(1);
    System.out.println(user1 == user2);			// true
}
~~~

这里有两点需要着重说明一下：

- 这套缓存抽象背后是通过 AOP 来实现的，所以如果执行缓存操作，那么必须访问代理后的对象（依赖注入帮我们处理好了）
- 只有那些**可幂等操作**的方法才适用于这套抽象，因为必须要保证相同的参数拥有一样的返回值。

| Spring 注解    | 说明                                                         |
| -------------- | ------------------------------------------------------------ |
| `@Cacheable`   | 从缓存中获取对应的缓存值，没有的话就执行方法并缓存，然后返回。其中 `sync` 如果为 `true`，在调用方法时会锁住缓存，相同的参数只有一个线程会计算，其他线程等待结果 |
| `@CachePut`    | 直接更新缓存                                                 |
| `@CacheEvict`  | 清除缓存，其中的 `allEntries` 如果设置为 `true`，则清除指定缓存 |
| `@Caching`     | 可以用来组合多个缓存抽象的注解，比如两个 `@CacheEvict`       |
| `@CacheConfig` | 添加在类上，为这个类里的缓存抽象注解提供公共配置，例如统一的 `cacheNames` 和 `cacheManager` |

这些注解中有很多一样的属性（除了 `@Caching`），具体如下

- `cacheNames`，标识一个缓存
- `key`，计算缓存Key名的 SpEL 表达式
- `keyGenerator`，自定义的 `KeyGenerator` Bean 名称，用来生成缓存键名，与 `key` 属性互斥。
- `cacheManager`，缓存管理器的 Bean 名称，负责管理实际的缓存
- `cacheResolver`，缓存解析器的 Bean 名称，与 `cacheManager` 属性互斥
- `condition`，在调用方法**之前**判断条件，决定是否缓存
- `unless`：在调用方法**之后**判断条件，决定是否**不**缓存。



key 属性值的说明

- `#root.methodName` 和 `#root.method.name` ：以方法名作为 key。

- `#root.targetClass`：以类名作为 key。

- `#root.args[0]`：以第一个参数名作为 key。如果只有一个参数，那么 `#id` 与 `#root.args[0]`作用一样

- `#root.caches[1]`：以 value[] 中的第一个参数作为 key

- 自定义 key 的生成：

  ~~~java
  @Component
  public class UserKeyGenerator implements KeyGenerator {
      @Override
      public Object generate(Object target, Method method, Object... params) {
          return method.getName() + params[0];
      }
  }
  ~~~

~~~java
// 此处将方法名、name 参数与 size 参数用“-”拼接在一起作为缓存的键名。
@Cacheable(key = "#root.methodName + '-' + #name + '-' + #size")
public Optional<MenuItem> getByNameAndSize(String name, Size size) {
    return menuRepository.findByNameAndSize(name, size);
}
~~~



删除缓存：

```java
@CacheEvict(cacheNames = "hello", key = "#id") 
public String delete(String id) {
    // 删除key为id的缓存
    return "删除成功";
}
```

修改缓存：

```java
@CachePut(cacheNames = "hello", key = "#id") 
public String update(String id) {
    return "修改后的缓存数据";
}
```



Spring 缓存抽象的默认实现为`ConcurrentHashMap`，其实 Spring 的缓存抽象能够支持多种不同的后端缓存实现，通过ChacheMananger来指定：

| 实现类                      | 底层实现            | 说明                                                      |
| --------------------------- | ------------------- | --------------------------------------------------------- |
| `ConcurrentMapCacheManager` | `ConcurrentHashMap` | 建议仅用于测试目的                                        |
| `NoOpCacheManager`          | 无                  | 不做任何缓存操作，可以视为关闭缓存                        |
| `CompositeCacheManager`     | 无                  | 用于组合多个不同的 `CacheManager`，会在其中遍历要找的缓存 |
| `EhCacheCacheManager`       | EhCache             | 适用于 EhCache                                            |
| `CaffeineCacheManager`      | Caffeine            | 适用于 Caffeine                                           |
| `JCacheCacheManager`        | JCache              | 适用于遵循 JSR-107 规范的缓存                             |

此外还有 Redis、Hazelcast、Infinispan。



## Security



## Quartz

Quartz 是定时调度框架。通过 Cron 表达式来描述任务的定期执行策略

~~~bash
【seconds minutes hours day-of-month month day-of-week year】
~~~

- **`\*`（通配符）**：匹配任意值，例如`* * * * * ?`表示每秒执行一次任务
- **`,`（列表）**：用于指定多个取值
- **`-`（范围）**：用于指定一个范围内的取值
- **`/`（步长）**：用于指定一个取值的步长，例如`0 */30 * * * ?`表示每30分钟执行一次任务。
- **`?`（无意义占位符）**：用于指定一个字段没有具体的取值
- **`#`（日历偏移量）**：用于指定某个月份的第几个周几，例如`0 0 0 ? * 3#1`表示每个月的第一个星期三执行任务。

Cron 使用示例：

- `0 0 10,14,16 * * ?` ：每天上午 10 点，下午 2 点、4 点
- `0 0/30 9-17 * * ?` ：上午 9 点到下午 5 点内每半小时
- `0 0 12 ? * WED` ：表示每个星期三中午 12 点
- `0 0 12 * * ?` ：每天中午 12 点触发
- `0 15 10 ? * *` ：每天上午 10:15 触发
- `0 15 10 * * ?` ：每天上午 10:15 触发
- `0 15 10 * * ? *` ：每天上午 10:15 触发
- `0 15 10 * * ? 2022` ：2022 年的每天上午 10:15 触发
- `0 * 14 * * ?` ：在每天下午 2 点到下午 2:59 期间的每 1 分钟触发
- `0 0/5 14 * * ?` ：在每天下午 2 点到下午 2:55 期间的每 5 分钟触发
- `0 0/5 14,18 * * ?` ：在每天下午 2 点到 2:55 期间和下午 6 点到 6:55 期间的每 5 分钟触发
- `0 0-5 14 * * ?` ：在每天下午 2 点到下午 2:05 期间的每 1 分钟触发
- `0 10,44 14 ? 3 WED` ：每年三月的星期三的下午 2:10 和 2:44 触发
- `0 15 10 ? * MON-FRI` ：周一至周五的上午 10:15 触发
- `0 15 10 15 * ?` ：每月 15 日上午 10:15 触发
- `0 15 10 L * ?` ：每月最后一日的上午 10:15 触发
- `0 15 10 ? * 6L` ：每月的最后一个星期五上午 10:15 触发
- `0 15 10 ? * 6L 2022-2023` ：2022 年至 2023 年的每月的最后一个星期五上午 10:15 触发
- `0 15 10 ? * 6#3` ：每月的第三个星期五上午 10:15 触发

Maven依赖

~~~xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-quartz</artifactId>
</dependency>
~~~

使用示例：

~~~java
@Service
public class ScheduleService {
    @Scheduled(cron = 0/5 * * * * *)
    public void test() {
        LOGGER.info("ScheduleService test invoke ......");
    }
}
~~~

这种方式只能把 Cron 表达式硬编码在代码中，无法做到动态定时任务管理。下面我们来实现这一点。首先 Quartz 是由 3 个核心 API 构成的，它们分别是：

- `Job` ：任务模型
- `Trigger` ：任务触发器
- `Scheduler` ：任务调度器

下面我们通过 RESTful API 来动态添加任务管理器

~~~java
@RestController
public class DynamicScheduleController {
    Autowired
    private Scheduler scheduler;
    
    @GetMapping("/addSchedule")
    public String addSchedule() throws SchedulerException {
		int random = ThreadLocalRandom.current().nextInt(1000);
        // 1. 创建 JobDetail
        JobDetail jobDetail = JobBuilder
            .newJob(SimpleJob.class) 
            .withIdentity("test-schedule" + random, "test-group")
            .build();
        
        // 2. 创建 Trigger，并指定每3秒执行一次
        CronScheduleBuilder cron = CronScheduleBuilder
            .cronSchedule("0/3 * * * * ?");
        
        Trigger trigger = TriggerBuilder
            .newTrigger()
            .withIdentity("test-trigger" + random, "test-trigger-group")
            .withSchedule(cron).build();
        
        // 3. 调度任务
        scheduler.scheduleJob(jobDetail, trigger);
    }
}

public class SimpleJob implements Job {
    private Logger logger = LoggerFactory.getLogger(this.getClass());
    
    @Override
    public void execute(JobExecutionContext context) {
        logger.info("简单任务执行 ......");
    }
}
~~~

默认将定时任务的信息保存在内容中，可以在 application.yaml 中配置相关参数，将信息保存在数据库中：

~~~yaml
# 设置将定时任务的信息保存到数据库
spring.quartz.job-store-type=jdbc
    
# 每次应用启动的时候都初始化数据库表结构
spring.quartz.jdbc.initialize-schema=always
~~~

![33. 自动初始化了数据库.png](./assets/e9df038fea954eea8e94483e95a30299tplv-k3u1fbpfcp-jj-mark1512000q75.webp)

暂停与恢复定时任务：

~~~java
@GetMapping("/pauseSchedule")
public String pauseSchedule(String jobName, String jobGroup) throws SchedulerException {
    JobKey jobKey = JobKey.jobKey(jobName, jobGroup);
    // 获取定时任务
    JobDetail jobDetail = scheduler.getJobDetail(jobKey);
    if (jobDetail == null) {
        return "error";
    }
    scheduler.pauseJob(jobKey);
    return "success";
}

@GetMapping("/remuseSchedule")
public String remuseSchedule(String jobName, String jobGroup) throws SchedulerException {
    JobKey jobKey = JobKey.jobKey(jobName, jobGroup);
    // 获取定时任务
    JobDetail jobDetail = scheduler.getJobDetail(jobKey);
    if (jobDetail == null) {
        return "error";
    }
    scheduler.pauseJob(jobKey);
    return "success";
}
~~~

移除定时任务：

~~~java
@GetMapping("/removeSchedule")
public String removeSchedule(String jobName, String jobGroup, String triggerName, String triggerGroup) throws SchedulerException {
    TriggerKey triggerKey = TriggerKey.triggerKey(triggerName, triggerGroup);
    JobKey jobKey = JobKey.jobKey(jobName, jobGroup);
    Trigger trigger = scheduler.getTrigger(triggerKey);
    if (trigger == null) {
        return "error";
    }
    // 停止触发器
    scheduler.pauseTrigger(triggerKey);
    // 移除触发器
    scheduler.unscheduleJob(triggerKey);
    // 删除任务
    scheduler.deleteJob(jobKey);
    return "success";
}
~~~



## Swagger

在线文档最大的亮点就是实时更新的，无需我们手动更新文档。而且可以在文档中对接口进行调试。而 OpenAPI 规范如何编写接口文档内容，这是 Swagger 需要考虑的事情

Swagger 希望我们使用遵循 RESTful 规范的方式来编写接口

~~~xml
<dependency>
    <groupId>io.springfox</groupId>
    <artifactId>springfox-boot-starter</artifactId>
    <version>3.0.0</version>
</dependency>
~~~



使用示例：

~~~java
@Configuration(proxyBeanMethods = false)
@EnableSwagger2
public class Swagger2Configuration {
    @Bean
    public Docket createRestApi() {
        return new Docket(DocumentationType.OAS_30)
            .apiInfo(apiInfo())
            .select()
            // 扫描 RestController
            .apis(RequestHandlerSelectors
                  .basePackage("com.linkedbear.boot.swagger"))
            .paths(PathSelectors.any())
            .build();
    }
    
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
                .title("SpringBoot手动集成Swagger2")
                .description("test swagger document")
                .contact(new Contact("https://github.com/LinkedBear", "", ""))
                .version("1.0")
                .build();
    }
}
~~~

启动工程后，我们来访问 [http://localhost:8080/swagger-ui.html](https://link.juejin.cn/?target=http%3A%2F%2Flocalhost%3A8080%2Fswagger-ui.html)



Swagger 以 RestController 类进行分组的，默认分组名是  Controller 类名。可以通过 @Api 为分组指定一个命名

~~~java
@RestController
@Api(tags = "这是一个测试接口类")
public class DemoController {
    
    // ......
}
~~~

 @ApiOperation 给接口添加说明信息

~~~java
@GetMapping("/test")
@ApiOperation(value = "这是一个测试的接口", notes = "仅供测试，切勿当真", hidden = false)
// 如果我们不希望接口出现在文档中，可以通过 hidden = true 来隐藏它
public String test() {
    return "test";
}
~~~



@ApiParam 给参数添加说明信息

| 属性名称        | 属性类型 | 默认值 | 作用                     |
| --------------- | -------- | ------ | ------------------------ |
| name            | String   | 空     | 定义参数的名称           |
| value           | String   | 空     | 定义参数的简单描述       |
| defaultValue    | String   | 空     | 定义参数的默认值         |
| allowableValues | String   | 空     | 定义参数的取值范围       |
| required        | boolean  | false  | 定义参数是否必填         |
| allowMultiple   | boolean  | false  | 定义参数能否接收多个数值 |
| example         | String   | 空     | 给出一个合法的参数值示例 |
| hidden          | boolean  | false  | 定义参数是否展示在文档上 |

~~~java
@GetMapping("/test")
@ApiOperation(value = "这是一个测试的接口", notes = "仅供测试，切勿当真")
public String test(
    @ApiParam(
        value = "测试姓名", 
        defaultValue = "zhangsan",
        allowableValues = "zhangsan,lisi,wangwu", 
        required = true) 
    String name,
    
    @ApiParam(
        value = "测试年龄", 
        allowableValues = "10, 20, 30, 40, 50", 
        example = "10") 
    Integer age) {
    
    return "test";
}
~~~



我们通过 @ApiModelProperty 来说明返回对象的信息：

~~~java
@ApiModel(value = "用户类", description = "封装用户的基本信息")
public class User {
    @ApiModelProperty(value = "用户id", required = true)
    private String id;
    
    @ApiModelProperty(value = "用户姓名", required = true)
    private String name;
    
    @ApiModelProperty(value = "用户年龄", notes = "只能传入大于0的数")
    private Integer age;
    
    // getter setter ......
~~~



