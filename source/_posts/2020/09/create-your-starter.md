---
title: 实现自己的SpringBoot Starter
date: 2020-09-05 16:11:30
tags:
    - SpringBoot

categories:
  - ['SpringBoot']
---

### 题外话
前两个月因为工作变动，水博客的节奏有点慢，平时为了熟悉公司业务，加上新公司加班较多，落下了....现在继续开始水

### 要做啥？
之前有了解过SpringBoot启动流程，其中一块就涉及到SpringBoot Starter的内容，现在公司封装的东西基本上都是配合SpringBoot快速使用的，比如引入一个依赖，然后加个@EnableXXXX注解，就可以了。。。今天要实现一个自己的SpringBoot starter,用来实现打印方法的执行时间，惭愧的想到了之前自己的实现方法：

```
public String getUserInfo(){
        long startTs = System.currentTimeMillis();
        //业务逻辑

        long endTs = System.currentTimeMillis();
        //输出时间
        log.info("getUserInfo 用时：{} ms",endTs - startTs);
        return "xxx";
}
```
这无疑是愚蠢的，记得有一次，领导找到我：‘xxx,你拉一下日志，类似于上面这种有计算执行用时的所有当天日志，然后计算一下每个方法的平均耗时，95线等，’，我：？？？
当场就懵逼了，因为每个方法里打印的日志格式都不统一，统计起来就很费力了，如：

- `log .info("xxx 耗时：{} ms",endTs - startTs);`
- `log .info("xxx 用时：{} 毫秒",endTs - startTs);`
- `log .info("xxx 用时 {} 秒",（endTs - startTs)/1000);`

### 怎么做更好？
- 当然，我第一时间是想到了 封装一个Util，在所有需要打印时间的地方，都调用它，这样，输出的格式就可以统一了。但是，代码还是侵入了业务代码，某天想停用，就很不方便。
- 另一种比较好的做法就是利用aop了，今天就试试手。当然还有更好的，如点评开源的Cat之类的，更专业。

### 动手实践
- 开始之前要对SpringBoot的启动流程有简单的了解，特别是ImportSelector这些
- 除此之外，AOP相关知识也必不可少，如@Aspect使用等。

#### 目标
- 通过AOP实现无侵入的打印特定格式日志的功能
- 可以封装成模块，方便复用
- 在某方法上添加指定注解即可打印该方法的执行时间
- 模块进入之后通过总开关控制：@EnableXXXX

#### 项目依赖
这里仅给出了关键部分的依赖。
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-autoconfigure</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-configuration-processor</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>

<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>1.7.30</version>
</dependency>
```

#### 定义在方法上使用的注解

```
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogUsedTime {

    /**
     * 自定义输出的方法名字,不指定则取方法名
     * @return
     */
    String aliasMethodName() default "";

    /**
     * 日志级别
     * @return
     */
    LogLevel logLevel() default LogLevel.INFO;
}

```
#### 定义aop切面处理类
指定切点以及切面逻辑

```
@Aspect
@EnableAspectJAutoProxy(exposeProxy = true,proxyTargetClass = true)
public class LogUsedTimeAspect implements PriorityOrdered {

    private Logger log = LoggerFactory.getLogger(getClass());

    /**
     * 定义切点
     */
    @Pointcut("@annotation(com.eqshen.anno.LogUsedTime)")
    private void logUsedTimeAnnotationCut(){}

    //使用上面定义的切点
    @Around(value = "logUsedTimeAnnotationCut() && @annotation(logUsedTime)")
    public Object methodExecute(ProceedingJoinPoint point, LogUsedTime logUsedTime) throws Throwable {
        String aliasMethodName = logUsedTime.aliasMethodName();
        LogLevel logLevel = logUsedTime.logLevel();

        String methodName = point.getSignature().getName();
        if(!StringUtils.isEmpty(aliasMethodName)){
            methodName = aliasMethodName;
        }

        long startTimestamp = System.currentTimeMillis();
        Object result = point.proceed();
        long endTimestamp = System.currentTimeMillis();

        //log
        doLog(logLevel,methodName,endTimestamp - startTimestamp);

        return result;
    }

    private void doLog(LogLevel logLevel,String methodName,long usedTimeMs){
        String format = String.format("[Time Use] Task %s used time: %d ms", methodName,usedTimeMs);

        switch (logLevel){
            case OFF:
                //do nothing
                break;
            case WARN:
                log.warn(format);
                break;
            case DEBUG:
                log.debug(format);
                break;
            case ERROR:
                log.error(format);
                break;
            case FATAL:
                log.error(format);
                break;
            case TRACE:
                log.trace(format);
                break;
            default:
                log.info(format);


        }
    }

    /**
     * aop优先级放置在最后
     * @return
     */
    @Override
    public int getOrder() {
        return Integer.MAX_VALUE;
    }
}

```

#### 定义自动引入Selector
自定义Selector，在SpringBoot启动阶段可以自动装载我们上面定义的aop切面处理类。

```
public class EnableEQTimeLogSelector implements ImportSelector {

    private Logger log = LoggerFactory.getLogger(getClass());

    @Override
    public String[] selectImports(AnnotationMetadata importingClassMetadata) {
        log.info("(*^▽^*) EQTimeLog组件启用...OK");
        return new String[]{LogUsedTimeAspect.class.getName()};
    }
}

```

#### 定义总开关 @EnableXXX
自定义开关注解，挂载Selector，让一切变得简单。
```
/**
 * 开关
 */
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
@Import(EnableEQTimeLogSelector.class)
public @interface EnableEQTimeLog {
}

```

#### 使用 & 测试
首先，创建一个新的SpringBoot项目，依赖我们 上面的项目。
```
<dependency>
    <groupId>com.eqshen</groupId>
    <artifactId>time-log-starter</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</dependency>
```
创建测试类
```
@Service
public class TestService {

    //使用注解，输出方法执行耗时
    @LogUsedTime(aliasMethodName = "打招呼")
    public String sayHello(String name){
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return "hello "+ name;
    }
}
```
编写Junit单元测试

```
@SpringBootTest
//注意这里，开启功能
@EnableEQTimeLog
class EqlogDemoApplicationTests {

    @Autowired
    private TestService testService;

    @Test
    void contextLoads() {
        testService.sayHello("tom");
    }

}
```

看一下日志输出

```
...
2020-09-09 23:56:48.572  INFO 10589 --- [           main] com.demo.EqlogDemoApplicationTests       : No active profile set, falling back to default profiles: default
2020-09-09 23:56:48.655  INFO 10589 --- [           main] c.e.selector.EnableEQTimeLogSelector     : (*^▽^*) EQTimeLog组件启用...OK
2020-09-09 23:56:49.201  INFO 10589 --- [           main] com.demo.EqlogDemoApplicationTests       : Started EqlogDemoApplicationTests in 15.8 seconds (JVM running for 16.704)
...
...
2020-09-09 23:56:50.342  INFO 10589 --- [           main] com.eqshen.aspect.LogUsedTimeAspect      : [Time Use] Task 打招呼 used time: 1015 ms

```


#### 总结

至此，整篇文章水完了。没有任何难度的demo，仅仅是熟悉如何自定义starter，以及之前学习过@Import这些知识的简单实践。
虽然很简单，但是在实操的时候还是可能会遇到意想不到的问题，如 maven无法引用本地仓库的jar（版本原因等）。共勉。