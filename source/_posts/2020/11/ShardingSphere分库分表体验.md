---
title: ShardingSphere分库分表体验
date: 2020-11-02 21:47:39
tags:
    - 分库分表
    - ShardingSphere
    - 实践
categories:
  - ['实践','分库分表']
---


### ShardingSphere分库分表实践

#### 简介

#####使用场景

当系统中数据库成功性能的瓶颈，特别是一张表中的数据量过大，达到500w甚至1000w时，这种情况下，通过建立有效索引也难以满足查询的效率要求时，就需要考虑到分库分表，分表一般是将表进行水平拆分，如根据用户id，订单编号等策略。

举个栗子：将订单表t_order拆分成四张表，分别是t_order_0,t_order_1,t_order_2,t_order_3；分表的路由策略为 `order_no % 4`；

这种情况下：

- 订单号为10000的订单 将被保存在 10000%4 = 0 即 t_order_0表；
- 订单号为 10001的订单 将被保存在 10001%4=1 即 t_order_1表；
- 其他同理。

##### 常见的分库分表方案

- 第一种是客户端型的，如 ShardingSphere(Sharding-JDBC)，分表路由等逻辑需要侵入到客户端的代码中。

  <img src="https://shardingsphere.apache.org/document/current/img/shardingsphere-jdbc-brief.png" alt="Sharding-JDBC" style="zoom:67%;" />

- 第二种是基于代理型的，如 MyCat。这种中间件对客户端是透明的，客户端是不知道自己的请求被代理了的。

  ![MyCat架构图](http://www.mycat.org.cn/index_files/arc.png)

#####区别&优缺点

- MyCat是一个三方中间件应用，需要独立部署并运行，ShardingJDBC是一个jar包，需要客户端集成；
- MyCat代理数据库，ShardingJDBC是直连数据库，相比而言，后者在性能上更占优势；
- 二者在多表Join等特性及内部函数上存在不同的限制；
- [个人理解] 目前工作中Sharding-JDBC用的多一点（了解到的像当当、喜马拉雅、微盟），因为MyCat需要专门的人员或团队去维护保证稳定性，而且就目前社区文档规范及活跃性来看，ShardingSphere也更好一点。

##### 分片策略及分片算法

- 分片策略与分片算法 二者区别与关系是什么？

  算法是策略中抽象出来的逻辑与精髓，策略是算法结合业务场景的具体实现。（个人理解，不同见解请留言交流）

- 分片策略的两种维度

  - 数据源分片策略（DatabaseShardingStrategy）：通俗点说，就是作用于多个数据库实例之间。
  - 表分片策略（TableShardingStrategy）：数据如何被分配到具体的表

- 策略分类

  - StandardShardingStrategy
  - ComplexShardingStrategy
  - InlineShardingStrategy
  - HintShardingStrategy
  - NoneShardingStrategy

  具体的分片策略及其区别推荐还是去官方阅读最新文档，[这里](https://shardingsphere.apache.org/document/current/en/features/sharding/concept/sharding/)仅供参考。

- 算法分类

  - PreciseShardingAlgorithm
  - RangeShardingAlgorithm
  - HintShardingAlgorithm
  - ComplexKeysShardingAlgorithm

Apache ShardingSphere 通过 SPI 方式允许开发者扩展算法，同时，它也内置了很多算法，详细可以阅读[官方文档](https://shardingsphere.apache.org/document/current/cn/user-manual/shardingsphere-jdbc/configuration/built-in-algorithm/) and [Here](https://shardingsphere.apache.org/document/current/cn/dev-manual/sharding/)。

#### 实践

Apache ShardingSphere 是一套开源的分布式数据库中间件解决方案组成的生态圈，其官网 [ShardingSphere](http://shardingsphere.apache.org/index_zh.html)，官方的入门资料可以先参考，这里不做详细介绍。

##### 创建项目并添加依赖

首先创建一个Maven管理的SpringBoot项目，并添加如下依赖

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter</artifactId>
</dependency>

<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-test</artifactId>
  <scope>test</scope>
  <exclusions>
    <exclusion>
      <groupId>org.junit.vintage</groupId>
      <artifactId>junit-vintage-engine</artifactId>
    </exclusion>
  </exclusions>
</dependency>

<!--mysql，根据自己数据库版本进行相关调整，不然会报错-->
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
  <version>8.0.13</version>
  <scope>runtime</scope>
</dependency>
<!--Mybatis-Plus-->
<dependency>
  <groupId>com.baomidou</groupId>
  <artifactId>mybatis-plus-boot-starter</artifactId>
  <version>3.1.1</version>
</dependency>
<!-- for spring boot -->
<dependency>
  <groupId>org.apache.shardingsphere</groupId>
  <artifactId>sharding-jdbc-spring-boot-starter</artifactId>
  <version>${sharding-sphere.version}</version>
</dependency>

<!-- for spring namespace -->
<dependency>
  <groupId>org.apache.shardingsphere</groupId>
  <artifactId>sharding-jdbc-spring-namespace</artifactId>
  <version>${sharding-sphere.version}</version>
</dependency>

<!-- lombok -->
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <version>1.18.16</version>
  <scope>provided</scope>
</dependency>
```

##### 创建数据库表

然后创建数据库，为了实现`分库` 功能，这里创建两个数据库，分别是：`sharding_practice_0`和`sharding_practice_1`。

分别在数据库中创建表 `t_user_0`、`t_user_1、`t_order_0`、`t_order_1。

```mysql
CREATE TABLE `t_order_0`  (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `order_no` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '订单号',
  `order_name` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '订单名称',
  `order_status` int NOT NULL COMMENT '订单状态',
  `user_id` bigint NOT NULL COMMENT '用户id',
  `create_time` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '创建时间',
  `update_time` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '更新时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = DYNAMIC;

```

```mysql
CREATE TABLE `t_user_0`  (
  `id` bigint NOT NULL AUTO_INCREMENT COMMENT '自增id',
  `name` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL COMMENT '用户名',
  `mobile` varchar(50) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NOT NULL COMMENT '订单名称',
  `address` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci NULL DEFAULT '' COMMENT '地址',
  `user_id` bigint NOT NULL COMMENT '用户id',
  `age` int NOT NULL COMMENT '年龄',
  `create_time` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) COMMENT '创建时间',
  `update_time` datetime(0) NOT NULL DEFAULT CURRENT_TIMESTAMP(0) ON UPDATE CURRENT_TIMESTAMP(0) COMMENT '更新时间',
  PRIMARY KEY (`id`) USING BTREE
) ENGINE = InnoDB AUTO_INCREMENT = 1 CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = DYNAMIC;

```

表结构都一样，就不重复贴了。

##### 创建实体类等

```java
@TableName("t_order")
@Data
public class OrderInfo {
    @TableId(value = "id",type = IdType.AUTO)
    private Long id;

    @TableField(value = "order_no")
    private String orderNo;

    private String orderName;

    private Integer orderStatus;

    private Long userId;

    private Date createTime;

    private Date updateTime;

}

```

```java
@TableName("t_user")
@Data
public class UserInfo {
    @TableId(value = "id",type = IdType.AUTO)
    private Long id;

    private Long userId;

    private String name;

    private Integer age;

    private String address;

    private String mobile;

    private Date createTime;

    private Date updateTime;
}
```

Mapper及Service直接省略，采用 mybatis-plus自动生成crud方法，不再贴出来。

##### 创建配置文件

配置文件是重点，注释都在其中，简单易懂。

其中，t_order表使用分片策略：`order_no % 2`;

t_user表采用强制路由，在配置中指定路由算法类 `com.eqshen.shardingpractice.algorithm.UserHintAlgorithm`,在代码中指定路由的键值。

```properties
# 应用名称
spring.application.name=sharding-practice
server.port=8098

# 数据源 ds0,ds1
spring.shardingsphere.datasource.names=ds0,ds1
# 第一个数据库
spring.shardingsphere.datasource.ds0.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.ds0.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.ds0.jdbc-url=jdbc:mysql://localhost:3306/sharding_practice_0?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8&characterSetResults=utf8&useSSL=false&verifyServerCertificate=false&autoReconnct=true&autoReconnectForPools=true&allowMultiQueries=true
spring.shardingsphere.datasource.ds0.username=root
spring.shardingsphere.datasource.ds0.password=123456

# 第二个数据库
spring.shardingsphere.datasource.ds1.type=com.zaxxer.hikari.HikariDataSource
spring.shardingsphere.datasource.ds1.driver-class-name=com.mysql.cj.jdbc.Driver
spring.shardingsphere.datasource.ds1.jdbc-url=jdbc:mysql://localhost:3306/sharding_practice_1?serverTimezone=UTC&useUnicode=true&characterEncoding=utf8&characterSetResults=utf8&useSSL=false&verifyServerCertificate=false&autoReconnct=true&autoReconnectForPools=true&allowMultiQueries=true
spring.shardingsphere.datasource.ds1.username=root
spring.shardingsphere.datasource.ds1.password=123456

# 水平拆分的数据库（表） 配置分库 + 分表策略 行表达式分片策略
## 分库策略
spring.shardingsphere.sharding.default-database-strategy.inline.sharding-column=order_no
spring.shardingsphere.sharding.default-database-strategy.inline.algorithm-expression=ds$->{order_no % 2}

## t_order分表策略
spring.shardingsphere.sharding.tables.t_order.actual-data-nodes=ds$->{0..1}.t_order_$->{0..1}
spring.shardingsphere.sharding.tables.t_order.key-generator.column=order_no
spring.shardingsphere.sharding.tables.t_order.key-generator.type=SNOWFLAKE
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.sharding-column=order_no
###分片算法表达式
spring.shardingsphere.sharding.tables.t_order.table-strategy.inline.algorithm-expression=t_order_$->{order_no % 2}


## t_user 分表策略
spring.shardingsphere.sharding.tables.t_user.actual-data-nodes=ds$->{0..1}.t_user_$->{0..1}
spring.shardingsphere.sharding.tables.t_user.key-generator.column=user_id
spring.shardingsphere.sharding.tables.t_user.key-generator.type=SNOWFLAKE
### 强制路由
spring.shardingsphere.sharding.tables.t_user.database-strategy.hint.algorithm-class-name=com.eqshen.shardingpractice.algorithm.UserHintAlgorithm
spring.shardingsphere.sharding.tables.t_user.table-strategy.hint.algorithm-class-name=com.eqshen.shardingpractice.algorithm.UserHintAlgorithm


# 打印执行的数据库以及语句
spring.shardingsphere.props.sql.show=true
spring.main.allow-bean-definition-overriding=true


# mybatis-plus
mybatis-plus.mapper-locations=classpath:/mapper/*.xml
mybatis-plus.configuration.jdbc-type-for-null='null'
```

完成以上内容就可以开始测试了

##### 强制路由

何为强制路由？ 且看官方原话。

- 使用动机

> 通过解析 SQL 语句提取分片键列与值并进行分片是 Apache ShardingSphere 对 SQL 零侵入的实现方式。若 SQL 语句中没有分片条件，则无法进行分片，需要全路由。
>
> 在一些应用场景中，分片条件并不存在于 SQL，而存在于外部业务逻辑。因此需要提供一种通过外部指定分片结果的方式，在 Apache ShardingSphere 中叫做 Hint。

- 实现机制

> Apache ShardingSphere 使用 `ThreadLocal` 管理分片键值。可以通过编程的方式向 `HintManager` 中添加分片条件，该分片条件仅在当前线程内生效。
>
> 除了通过编程的方式使用强制分片路由，Apache ShardingSphere 还计划通过 SQL 中的特殊注释的方式引用 Hint，使开发者可以采用更加透明的方式使用该功能。
>
> 指定了强制分片路由的 SQL 将会无视原有的分片逻辑，直接路由至指定的真实数据节点。

```java
@Slf4j
@NoArgsConstructor
public class UserHintAlgorithm implements HintShardingAlgorithm {
    @Override
    public Collection<String> doSharding(Collection collection, HintShardingValue hintShardingValue) {
        log.info("强制路由开始，表总数：{}，路由字段：{}",collection,hintShardingValue.getColumnName());

        final Collection shardingValues = hintShardingValue.getValues();
        List<String> result = new ArrayList<>();
        for (Object o : collection) {
            String targetName = (String) o;
            String suffix = targetName.substring(targetName.length() - 1);
            for (Object value : shardingValues) {
                Integer v = new Integer(value+"");
                if(v % 2 == Integer.parseInt(suffix)){
                    log.info("值：{} 被路由到表：{}",v,targetName);
                    result.add(targetName);
                }
            }
        }
        log.info("强制路由结束：{}",result);
        return result;
    }
}
```



##### 测试

创建测试类

```java
@SpringBootTest
@Slf4j
class ShardingPracticeApplicationTests {

    @Autowired
    private OrderService orderService;

    @Autowired
    private UserService userService;

    @Test
    void contextLoads() {
    }

    /**
     * 测试分片
     */
    @Test
    void testSave(){
        for (int i = 0; i < 5; i++) {
            OrderInfo orderInfo = new OrderInfo();
            orderInfo.setOrderStatus(1);
            orderInfo.setUserId(10001L + i);
            orderInfo.setOrderName("黄焖鸡米饭" + LocalDateTime.now().toString());
            final boolean save = this.orderService.save(orderInfo);
            log.info("============done:{}",save);
        }


    }

    /**
     * 测试强制路由
     */
    @Test
    void testHint(){

        for (int i = 0; i < 5 ; i++) {
            HintManager hintManager = HintManager.getInstance();
            hintManager.addDatabaseShardingValue("t_user",i%2);
            hintManager.addTableShardingValue("t_user",i%2);

            UserInfo userInfo = new UserInfo();
            userInfo.setAddress("上海市黄浦区延安东路618号");
            userInfo.setAge(18);
            userInfo.setName("张飞"+ i);
            userInfo.setMobile("1762134523"+i);

            final boolean save = this.userService.save(userInfo);
            log.info("============done:{}",save);
            hintManager.close();
        }
    }

}
```



完整的代码参见github  [ShardingJDBC菜鸟入门练习](https://github.com/eqshen/sharding-practice)

#### 参考

- [MyCat和ShardingJDBC对比](https://my.oschina.net/u/4318872/blog/4281049)

- [MyCat官网](http://www.mycat.org.cn/)
- [ShardingSphere官网](https://shardingsphere.apache.org/)


