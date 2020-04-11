---
title: 一次Http未设置超时导致的线程假死阻塞
date: 2020-04-11 22:47:36
tags:
    - 线程
    - HTTP
    - BUG
categories:
  - ['BUG']
  - ['线程']
---


## 故事开始

某一天，我正在开开心心的写BUG，突然工作群里 @苟且偷生的BUG猿....我一看，这不是我嘛，再一看问题：xxx这笔订单怎么流程走到一半就终止了？快看看是啥问题。

## 初步排查

我一看这问题，难道报错了？没收到预警啊...迅速打开生产日志，一顿操作`cd ,ls,less,?,ctrl+c,ctrl+v,n`,终于找到这笔订单的遗言：

```java
Order-xxxxx007 开始处理
 ...
Order-xxxxx007 风险评估 ....
...
Order-xxxxx007 调用Third-Api-X接口
```

没有`Exception` ，惊喜不？意外不？果断打开吃饭工具IDEA,定位到代码

```java
ExecutorService executorService = Executors.newCachedThreadPool();      
        
        /*
        * ...此处略去n行bug
        * */
executorService.submit(() -> {
	//调用三方api X
	invokeThirdApi();
});
//....
```

这？黑人抬棺的节奏，这什么鬼？线程跑到一半死了？所有流程中断了？怎么会这样。。一幅动图渐渐渗透到我的脑海里...

<img src="https://i.loli.net/2020/04/11/1APBuLXqSk49Ggh.gif" alt="黑人" style="zoom:67%;" />

再看下invokeThirdAPi，发现里面就postJson发送了一个http请求，不应该啊？

```java
public static String postJson(String url,String data){
        HttpRequest httpRequest = cn.hutool.http.HttpUtil.createPost(url);
        httpRequest.header(Header.CONTENT_TYPE,"application/json;charset=UTF-8").body(data);
        HttpResponse response = httpRequest.execute();
        return response.body();
}
```

咦？怎么http调用没有返回结果？http请求没有设置超时时间，第三方接口因为某些异常原因一直没有返回结果！线程一直被占用阻塞！

## 初步得出的错误结论

仔细看了一下中的线程id： `[04-09 18:40:15,477] INFO [Thread-4-pool-1329] ...`

再看看 一小时前的（一小时前发版服务重启过）``[04-09 17:00:15,472] INFO [Thread-4-pool-10] ...``

一个小时线程id从10到1300+，按照现在的qps，50个线程都吃不完啊，阻塞这么多线程？恐慌，担忧😟，不知道黑人抬棺费用高不高，给自己也预约一个吧....

##真正的原因

`.......few minutes later........`

正在收拾行李的我（别问，问就是跑路），不对啊，我用的是 `Executors.newCachedThreadPool()`,默认情况下，线程空闲了会销毁的吧？正常销毁了再new出来，线程id是会自增的吧？？？上源码！

```java
public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
}
```

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
}
```

```java
public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler) {
        if (corePoolSize < 0 ||
            maximumPoolSize <= 0 ||
            maximumPoolSize < corePoolSize ||
            keepAliveTime < 0)
            throw new IllegalArgumentException();
        if (workQueue == null || threadFactory == null || handler == null)
            throw new NullPointerException();
        this.corePoolSize = corePoolSize;
        this.maximumPoolSize = maximumPoolSize;
        this.workQueue = workQueue;
        this.keepAliveTime = unit.toNanos(keepAliveTime);
        this.threadFactory = threadFactory;
        this.handler = handler;
}
```

这？这些参数好熟悉又好陌生啊。。。好吧，拿起的我小本本找找...嗯，有了。

- int corePoolSize  该线程池中**核心线程数最大值**

  > 核心线程，无论有没有任务，都不会被销毁（国企待遇啊），当然，设置了allowCoreThreadTimeOut(true)之后，也是可以被销毁的。

- int maximumPoolSize 该线程池中**线程总数最大值** 

  > 该值等于核心线程数量 + 非核心线程数量。

- long keepAliveTime **非核心线程闲置超时时长**
- TimeUnit unit  keepAliveTime的单位

- BlockingQueue workQueue 阻塞队列，维护着**等待执行的Runnable任务对象**,也就是任务队列。

  有以下几种实现

  - LinkedBlockingQueue
  - ArrayBlockingQueue
  - SynchronousQueue  同步队列，内部容量为0，每个put操作必须等待一个take操作，反之亦然
  - DelayQueue 延迟队列，该队列中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素 

- ThreadFactory threadFactory 创建线程的工厂 ，用于批量创建线程。

- RejectedExecutionHandler handler 拒绝策略，此处不展开

更多关于线程池的资料 看这里 [gitbook-线程池](https://redspider.gitbook.io/concurrent/di-san-pian-jdk-gong-ju-pian/12)

然后，咱们再回头看看 newCachedThreadPool，这是人干的事吗？

- 核心线程数为 0
- 非核心线程数不限，但存活时间  60 秒

那么上面这些参数到底是怎么运转的呢？看图

<img src="https://i.loli.net/2020/04/11/e3y69UpSxah1nu4.jpg" alt="流程图" style="zoom:80%;" />



再看下 ThreadFactory `Executors.defaultThreadFactory()`

```java
private static class DefaultThreadFactory implements ThreadFactory {
        private static final AtomicInteger poolNumber = new AtomicInteger(1);
        private final ThreadGroup group;
        private final AtomicInteger threadNumber = new AtomicInteger(1);
        private final String namePrefix;

        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())
                t.setDaemon(false);
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY);
            return t;
        }
    }
```

`Thread t = new Thread(group, r,namePrefix + threadNumber.getAndIncrement(),0);`

所以：正常的销毁创建也会id自增，这我就放心了。。。

### 修复

- 设置http超时

- 更换自定义线程池（定制化的才是最好的）

  ThreadPoolTaskExecutor 是spring提供的，是对 ThreadPoolExecutor的封装。

  ```java
  @Bean("thirdApiExecutor")
  public ThreadPoolTaskExecutor thirdApiExecutor(){
      ThreadPoolTaskExecutor taskExecutor = new ThreadPoolTaskExecutor();
      taskExecutor.setCorePoolSize(20);
      taskExecutor.setMaxPoolSize(40);
      taskExecutor.setKeepAliveSeconds(300);
      taskExecutor.setAllowCoreThreadTimeOut(true);
      taskExecutor.setWaitForTasksToCompleteOnShutdown(true);
      taskExecutor.setAwaitTerminationSeconds(10);
      taskExecutor.setQueueCapacity(200);
      taskExecutor.setThreadNamePrefix("ThirdApi-Executor-");
      taskExecutor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
      return taskExecutor;
  }
  ```

  

## 小结

- 线程内的 http等IO阻塞行为一定要设置timeout。
- 自定义线程池，根据业务场景和特点设置参数。
- 平时基本功要扎实，不然翻小本本也挺浪费时间，特别是生产环境出问题了，都是比较急的，没有多少时间给你去学习和查找资料，所以平时要多积累。

## 到这就完了？就这？

**到现在，我还有一个最大的疑问：如何确定目前生产环境是否存在被永久阻塞在IO上的线程？**

## Java线程状态

到这里我们就需要知道 Java线程在生命周期中存在哪些状态了

```java
// Thread.State 源码
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
}
```

具体每个状态的意义以及状态的转换场景看这里 [Java线程状态及主要转化方法](https://redspider.gitbook.io/concurrent/di-yi-pian-ji-chu-pian/4)

## 复现事故现场

### 再次分析

文中也多次提到”阻塞“关键字，那永久等待Http 响应的那个线程应该是什么状态呢？？首先猜想是`Blocked`,但按照理论来讲，只有等待锁的线程才会是Blocked状态，使用排除法，我最终猜想它是`Runnable`状态，下面就来证实一下。

### 代码复现

```java
@Test
    public void testThreadBlock() throws InterruptedException {
        ExecutorService executorService = Executors.newCachedThreadPool();

        executorService.submit(() -> {
            //模拟调用三方api
            invokeThirdApi();
        });

        //添加一个其他任务
        executorService.submit(()->{
           while (true){
               log.info("线程 {} 正在干活",Thread.currentThread().getName());
               Thread.sleep(3000);
           }
        });
				//让主线程在此休息
        Thread.sleep(Integer.MAX_VALUE);
    }

    public String invokeThirdApi(){
        InputStream is = System.in;
        Scanner scanner = new Scanner(is);
        String threadName = Thread.currentThread().getName();
        log.info("{} IO输入开始",threadName);
        String input =  scanner.next();
        log.info("{} IO输入结束：{}",threadName,input);
        return null;
    }
```

简单解释一下上面程序：

- 基于junit测试的方式

- testThreadBlock方法中定义一个线程池`newCachedThreadPool`,然后向线程池提交了两个任务
  - 第一个是一次性调用三方接口的任务
  - 第二个是一个重复执行的任务
- 调用三方接口，这里使用`Scanner`监听终端输入来模拟（当然也可以自己本地启一个测试项目，暴露一个接口，然后接口逻辑中打个断点，这样在http请求的时候被断点卡住就行了）

看一下程序运行的日志：

```verilog
2020-04-11 19:14:56.852  INFO 27281 --- [pool-1-thread-1] com.eqshen.springdemo.ThreadPoolTest     : pool-1-thread-1 IO输入开始
2020-04-11 19:14:56.850  INFO 27281 --- [pool-1-thread-2] com.eqshen.springdemo.ThreadPoolTest     : 线程 pool-1-thread-2 正在干活
2020-04-11 19:14:59.858  INFO 27281 --- [pool-1-thread-2] com.eqshen.springdemo.ThreadPoolTest     : 线程 pool-1-thread-2 正在干活
2020-04-11 19:15:02.861  INFO 27281 --- [pool-1-thread-2] com.eqshen.springdemo.ThreadPoolTest     : 线程 pool-1-thread-2 正在干活
2020-04-11 19:15:05.865  INFO 27281 --- [pool-1-thread-2] com.eqshen.springdemo.ThreadPoolTest     : 线程 pool-1-thread-2 正在干活
2020-04-11 19:15:08.867  INFO 27281 --- [pool-1-thread-2] com.eqshen.springdemo.ThreadPoolTest     : 线程 pool-1-thread-2 正在干活
2020-04-11 19:15:11.870  INFO 27281 --- [pool-1-thread-2] com.eqshen.springdemo.ThreadPoolTest     : 线程 pool-1-thread-2 正在干活
2020-04-11 19:15:14.875  INFO 27281 --- [pool-1-thread-2] com.eqshen.springdemo.ThreadPoolTest     : 线程 pool-1-thread-2 正在干活
```

可以发现thread-1启动之后直接就被阻塞了

### 使用jstack

下面就使用jstack查看一下线程的真正状态

使用`jps`命令查看进程id

```shell
27281 JUnitStarter
20500
27285 Jps
26586 RemoteMavenServer
27279 Launcher
```

然后执行命令 `jstack 27281 > test.jstack`

到 test.jstack中间中按照关键字搜索 `pool-1-thread-1`会找到如下内容：

```java
"pool-1-thread-1" #19 prio=5 os_prio=31 cpu=2.46ms elapsed=215.35s tid=0x00007fddfabf0000 nid=0x6703 runnable  [0x0000700007286000]
   java.lang.Thread.State: RUNNABLE
        at java.io.FileInputStream.readBytes(java.base@11.0.2/Native Method)
        at java.io.FileInputStream.read(java.base@11.0.2/FileInputStream.java:279)
        at java.io.BufferedInputStream.read1(java.base@11.0.2/BufferedInputStream.java:290)
        at java.io.BufferedInputStream.read(java.base@11.0.2/BufferedInputStream.java:351)
        - locked <0x0000000700133178> (a java.io.BufferedInputStream)
        at sun.nio.cs.StreamDecoder.readBytes(java.base@11.0.2/StreamDecoder.java:284)
        at sun.nio.cs.StreamDecoder.implRead(java.base@11.0.2/StreamDecoder.java:326)
        at sun.nio.cs.StreamDecoder.read(java.base@11.0.2/StreamDecoder.java:178)
        - locked <0x000000070aa0fef0> (a java.io.InputStreamReader)
        at java.io.InputStreamReader.read(java.base@11.0.2/InputStreamReader.java:185)
        at java.io.Reader.read(java.base@11.0.2/Reader.java:189)
        at java.util.Scanner.readInput(java.base@11.0.2/Scanner.java:882)
        at java.util.Scanner.next(java.base@11.0.2/Scanner.java:1476)
        at com.eqshen.springdemo.ThreadPoolTest.invokeThirdApi(ThreadPoolTest.java:44)
        at com.eqshen.springdemo.ThreadPoolTest.lambda$testThreadBlock$0(ThreadPoolTest.java:25)
        at com.eqshen.springdemo.ThreadPoolTest$$Lambda$767/0x0000000800568440.run(Unknown Source)
        at java.util.concurrent.Executors$RunnableAdapter.call(java.base@11.0.2/Executors.java:515)
        at java.util.concurrent.FutureTask.run$$$capture(java.base@11.0.2/FutureTask.java:264)
        at java.util.concurrent.FutureTask.run(java.base@11.0.2/FutureTask.java)
        at java.util.concurrent.ThreadPoolExecutor.runWorker(java.base@11.0.2/ThreadPoolExecutor.java:1128)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(java.base@11.0.2/ThreadPoolExecutor.java:628)
        at java.lang.Thread.run(java.base@11.0.2/Thread.java:834)
```

可以看出线程状态和我们猜想的是一致的：RUNNABLE

### 递归懵逼

那现在如何在茫茫程海中定位到这种问题？？？

> "pool-1-thread-1" #19 prio=5 os_prio=31 cpu=2.46ms elapsed=215.35s tid=0x00007fddfabf0000 nid=0x6703 runnable  [0x0000700007286000]

如果多次执行jstack，发现 `cpu=2.46ms` 值是不变的（IO阻塞了，永远不会再被分配到cpu时间片，当然不变了），这也是我目前能想到的办法☹️