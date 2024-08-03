# ShardingSphere

[TOC]

## 概述

Apache ShardingSphere 是分布式数据库中间件，它由 JDBC、Proxy 和 Sidecar（规划中）这 3 款相互独立，却又能够混合部署配合使用的产品组成。

ShardingSphere-JDBC 定位为轻量级 Java 框架，在 Java 的 JDBC 层提供的额外服务。可理解为增强版的 JDBC 驱动，完全兼容 JDBC 和各种 ORM 框架。

![image-20220804195402870](./assets/image-20220804195402870.png)

ShardingSphere-JDBC 的 Maven 依赖如下：

~~~xml
<!-- 必须引入的包 ShardingSphere -->
<dependency>
    <groupId>org.apache.shardingsphere</groupId>
    <artifactId>shardingsphere-jdbc-core-spring-boot-starter</artifactId>
    <version>5.2.0</version>
</dependency>
~~~

下面配置定义了 ShardingSphere 使用的数据源：

~~~yaml
spring:
    shardingsphere:
        datasource:
          names: master,slave 	# 这里定义了各个数据源的名称
          # 主数据源
          master:
            type: com.alibaba.druid.pool.DruidDataSource 	# 指定使用何种数据源
            driver-class-name: com.mysql.cj.jdbc.Driver
            url: jdbc:mysql://192.168.60.100:3307/db1
            username: root
            password: "you_password"
            
          # 从数据源
          slave:
            type: com.alibaba.druid.pool.DruidDataSource
            driver-class-name: com.mysql.cj.jdbc.Driver
            url: jdbc:mysql://192.168.60.100:3308/db2
            username: root
            password: "you_password"
~~~



ShardingSphere-Proxy 定位为透明化的**数据库代理端**，提供封装了数据库二进制协议的服务端版本

![image-20220804195432673](./assets/image-20220804195432673.png)



虽然我们在 ShardingSphere 中配置了多个数据源，但是 ShardingSphere 将多个数据源合并为一个统一的逻辑数据源。所以从客户端来看，只有一个逻辑数据源。在配置分表分库以及数据分片时，都会指定一个逻辑表名，客户端要以逻辑表名为依据，来执行 SQL 语句。

## 分库分表

当表过大（超过 20,000k 行），但是主机 IO 负载并不高，那么就将一张大表拆分成几张小表，表的 schema 很保持相同，这就是**「分表」**。如果 IO 负载过高，那么就考虑在别的主机上建立新的数据库，这就是**「分库」**。

现在我们有一个需求：将 t_order 表拆分到两个数据库中，分别为`db1`和`db2`，每个数据库又将该表拆分为三张表，分别为`t_order_1`、`t_order_2`和`t_order_3`。

~~~html
db0
├── t_order_0
├── t_order_1
└── t_order_2
db1
├── t_order_0
├── t_order_1
└── t_order_2
~~~



~~~yaml
# 分片规则配置
rules:
    sharding:
    	# 分片算法配置
    	sharding-algorithms:
    		database-inline:
             	# 分片算法类型
            	type: INLINE
                props:
                # 分片算法的行表达式
                # 行表达式标识符可以使用 ${...} 或 $->{...}，但前者与 Spring 本身的属性文件占位符冲突，因此在 Spring 环境中使用行表达式标识符建议使用 $->{...}
            		algorithm-expression: db$->{order_id > 4 ? 1 : 0}
    		table-inline:
                # 分片算法类型
                type: INLINE
                props:
                    # 分片算法的行表达式
                    algorithm-expression: t_order_$->{order_id % 4}

    tables:
        t_order:	# 定义一个逻辑表名称
            actual-data-nodes: db${0..1}.t_order_${0..3}
            # 分库策略
            database-strategy:
                standard:
                    # 分片列名称
                    sharding-column: order_id
                    # 分片算法名称
                    sharding-algorithm-name: database-inline
            # 分表策略
            table-strategy:
                standard:
                    # 分片列名称
                    sharding-column: order_id
                    # 分片算法名称
                    sharding-algorithm-name: table-inline
~~~





## 主从同步

> 这里主要介绍单主复制。

![img](./assets/image-20220714133617856.png)

MySQL 主从同步的基本原理是：「slave 从 master 中读取 binlog 来进行数据同步」。具体步骤如下：

1. master 将数据改变记录到二进制日志（binary log）中。
2. 当 slave 上执行 `start slave` 命令之后，slave会创建一个 IO 线程用来连接 master，请求 master 中的 binlog。
3. 当 slave 连接 master 时，master 会创建一个 `log dump` 线程，用于发送 binlog 的内容。在读取 binlog 的内容的操作中，会对主节点上的 binlog 加锁，当读取完成并发送给从服务器后解锁。
4. IO 线程接收主节点 binlog dump 进程发来的更新之后，保存到 中继日志（relay log） 中
5. slave的 SQL 线程，读取 relay log 日志，并解析成具体操作，从而实现主从操作一致，最终数据一致。



下面简单搭建一个一主多从的 MySQL 架构：

![image-20220807183231101](./assets/image-20220807183231101.png)

1. 在 docker 中创建并启动 MySQL 主服务器，并创建 MySQL 主服务器配置文件

   ~~~shell
   [mysqld]
   # 服务器唯一id，默认值1
   server-id=1
   # 设置日志格式，默认值ROW
   binlog_format=STATEMENT
   
   # 二进制日志名，默认binlog
   # log-bin=binlog
   # 设置需要复制的数据库，默认复制全部数据库
   # binlog-do-db=mytestdb1
   # binlog-do-db=mytestdb2
   # 设置不需要复制的数据库
   # binlog-ignore-db=mysql
   # binlog-ignore-db=infomation_schema
   ~~~

   `binlog_format` 的参数说明：

   - `STATEMENT`：日志记录的是主机数据库的`写指令`，性能高，但是 now() 之类的函数，以及获取系统参数的操作，会出现主从数据不同步的问题。
   - `ROW`（默认）：日志记录的是主机数据库的`写后的数据`，批量操作时性能较差，解决now()或者 user()或者 @@hostname 等操作，在主从机器上不一致的问题。
   - `MIXED`：是以上两种 level 的混合使用，有函数用 ROW，没函数用 STATEMENT

2. 使用命令行登录到 MySQL 主服务器：

   ~~~shell
   #进入容器：env LANG=C.UTF-8 避免容器中显示中文乱码
   docker exec -it atguigu-mysql-master env LANG=C.UTF-8 /bin/bash
   #进入容器内的mysql命令行
   mysql -uroot -p
   ~~~

3. 在主机中创建 slave 用户：

   ~~~sql
   -- 修改默认密码校验方式
   ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
   
   -- 创建slave用户
   CREATE USER 'atguigu_slave'@'%';
   
   -- 设置密码
   ALTER USER 'atguigu_slave'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
   
   -- 授予复制权限
   GRANT REPLICATION SLAVE ON *.* TO 'atguigu_slave'@'%';
   
   -- 刷新权限
   FLUSH PRIVILEGES;
   
   
   ~~~

4. 主机中查询 master 状态：

   ~~~sql
   -- 查询 master 状态
   SHOW MASTER STATUS;
   ~~~

   记下`File`和`Position`的值。执行完此步骤后不要再操作主服务器MYSQL，防止主服务器状态值变化。

   [![image-20220804191852164](./assets/image-20220804191852164.png)](https://image-tuchuang.oss-cn-chengdu.aliyuncs.com/image-20220804191852164.png)

5. 在 docker 中创建并启动 MySQL 从服务器，创建 MySQL 从服务器配置文件，并配置如下内容：

   ~~~shell
   [mysqld]
   # 服务器唯一id，每台服务器的id必须不同，如果配置其他从机，注意修改id
   server-id=2
   # 中继日志名，默认xxxxxxxxxxxx-relay-bin
   # relay-log=relay-bin
   ~~~

6. 使用命令行登录MySQL从服务器：

   ~~~shell
   #进入容器：
   docker exec -it atguigu-mysql-slave1 env LANG=C.UTF-8 /bin/bash
   #进入容器内的mysql命令行
   mysql -uroot -p
   ~~~

7. 在从机上配置主从关系：

   ~~~sql
   #修改默认密码校验方式
   ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
   
   CHANGE MASTER TO MASTER_HOST='192.168.100.201', 
   MASTER_USER='atguigu_slave',MASTER_PASSWORD='123456', MASTER_PORT=3306,
   MASTER_LOG_FILE='binlog.000003',MASTER_LOG_POS=1357; 
   ~~~

8. 启动从机的复制功能：

   ~~~sql
   START SLAVE;
   -- 查看状态（不需要分号）
   SHOW SLAVE STATUS
   ~~~

   面两个参数都是Yes，则说明主从配置成功！

   [![img](./assets/image-20220715000533951.png)](https://image-tuchuang.oss-cn-chengdu.aliyuncs.com/image-20220715000533951.png)



需要停止/重置时，可以使用如下SQL语句

```sql
-- 停止 I/O 线程和 SQL 线程的操作。
stop slave; 

-- 在从机上执行。用于删除 SLAVE 数据库的 relaylog 日志文件，并重新启用新的 relaylog 文件。
reset slave;

-- 在主机上执行。删除所有的 binglog 日志文件，并将日志索引文件清空，重新开始所有新的日志文件。
-- 用于第一次进行搭建主从库时，进行主库 binlog 初始化工作；
reset master;
```

## 读写分离

~~~properties
rules:
- !READWRITE_SPLITTING
  dataSourceGroups:
  	# 定义读写分离逻辑数据源名称
    <data_source_name> (+): 
    	# 所引用的写数据源的名称，默认使用 Groovy 的行表达式 SPI 实现来解析
       write_data_source_name: 
       # 所引用的读数据源的名称，多个从数据源用逗号分隔，默认使用 Groovy 的行表达式 SPI 实现来解析
       read_data_source_names: 
        # 事务内读请求的路由策略，可选值：PRIMARY（路由至主库）、FIXED（同一事务内路由至固定数据源）、DYNAMIC（同一事务内路由至非固定数据源）。默认值：DYNAMIC
       transactionalReadQueryStrategy (?):
       # 负载均衡算法名称
       loadBalancerName: 
  
  # 负载均衡算法配置
  loadBalancers:
  # 定义负载均衡算法名称
    <load_balancer_name> (+): 
    # 负载均衡算法类型
      type: 
      # 负载均衡算法属性配置
      props: 
        # ...
~~~

下面给出一个示例：

~~~yaml
rules:
- !READWRITE_SPLITTING
  dataSourceGroups:
    readwrite_ds:
      writeDataSourceName: write_ds
      readDataSourceNames:
        - read_ds_0
        - read_ds_1
      transactionalReadQueryStrategy: PRIMARY
      loadBalancerName: random
  loadBalancers:
    random:
      type: RANDOM		# 随机负载均衡算法
    round:
      type: RANDOM
    weight:
      tpye: WEIGHT
      props:
		read_ds_0: 1
		read_ds_1: 2
~~~



直接在分库中指定逻辑数据源，即可将读写分离和分库分表结合起来，下面给出一个示例：

~~~html
// Master_A
sharding-test_0
├── student_0
└── student_1


// Master_B
sharding-test_1
├── student_0
└── student_1


// SLAVE_A_0
sharding-test_0
├── student_0
└── student_1


// SLAVE_B_0
sharding-test_1
├── student_0
└── student_1

// SLAVE_A_1
sharding-test_0
├── student_0
└── student_1


// SLAVE_B_1
sharding-test_1
├── student_0
└── student_1
~~~

~~~yaml


schemaName: readwrite_splitting_db
 
dataSources:
  write_ds_0:
    url: 
  write_ds_1:
    url: 
  read_ds_0:
    url: 
  read_ds_1:
    url: 
  read_ds_2:
    url: 
  read_ds_3:
    url: 
 
rules:
- !READWRITE_SPLITTING
  dataSources:
    # 读写分离
    readwrite_ds_0: 	# 逻辑数据源
      writeDataSourceName: write_ds_0
      readDataSourceNames:
        - read_ds_0
        - read_ds_1
      loadBalancerName: random
    readwrite_ds_1:		# 逻辑数据源
      writeDataSourceName: write_ds_1
      readDataSourceNames:
        - read_ds_2
        - read_ds_3
      loadBalancerName: random
  loadBalancers:
    random:
      type: RANDOM
 
- !SHARDING
  tables:
  #	分表分库
    student:		# 逻辑数据源
     # 这里指定的是读写分离中的逻辑数据源，后面跟一个真实的表名
      actualDataNodes: readwrite_ds_${0..1}.student_${0..1}
      
~~~



## 多数据源

这里使用 `dynamic-datasource` 来实现多数据源的切换。

~~~xml

<!-- dynamic多数据源 -->
<dependency>
	<groupId>com.baomidou</groupId>
	<artifactId>dynamic-datasource-spring-boot-starter</artifactId>
	<version>3.1.1</version>
</dependency>
~~~

~~~yaml

spring:
  datasource:
    # 从 spring.datasource.dynamic 中指定普通的数据源
    dynamic:
      datasource:
        tour:
          type: com.alibaba.druid.pool.DruidDataSource
          driver-class-name: com.mysql.jdbc.Driver
          url: jdbc:mysql://..
          username: root
          password: root
        avalon:
          type: com.alibaba.druid.pool.DruidDataSource
          driver-class-name: com.mysql.jdbc.Driver
          url: jdbc:mysql://...
          password: root
          
      # 指定默认数据源名称
      primary: tour
  # 指定 shardingsphere 的数据源，这里就不演示了
  shardingsphere:
    datasource:
      
~~~

添加多数据源配置类：

~~~java

@Configuration
@AutoConfigureBefore({DynamicDataSourceAutoConfiguration.class,
        SpringBootConfiguration.class})
public class DataSourceConfiguration {
 
    private final DynamicDataSourceProperties properties;

    private final Map<String, DataSource> dataSources;
	// 这里会依赖注入的
    public DataSourceConfiguration(DynamicDataSourceProperties properties, @Lazy Map<String, DataSource> dataSources) {
        this.properties = properties;
        this.dataSources = dataSources;
    }
 
 
    @Bean
    public DynamicDataSourceProvider dynamicDataSourceProvider() {
        Map<String, DataSourceProperty> datasourceMap = properties.getDatasource();
        return new AbstractDataSourceProvider() {
            @Override
            public Map<String, DataSource> loadDataSources() {
                Map<String, DataSource> dataSourceMap = createDataSourceMap(datasourceMap);
                // 将 shardingjdbc 管理的数据源也交给动态数据源管理
                dataSourceMap.put(DBConstants.SHARDING, shardingSphereDataSource);
                return dataSourceMap;
            }
        };
    }
 

    @Primary
    @Bean
    public DataSource dataSource(DynamicDataSourceProvider dynamicDataSourceProvider) {
        DynamicRoutingDataSource dataSource = new DynamicRoutingDataSource();
        dataSource.setPrimary(properties.getPrimary());
        dataSource.setStrict(properties.getStrict());
        dataSource.setStrategy(properties.getStrategy());
        dataSource.setProvider(dynamicDataSourceProvider);
        dataSource.setP6spy(properties.getP6spy());
        dataSource.setSeata(properties.getSeata());
        return dataSource;
    }
}
~~~

接着使用 @DS 注解来指定所使用的数据源：

~~~java

@Repository
public interface CommonDealerMapper {
 
    // 使用多数据源默认的 tour 库
    CommonDealerEntity getCommonDealer(@Param("clientCode") String clientCode);
 
    // 使用多数据源的 sharding 库
    @DS(value = "sharding")
    AvalonCommonDealer getShuCangCommonDealer(@Param("clientCode") String clientCode);
}
~~~

推荐使用枚举值来代替硬编码的字符串

## 广播表

  广播表指所有的分片数据源中都存在的表，表结构及其数据在每个数据库中均完全一致。 适用于数据量不大且需要与海量数据的表进行关联查询的场景。

1. 插入、更新操作会实时在所有节点上执行，保持各个分片的数据一致性
2. 查询操作，只从一个节点获取

~~~yaml
rules:
- !BROADCAST
  tables: # 广播表规则列表
  	# 确保每个数据源中都有 table_name 这张表
    - <table_name>
    - <table_name>
~~~

