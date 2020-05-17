---
title: SpringBoot-RabbitMQ实践记录
date: 2020-05-17 14:40:01
tags:
    - SpringBoot
    - MQ
    - RabbitMQ
    - 实践

categories:
  - ['消息队列']
  
---

### RabbitMQ + SpringBoot实践 

#### 为啥写这篇

接触并使用RabbitMQ已经有好几年了，每次配置MQ的时候，都要确保消息可靠不丢失、幂等性等问题。时间久了，曾经做过的最佳时间配置难免会忘记，甚至重复踩坑，这边目的就是整合一下靠谱的资源并做个笔记分享。

使用MQ时应该注意下面几个问题：

- 消息丢失
  - 持久化
  - 发送端发送失败：发送确认机制 confirm
  - 消费端丢失：消费确认机制 ack
- 重复消费、幂等性问题

#### 基础概念

关于RabbitMQ的相关基础概念、消息投递模式就不再重复了，可以参考此篇文章复习 [RabbitMQ高可用](https://github.com/suxiongwei/springboot-rabbitmq)

#### 实践

##### 环境及依赖

- SpringBoot  2.2.7.RELEASE
- JDK 1.8
- MySQL
- RabbitMQ

##### 流程

1. 生产者发送消息，并保存发送记录-发送提交
2. 接收发送结果回调，更新发送记录-发送成功/失败
3. 消费者接收到消息，查询消息记录是否存在，是否重复消费
4. 执行具体业务逻辑，消费消息，更新消息记录为-已消费/消费异常  状态
5. 根据消费结果执行手动 channel.basicAck

##### 配置文件 

`application.properties` 关键部分作用都有注释说明

```properties

spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.url=jdbc:mysql://localhost:3306/mq_practice?charset=utf8mb4&useSSL=false&serverTimezone=GMT%2B8
spring.datasource.username=root
spring.datasource.password=123456

## Hikari连接池的设置
#最小连接
spring.datasource.hikari.minimum-idle=5
#最大连接
spring.datasource.hikari.maximum-pool-size=15
#自动提交
spring.datasource.hikari.auto-commit=true
#最大空闲时常
spring.datasource.hikari.idle-timeout=30000
#连接池名
spring.datasource.hikari.pool-name=DatebookHikariCP
#最大生命周期
spring.datasource.hikari.max-lifetime=900000
#连接超时时间
spring.datasource.hikari.connection-timeout=15000
#心跳检测
spring.datasource.hikari.connection-test-query=SELECT 1

## mybatis配置
#xml路径
mybatis-plus.mapper-locations=classpath*:mapper/*.xml
#model路径
mybatis-plus.type-aliases-package=com.microloan.entity

#rabbitmq config begin
spring.rabbitmq.addresses=114.67.89.67:5672
spring.rabbitmq.username=admin
spring.rabbitmq.password=admin
spring.rabbitmq.virtual-host=/
#发送确认回调(exchange是否成功受理)
spring.rabbitmq.publisher-confirm-type=correlated
# 发送失败退回（exchange到queue失败）
spring.rabbitmq.publisher-returns=true

# 确认机制：auto、manual、node，
spring.rabbitmq.listener.simple.acknowledge-mode=manual
spring.rabbitmq.listener.simple.concurrency=5
spring.rabbitmq.listener.simple.max-concurrency=10
spring.rabbitmq.listener.simple.prefetch=1
#间隔事件2（multiplier）倍递增，如1,2,4,8,16...但最大不超过max-attempts次
spring.rabbitmq.listener.simple.retry.enabled=true
spring.rabbitmq.listener.simple.retry.max-attempts=8
spring.rabbitmq.listener.retry.initial-interval=2000
spring.rabbitmq.listener.simple.retry.multiplier=2
spring.rabbitmq.listener.simple.retry.max-interval=16000
#rabbitmq config end
```

##### RabbitMqConfig.java配置

```java
@Configuration
public class RabbitMqConfig {
    public static final String TEST_QUEUE = "approveRequestQueue";

    public static final String TEST_TOPIC_EXCHANGE = "approveRequestExchange_topic";

    public static final String TEST_ROUTING_KEY = "approveRequestRouting";

    @Value("${spring.rabbitmq.addresses}")
    private String addresses;

    @Value("${spring.rabbitmq.username}")
    private String username;

    @Value("${spring.rabbitmq.password}")
    private String password;

    @Value("${spring.rabbitmq.virtual-host}")
    private String virtualHost;

    @Value("${spring.rabbitmq.publisher-confirm-type}")
    private String publisherConfirmType;

    @Value("${spring.rabbitmq.publisher-returns}")
    private Boolean publisherConfirmReturns;


    @Bean
    public ConnectionFactory connectionFactory() {

        CachingConnectionFactory connectionFactory = new CachingConnectionFactory();
        connectionFactory.setAddresses(addresses);
        connectionFactory.setUsername(username);
        connectionFactory.setPassword(password);
        connectionFactory.setVirtualHost(virtualHost);
        /** 如果要进行消息回调，则这里必须要设置*/
        connectionFactory.setPublisherConfirmType(CachingConnectionFactory.ConfirmType.valueOf(publisherConfirmType.toUpperCase()));
        connectionFactory.setPublisherReturns(publisherConfirmReturns);
        return connectionFactory;
    }
	
    //TODO 此处应改成非单例模式,下文有解释，先插眼
    @Bean("rabbitTemplateCallback")
    public RabbitTemplate rabbitTemplateCallBack() {
        RabbitTemplate template = new RabbitTemplate(connectionFactory());
        return template;
    }


    @Bean
    public Queue approveRequestQueue(){
        return new Queue(TEST_QUEUE);
    }

    @Bean
    public TopicExchange approveRequestExchange() {
        return new TopicExchange(TEST_TOPIC_EXCHANGE);
    }

    @Bean
    public Binding bindingApproveReqEx(Queue reqQueue, TopicExchange reqExchange){
        return BindingBuilder.bind(reqQueue).to(reqExchange).with(TEST_ROUTING_KEY);
    }
}
```

##### 消息保存

为了防止消息重复消费，需要把消息入库。

创建一张表用来保存消息发送状态

```mysql
CREATE TABLE `mq_record` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `msg_type` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `msg_id` varchar(255) CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci NOT NULL,
  `status` int(2) NOT NULL COMMENT '1成功，0失败',
  `create_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP,
  `update_time` datetime NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`) USING BTREE,
  KEY `unique_index` (`msg_id`)
) ENGINE=InnoDB AUTO_INCREMENT=9 DEFAULT CHARSET=utf8mb4;
```

##### 消息生产者

rabbitTemplate支持设置两个回调函数，`setConfirmCallback`和`setReturnCallback`。作用如下：

- setConfirmCallback 消息从生产者到达exchange触发，有个boolean类型的ack字段表示是否成功发送到路由exchange
- setReturnCallback 消息成功发送到exchange，但分配给具体queue时失败会触发。

要注意的一点是 rabbitTemplate一旦设置了这两个回调函数，是全局生效的，但是真正的项目里我们可能有多个生产者，而每个生产者发送消息后的回调处理逻辑可能是不一样的，所以我们**应该在每个Producer中注入rabbitTemplate后才设置回调函数，而且rabbitTemplate应该是prototype类型（非单例）**

首先需要修改上面RabbitMqconfig类中的TODO处

```java
//支持回调的template
@Scope(ConfigurableBeanFactory.SCOPE_PROTOTYPE)  
@Bean("rabbitTemplateCallback")
public RabbitTemplate rabbitTemplateCallBack() {
    RabbitTemplate template = new RabbitTemplate(connectionFactory());
    return template;
}
```

然后producer中的代码大概应该是这样的

```java
@Component
class TestProducer{
    private RabbitTemplate rabbitTemplate;
    
    @Resource(name = "rabbitTemplateCallback") //应该和RabbitMqConfig类中配置的名称一致
    public void setRabbitTemplate(RabbitTemplate rabbitTemplate){
        this.rabbitTemplate = rabbitTemplate;
        this.rabbitTemplate.setConfirmCallback(...);
        this.rabbitTemplate.setReturnCallback(...);
    }
    
    public void sendMsg(String msg){
        //xxxx
    }
}

```

光有回调不行啊，得找个地方记录每个消息的发送记录：消息id,发送时间，是否成功发送等信息。这里我们选择数据库 MySQL。完整的Producer如下：

```java
@Component
@Slf4j
public class TestProducer {

    @Autowired
    private MqRecordService mqRecordService;

    private RabbitTemplate rabbitTemplate;

    @Resource(name = "rabbitTemplateCallback")
    public void setRabbitTemplate(RabbitTemplate rabbitTemplate){
        this.rabbitTemplate = rabbitTemplate;
        //消息从生产者到达exchange时返回ack，消息未到达exchange返回nack；
        this.rabbitTemplate.setConfirmCallback(new RabbitTemplate.ConfirmCallback() {
            @Override
            public void confirm(CorrelationData correlationData, boolean ack, String cause) {
                if(ack){
                    //更新消息发送记录表的状态，采用乐观锁，
                    mqRecordService.updateRecordStatus(correlationData.getId(),
                            MqRecordStatusEnum.SEND.getCode(),MqRecordStatusEnum.SEND_SUCCESS.getCode());
                    log.info("[MQ]消息发送结果确认 ack:{},cause:{},correlationData:{}",ack,cause,correlationData);
                }else{
                    log.error("[MQ]消息发送失败，ack:{},cause:{},correlationData:{}",ack,cause,correlationData);
                }
            }
        });

        //消息进入exchange但未进入queue时会被调用。
        this.rabbitTemplate.setReturnCallback(new RabbitTemplate.ReturnCallback() {
            @Override
            public void returnedMessage(Message message, int replyCode, String replyText, String exchange, String routingKey) {
                log.error("[MQ]消息未成功进入queue:exchange:{},route:{},replyCode:{},replyText:{},message:{}",exchange,routingKey,replyCode,replyText,message);
            }
        });
    }


    /**
     * @param approveReqId  approveInfo自增id字段
     * @param msg
     */
    @Transactional
    public void sendRequest(long approveReqId,String msg){
        try{
            String key = UUID.randomUUID().toString().replaceAll("-","");
            MessageProperties mp = new MessageProperties();
            mp.setMessageId(key);
            Message msgObj = new Message(msg.getBytes(),mp);
            //消息唯一键
            CorrelationData correlationData = new CorrelationData(key);

            //保存消息发送记录
            MqRecord mqRecord = new MqRecord();
            mqRecord.setMsgType("test");
            mqRecord.setMsgId(key);
            mqRecord.setStatus(MqRecordStatusEnum.SEND.getCode());
            mqRecord.setDataId(approveReqId);
            this.mqRecordService.businessSave(mqRecord);

            //Topic模式发送
            this.rabbitTemplate.convertAndSend(RabbitMqConfig.TEST_TOPIC_EXCHANGE,
                    RabbitMqConfig.TEST_ROUTING_KEY,msgObj,correlationData);
            log.info("[MQ]消息发送成功 {},{}",key,msg);

        }catch (Exception e){
            log.error("[MQ]消息发送失败 {},{}",approveReqId,msg,e);
            throw e;
        }
    }
}
```

##### 消息消费者

消费消息，成功之后手动调用ack。

```java
@Component
@Slf4j
public class TestConsumer {

    @Autowired
    private MqRecordService mqRecordService;


    @RabbitListener(queues = {RabbitMqConfig.TEST_QUEUE})
    public void consumeApproveRequest(Channel channel, Message msg) throws Exception {
        String msgId = msg.getMessageProperties().getMessageId();
        String msgBody = new String(msg.getBody());
        try{
            MqRecord mqRecord = this.mqRecordService.queryByUniqueKey(msgId);
            //先检查是否已经消费过
            if(!checkRepetableMsg(mqRecord,msgBody)){
                return;
            }

            //msgBody消息内容，调用具体的业务逻辑
            //...

            log.info("[MQ]消费消息成功:{}-{}",msgId,msgBody);
            this.mqRecordService.updateRecordStatus(msgId, MqRecordStatusEnum.CONSUMED.getCode());
            //每个参数的作用
            channel.basicAck(msg.getMessageProperties().getDeliveryTag(),false);
        }catch (Exception e){
            log.error("[MQ]消费消息失败:{}-{}",msgId,msgBody);
            throw e;
        }
    }

    /**
     * 检查重复检查的消息
     * @param mqRecord
     * @return
     */
    private boolean checkRepetableMsg(MqRecord mqRecord,String msgBody){
        if(mqRecord != null){
            if(mqRecord.getStatus() == MqRecordStatusEnum.CONSUMED.getCode()){
                log.warn("重复的请求，忽略 {}",mqRecord.getMsgId());
                return false;
            }
        }else{
            log.warn("未知的申请 {}",msgBody);
            return false;
        }
        return true;
    }
}
```

#### 总结

文章只是记录了一下常用的代码，以及MQ使用中需要注意的细节。完整代码看[这里](https://github.com/eqshen/RabbitMQBestPractice)。



