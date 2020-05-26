---
title: Java父子线程的ThreadLocal数据传递问题
date: 2020-05-26 15:55:57
tags:
    - 多线程
    - ThreadLocal

categories:
  - ['Java基础']

---


## JAVA父子线程之间如何传递数据

### 线程之间通信的方式

常见的方式由以下几种

- 同步锁
- 通知等待机制：notify/wait
- 信号量 CountdownLatch/CyclicBarrier/Semaphore
- 管道PipedWriter/ PipedReader/ PipedOutputStream/ PipedInputStream

当然，上面这些跟今天要讨论的东西没关系。

### 父子线程ThreadLocal问题

首先看一段代码，提出问题:

```java
public static void main(String[] args) throws InterruptedException {
        ThreadLocal<String> threadLocal = ThreadLocal.withInitial(()->{return "hello world";});
        threadLocal.set("modify hello world");
        System.out.println(Thread.currentThread().getId()+" get:"+threadLocal.get());
        for (int i = 0; i <10 ; i++) {
            new Thread(() -> System.out.println("Thread:"+ Thread.currentThread().getId()+" get:"+ threadLocal.get())).start();
        }
        Thread.sleep(100);
        System.out.println("====end======");
}
```

代码:父线程（main）线程定义了一个ThreadLocal变量，并定义了初始值。然后，启动多个子线程，在子线程中执行 `threadLocal.get()`那么根据threadLocal的特性，会拿到什么值呢？看输出：

```verilog
1 get:modify hello world
Thread:13 get:hello world
Thread:14 get:hello world
Thread:15 get:hello world
Thread:16 get:hello world
Thread:17 get:hello world
Thread:18 get:hello world
Thread:19 get:hello world
Thread:21 get:hello world
Thread:20 get:hello world
Thread:22 get:hello world
====end======
```

这边我们可以得出两个结论

1. ThreadLocal.withInitial设置的初始值其实就是默认值，所有线程都会影响到（看源码，SuppliedThreadLocal重写了initialValue方法 ）。
2. ThreadLocal实现数据的线程隔离。

那么有没有办法可以实现兄弟线程之间数据隔离，但父子线程之间数据互通呢？

### InheritableThreadLocal

InheritableThreadLocal可以实现数据的继承，但是继承之后，子线程修改threadLocal中的值是无法传递到父线程的，即这种数据传递是单向的。

```java
ThreadLocal<String> threadLocal = new InheritableThreadLocal();
        threadLocal.set("modify hello world");
        System.out.println(Thread.currentThread().getId()+" get:"+threadLocal.get());
        for (int i = 0; i <10 ; i++) {
            new Thread(() -> {
                System.out.println("Thread:"+ Thread.currentThread().getId()+" get:"+ threadLocal.get());
            }).start();
        }
        Thread.sleep(100);
        System.out.println("====end======");
```

输出

```verilog
1 get:modify hello world
Thread:13 get:modify hello world
Thread:14 get:modify hello world
Thread:15 get:modify hello world
Thread:16 get:modify hello world
Thread:17 get:modify hello world
Thread:18 get:modify hello world
Thread:19 get:modify hello world
Thread:20 get:modify hello world
Thread:21 get:modify hello world
Thread:22 get:modify hello world
====end======
```

那么这是怎么做到的呢？

查看源码，发现InheritableThreadLocal源码很简单，继承了ThreadLocal，重写了三个方法

```java
public class InheritableThreadLocal<T> extends ThreadLocal<T> {
    /**
     * Computes the child's initial value for this inheritable thread-local
     * variable as a function of the parent's value at the time the child
     * thread is created.  This method is called from within the parent
     * thread before the child is started.
     * <p>
     * This method merely returns its input argument, and should be overridden
     * if a different behavior is desired.
     *
     * @param parentValue the parent thread's value
     * @return the child thread's initial value
     */
    protected T childValue(T parentValue) {
        return parentValue;
    }

    /**
     * Get the map associated with a ThreadLocal.
     *
     * @param t the current thread
     */
    ThreadLocalMap getMap(Thread t) {
       return t.inheritableThreadLocals;
    }

    /**
     * Create the map associated with a ThreadLocal.
     *
     * @param t the current thread
     * @param firstValue value for the initial entry of the table.
     */
    void createMap(Thread t, T firstValue) {
        t.inheritableThreadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

重点关注 childValue 这个方法，注释大概意思是：这个方法会在父线程创建子线程的时候被调用，子线程创建的时候，inheritableThreadLocals这个map要把父线程的inheritableThreadLocals 中的所有k,v拷贝过来，拷贝的时候，每个entry的value是否需要重新计算呢？childValue就是负责重新计算的，只不过默认的实现是返回传入参数，do nothing。

继续追踪一下调用链路，验证上面的说法，看看childValue被谁调用了

```java
/**
         * Construct a new map including all Inheritable ThreadLocals
         * from given parent map. Called only by createInheritedMap.
         *
         * @param parentMap the map associated with parent thread.
         */
        private ThreadLocalMap(ThreadLocalMap parentMap) {
            Entry[] parentTable = parentMap.table;
            int len = parentTable.length;
            setThreshold(len);
            table = new Entry[len];

            for (Entry e : parentTable) {
                if (e != null) {
                    @SuppressWarnings("unchecked")
                    ThreadLocal<Object> key = (ThreadLocal<Object>) e.get();
                    if (key != null) {
                        Object value = key.childValue(e.value);
                        Entry c = new Entry(key, value);
                        int h = key.threadLocalHashCode & (len - 1);
                        while (table[h] != null)
                            h = nextIndex(h, len);
                        table[h] = c;
                        size++;
                    }
                }
            }
        }
```

这个是ThreadLocal类的一个private构造方法，从功能上看，就是拷贝传进来的threadLocalMap的内容，从注释上看，这个构造函数的调用是：`Called only by createInheritedMap`,看一下上一层调用的地方

```java
static ThreadLocalMap createInheritedMap(ThreadLocalMap parentMap) {
        return new ThreadLocalMap(parentMap);
}
```

继续追上层调用，就会跳到Thread的源码，如下：

```java
private Thread(ThreadGroup g, Runnable target, String name,
                   long stackSize, AccessControlContext acc,
                   boolean inheritThreadLocals) {
        if (name == null) {
            throw new NullPointerException("name cannot be null");
        }

        this.name = name;

        Thread parent = currentThread();
        SecurityManager security = System.getSecurityManager();
        if (g == null) {
            /* Determine if it's an applet or not */

            /* If there is a security manager, ask the security manager
               what to do. */
            if (security != null) {
                g = security.getThreadGroup();
            }

            /* If the security manager doesn't have a strong opinion
               on the matter, use the parent thread group. */
            if (g == null) {
                g = parent.getThreadGroup();
            }
        }

        /* checkAccess regardless of whether or not threadgroup is
           explicitly passed in. */
        g.checkAccess();

        /*
         * Do we have the required permissions?
         */
        if (security != null) {
            if (isCCLOverridden(getClass())) {
                security.checkPermission(
                        SecurityConstants.SUBCLASS_IMPLEMENTATION_PERMISSION);
            }
        }

        g.addUnstarted();

        this.group = g;
        this.daemon = parent.isDaemon();
        this.priority = parent.getPriority();
        if (security == null || isCCLOverridden(parent.getClass()))
            this.contextClassLoader = parent.getContextClassLoader();
        else
            this.contextClassLoader = parent.contextClassLoader;
        this.inheritedAccessControlContext =
                acc != null ? acc : AccessController.getContext();
        this.target = target;
        setPriority(priority);
  			//注意看这里
        if (inheritThreadLocals && parent.inheritableThreadLocals != null)
            this.inheritableThreadLocals =
                ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
        /* Stash the specified stack size in case the VM cares */
        this.stackSize = stackSize;

        /* Set thread ID */
        this.tid = nextThreadID();
    }
```

这是一个线程的构造函数，也是证明了我们上面的说法，子线程被创建的时候会拷贝一份父线程的inheritableThreadLocal变量到自己的InheritableThreadLocal变量中。

### InheritableThreadLocal存在的问题

所以这样就结束了？不，且看下面一段代码

```java
ThreadLocal<String> threadLocal = new InheritableThreadLocal();
        threadLocal.set("default value set by main thread");
        ExecutorService executorService = Executors.newFixedThreadPool(1);
        executorService.submit(()->{
            System.out.println("Pool-Thread:" + Thread.currentThread().getId() + " get 1:" + threadLocal.get());
        });
        //主线程修改一次
        threadLocal.set("updated value set by main thread");

        executorService.submit(()->{
            System.out.println("Pool-Thread:" + Thread.currentThread().getId() + " get 2:" + threadLocal.get());
        });

        executorService.shutdown();
```

日志输出：

```
Pool-Thread:13 get 1:default value set by main thread
Pool-Thread:13 get 2:default value set by main thread
```

可以发现，如果是使用线程池，线程被复用，那么父线程就无法修改了。经过了上面对InheritableThreadLocal源码的分析，我们很容易找到原因：Thread的构造函数只会调用一次

**在线程池化的场景中，InheritableThreadLocal不再满足需求**

典型的使用场景

- 分布式跟踪系统
- 日志收集记录系统上下文
- Session级别的cache
- 应用容器或上层框架跨应用代码给下层`SDK`传递信息
- ...

### TransmittableThreadLocal

这个是阿里开源的一种方案，它继承了InheritableThreadLocal，具体的资料可以参考  [Transmittable ThreadLocal(TTL)](https://github.com/alibaba/transmittable-thread-local) 。

先看如何使用,首先是引用jar包，有多种方式，此处采用maven

```
<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>transmittable-thread-local</artifactId>
    <version>2.11.4</version>
</dependency>
```

然后是同样的逻辑代码：

```java
TransmittableThreadLocal<String> context = new TransmittableThreadLocal<String>();
        context.set("value-set-in-parent");

        ExecutorService executorService = Executors.newFixedThreadPool(1);
        // 额外的处理，生成修饰了的对象executorService
        executorService = TtlExecutors.getTtlExecutorService(executorService);

        executorService.submit(()->{
            System.out.println("Pool-Thread:" + Thread.currentThread().getId() + " get 1:" + context.get());
        });

        context.set("value-update-in-parent");

        executorService.submit(()->{
            System.out.println("Pool-Thread:" + Thread.currentThread().getId() + " get 2:" + context.get());
        });

        executorService.shutdown();
```

程序输出:

```
Pool-Thread:14 get 1:value-set-in-parent
Pool-Thread:14 get 2:value-update-in-parent
```

关于TransmittableThreadLocal原理是啥，还需要仔细阅读源码，这边暂时`打个欠条`，等我多梳理几遍再补上。可以参考 [开发者文档](https://github.com/alibaba/transmittable-thread-local/blob/master/docs/developer-guide.md) 还有这个 [应用场景和设计思考](https://github.com/alibaba/transmittable-thread-local/issues/123)


