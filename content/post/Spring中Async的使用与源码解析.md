---
title: "Spring中Async的使用与源码解析"
date: 2020-11-26T16:39:17+08:00
description: "Article description."
draft: true
toc: TRUE
categories:
  - SpringBoot
  - 线程池
tags:
  - 线程池
  - 异步
  - SpringBoot
---

# 背景介绍

对于异步方法调用，从Spring3开始提供了@Async注解，该注解可以被标注在方法上，以便异步地调用该方法。调用者将在调用时立即返回，方法的实际执行将提交给Spring TaskExecutor的任务中，由指定的线程池中的线程执行。

# 常见的场景

- 系统日志记录
- 耗时任务的执行



# 使用方法

1.启动类增加@EnableAsync注解(since Spring 3.1)

```java
@EnableAsync
@SpringBootApplication
public class SpringBootDemoAsyncApplication {

    public static void main(String[] args) {
        SpringApplication.run(SpringBootDemoAsyncApplication.class, args);
    }

}
```

2.如有需要，可以自定义线程池

```java
@Configuration
public class ExecutorConfiguration {

    /**
     * 配置应用访问日志专用线程池
     * @return
     */
    @Bean(name = "sodAppLogAsyncExecutor")
    public ThreadPoolTaskExecutor asyncExecutor() {
        ThreadPoolTaskExecutor threadPool = new ThreadPoolTaskExecutor();
        threadPool.setThreadNamePrefix("drs-sodAppLog-");
        threadPool.setCorePoolSize(3);
        threadPool.setMaxPoolSize(4);
        threadPool.setKeepAliveSeconds(60);
        threadPool.setQueueCapacity(11);
        threadPool.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
        //优雅关闭
        threadPool.setWaitForTasksToCompleteOnShutdown(true);
        threadPool.setAwaitTerminationSeconds(60 * 15);
        return threadPool;
    }
}
```



3.在需要使用异步的方法上添加@Async注解，可以通过value属性指定线程池,返回值支持void、Future、ListenableFuture、CompletableFuture，如果不指定value，那么采用默认线程池

```java
    /**
     * 模拟5秒的异步任务
     */
    @Async
    public Future<Boolean> asyncTask1() throws InterruptedException {
        doTask("asyncTask1", 5);
        return new AsyncResult<>(Boolean.TRUE);
    }

    /**
     * 模拟业务代码
     * @param taskName
     * @param time
     */
    private void doTask(String taskName, Integer time) {
        log.info("{}模拟执行【{}】,线程内存地址:{}", taskName, Thread.currentThread().getName(), UnsafeUtil.addressOf(Thread.currentThread()));
    }
```

# Spring实现的线程池

![image-2020112616444257](https://gitee.com/1162492411/pic/raw/master/Spring线程池类图.png)

- SimpleAsyncTaskExecutor：默认线程池，每次调用都启动一个新线程(并不会复用线程池已有线程),支持对并发总数设限（ConcurrencyLimit，默认-1不限制，0不允许），当超过线程并发总数限制时，阻塞新的调用
- ThreadPoolTaskExecutor:对JDK的ThreadPoolExecutor的封装，SpringBoot通过TaskExecutionAutoConfiguration自动装配了一个名为applicationTaskExecutor的ThreadPoolTaskExecutor

```java
@ConditionalOnClass(ThreadPoolTaskExecutor.class)
@Configuration
@EnableConfigurationProperties(TaskExecutionProperties.class)
public class TaskExecutionAutoConfiguration {
	@Bean
	@ConditionalOnMissingBean
	public TaskExecutorBuilder taskExecutorBuilder() {
		TaskExecutionProperties.Pool pool = this.properties.getPool();
		TaskExecutorBuilder builder = new TaskExecutorBuilder();
		builder = builder.queueCapacity(pool.getQueueCapacity());
		builder = builder.corePoolSize(pool.getCoreSize());
		builder = builder.maxPoolSize(pool.getMaxSize());
		builder = builder.allowCoreThreadTimeOut(pool.isAllowCoreThreadTimeout());
		builder = builder.keepAlive(pool.getKeepAlive());
		builder = builder.threadNamePrefix(this.properties.getThreadNamePrefix());
		builder = builder.customizers(this.taskExecutorCustomizers);
		builder = builder.taskDecorator(this.taskDecorator.getIfUnique());
		return builder;
	}

	@Lazy
	@Bean(name = APPLICATION_TASK_EXECUTOR_BEAN_NAME)
	@ConditionalOnMissingBean(Executor.class)
	public ThreadPoolTaskExecutor applicationTaskExecutor(TaskExecutorBuilder builder) {
		return builder.build();
	}
}
```



## SimpleAsyncTaskExecutor

属性列表

- Daemon:是否为守护线程，默认false
- ThreadPriority:线程优先级,默认5
- ThreadNamePrefix:线程名前缀，默认"SimpleAsyncTaskExecutor"
- ConcurrencyLimit:并发上限,默认-1不限制，0表示不允许并发？？？？



## ThreadPoolTaskExecutor

属性列表

- CorePoolSize：线程池创建时候初始化的线程数,默认1
- MaxPoolSize：线程池最大的线程数，只有在缓冲队列满了之后才会申请超过核心线程数的线程，默认Integer.MAX
- QueueCapacity：用来缓冲执行任务的队列的队列大小，默认Integer.MAX
- KeepAliveSeconds：线程的空闲时间，单位/s，当超过了核心线程出之外的线程在空闲时间到达之后会被销毁,默认60
- ThreadNamePrefix：线程池中线程名的前缀，继承自父类ExecutorConfigurationSupport，默认是BeanName/方法名
- RejectedExecutionHandler：线程池对拒绝任务的处理策略，自父类ExecutorConfigurationSupport,（策略为JDK ThreadPoolExecutor自带）
  - AbortPolicy：默认策略，直接抛出异常 RejectedExecutionException
  - CallerRunsPolicy：直接在 execute 方法的调用线程中运行被拒绝的任务；如果执行程序已关闭，则会丢弃该任务
  - DiscardPolicy：该策略直接丢弃
  - DiscardOldestPolicy：该策略会先将最早入队列的未执行的任务丢弃掉，然后尝试执行新的任务。如果执行程序已关闭，则会丢弃该任务
- waitForTasksToCompleteOnShutdown：关闭程序时是否等待任务执行完毕，继承自父类ExecutorConfigurationSupport，默认false表示中断正在执行的任务，清空队列
- awaitTerminationSeconds：关闭程序时的等待时间，需配合waitForTasksToCompleteOnShutdown使用，继承自父类ExecutorConfigurationSupport，默认0

## 线程处理流程

```java
    /**
     * Executes the given task sometime in the future.  The task
     * may execute in a new thread or in an existing pooled thread.
     *
     * If the task cannot be submitted for execution, either because this
     * executor has been shutdown or because its capacity has been reached,
     * the task is handled by the current {@code RejectedExecutionHandler}.
     *
     * @param command the task to execute
     * @throws RejectedExecutionException at discretion of
     *         {@code RejectedExecutionHandler}, if the task
     *         cannot be accepted for execution
     * @throws NullPointerException if {@code command} is null
     */
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();
        /*
         * Proceed in 3 steps:
         *
         * 1. If fewer than corePoolSize threads are running, try to
         * start a new thread with the given command as its first
         * task.  The call to addWorker atomically checks runState and
         * workerCount, and so prevents false alarms that would add
         * threads when it shouldn't, by returning false.
         *
         * 2. If a task can be successfully queued, then we still need
         * to double-check whether we should have added a thread
         * (because existing ones died since last checking) or that
         * the pool shut down since entry into this method. So we
         * recheck state and if necessary roll back the enqueuing if
         * stopped, or start a new thread if there are none.
         *
         * 3. If we cannot queue task, then we try to add a new
         * thread.  If it fails, we know we are shut down or saturated
         * and so reject the task.
         */
        int c = ctl.get();
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        else if (!addWorker(command, false))
            reject(command);
    }
```



- 如果此时线程池中的数量**小于**corePoolSize，即使线程池中的线程都处于空闲状态，也要**创建新**的线程来处理被添加的任务。
- 如果此时线程池中的数量**等于** corePoolSize，但是缓冲队列 workQueue未满，那么任务被放入缓冲队列。
- 如果此时线程池中的数量**大于**corePoolSize，缓冲队列workQueue满，并且线程池中的数量小于maxPoolSize，那么建新的线程来处理被添加的任务。
- 如果此时线程池中的数量**大于**corePoolSize，缓冲队列workQueue满，并且线程池中的数量等于maxPoolSize，那么通过handler所指定的策略来处理此任务。（也就是：处理任务的优先级为：核心线程corePoolSize、任务队列workQueue、最大线程 maxPoolSize，如果三者都满了，使用handler处理被拒绝的任务）
- 当线程池中的线程数量**大于**corePoolSize时，如果某线程空闲时间超过keepAliveTime，线程将被终止。这样，线程池可以动态的调整池中的线程数



# 自定义线程池



自定义线程池有如下模式：

- 配置由自定义的TaskExecutor
- 重新实现接口AsyncConfigurer
- 继承AsyncConfigurerSupport

## 方式一：自定义TaskExecutor

```java
@Configuration
public class ExecutorConfiguration {

    /**
     * 配置应用访问日志专用线程池
     * @return
     */
    @Bean(name = "sodAppLogAsyncExecutor")
    public ThreadPoolTaskExecutor asyncExecutor() {
        ThreadPoolTaskExecutor threadPool = new ThreadPoolTaskExecutor();
        threadPool.setThreadNamePrefix("drs-sodAppLog-");
        threadPool.setCorePoolSize(3);
        threadPool.setMaxPoolSize(4);
        threadPool.setKeepAliveSeconds(60);
        threadPool.setQueueCapacity(11);
        threadPool.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
        //优雅关闭
        threadPool.setWaitForTasksToCompleteOnShutdown(true);
        threadPool.setAwaitTerminationSeconds(60 * 15);
        return threadPool;
    }
}
```



## 方式二：实现AsyncConfigurer

```java
/**
 * 自定义线程池方法二：自定义类，配置默认Executor与默认异步异常处理器
 * @author zyg
 */
@Configuration
public class CusAsyncConfigure implements AsyncConfigurer {

    /**
     * 配置默认Executor
     */
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor threadPool = new ThreadPoolTaskExecutor();
        threadPool.setThreadNamePrefix("cus-async-configure-");
        threadPool.setCorePoolSize(2);
        threadPool.setMaxPoolSize(3);
        threadPool.setKeepAliveSeconds(60);
        threadPool.setQueueCapacity(5);
        threadPool.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
        return threadPool;
    }

    /**
     * 配置默认异步异常处理器
     */
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CusAsyncUncaughtExceptionHandler();
    }
}
```

（原理是ProxyAsyncConfiguration的父类AbstractAsyncConfiguration的setConfigurers(Collection<AsyncConfigurer>)中执行了AsyncConfigurer的方法来配置Executor与AsyncUncaughtExceptionHandler）

## 方式三：继承AsyncConfigurerSupport

```java
/**
 * 自定义线程池方法三:继承AsyncConfigurerSupport,重写getAsyncExecutor与getAsyncUncaughtExceptionHandler
 * @author zyg
 */
@Configuration
public class CusAsyncConfigurerSupport extends AsyncConfigurerSupport {

    /**
     * 配置默认Executor
     */
    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor threadPool = new ThreadPoolTaskExecutor();
        threadPool.setThreadNamePrefix("cus-async-configure-support-");
        threadPool.setCorePoolSize(2);
        threadPool.setMaxPoolSize(3);
        threadPool.setKeepAliveSeconds(60);
        threadPool.setQueueCapacity(5);
        threadPool.setRejectedExecutionHandler(new ThreadPoolExecutor.DiscardPolicy());
        return threadPool;
    }

    /**
     * 配置默认异步异常处理器
     */
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new CusAsyncUncaughtExceptionHandler();
    }
}
```

（原理是AsyncConfigurerSupport的父类是AsyncConfigurer）

# 异常处理

如果任务的返回类型是Future，那么将直接抛出异常，否则异常由AsyncUncaughtExceptionHandler的handleUncaughtException()进行处理，Spring自4.1默认提供了SimpleAsyncUncaughtExceptionHandler，该类处理异常的逻辑是通过日志打印错误，如有需要可以自定义类继承AsyncUncaughtExceptionHandler，复写其handleUncaughtException()方法。

```java
/**
 * 自定义线程池方法二：自定义默认异步异常处理器
 * @author zyg
 */
@Component
public class CusAsyncUncaughtExceptionHandler implements AsyncUncaughtExceptionHandler {
    private Logger logger = LoggerFactory.getLogger(this.getClass());

    @Override
    public void handleUncaughtException(Throwable ex, Method method, Object... params) {
        logger.error("自定义异步异常处理器捕捉到异常，",ex);
    }
}
```



# @EnableAsync加载流程

## 前置知识点

- @Import注解的作用
- BeanPostProcessor在Spring中的作用
- Aware类接口在Spring中的作用
- 切面与通知的概念与作用



## 代码分析：

- @EnableAsync中Import了AsyncConfigurationSelector；
- AsyncConfigurationSelector的作用是通过配置确定是调用ProxyAsyncConfiguration还是AspectJ的AspectJAsyncConfiguration；
- 在ProxyAsyncConfiguration的asyncAdvisor()方法可以看到，其中定义了后置处理器AsyncAnnotationBeanPostProcessor
- AsyncAnnotationBeanPostProcessor直接或间接实现了BeanFactoryAware、BeanPostProcessor两个接口，既然AsyncAnnotationBeanPostProcessor实现了BeanFactoryAware，那么就会执行setBeanFactory(BeanFactory)方法,该方法中设置了切面AsyncAnnotationAdvisor
  - 切面中定义了切点：类上标注@Async、@Asynchronous注解的切点与在方法上标注@Async、@Asynchronous注解的切点
  - 切面中定义了通知：通知Executor与SimpleAsyncUncaughtExceptionHandler，
  - 通知具体的实现类为AnnotationAsyncExecutionInterceptor，它的父类AsyncExecutionInterceptor进行了实际的通知处理操作



## 配置默认Executor

在生成切面AsyncAnnotationAdvisor对象时，生成了AnnotationAsyncExecutionInterceptor对象，调用了AnnotationAsyncExecutionInterceptor的configure(Supplier<Executor>,Supplier<AsyncUncaughtExceptionHandler>)方法,在该方法中，调用了getDefaultExecutor(BeanFactory)来寻找默认Executor，查找Executor的优先级如下：

- 从BeanFactory中查找类型为TaskExecutor的对象
- 从BeanFactory中查找类型为Executor、Bean名称为taskExecutor的对象
- 如果上述步骤中找不到，那么子类AsyncExecutionInterceptor中生成SimpleAsyncTaskExecutor对象



# 通知的处理



- 通过determineAsyncExecutor(Method)方法查找AsyncExecutor
- 包装一下任务，当任务出现异常时调用AsyncUncaughtExceptionHandler的handleUncaughtException()处理异常
- 调用AsyncExecutor的submit()/submitListenable()/CompletableFuture.supplyAsync()等方法提交任务



## 查找AsyncExecutor

AsyncExecutionAspectSupport的determineAsyncExecutor(Method)中查找了AsyncEexcutor，逻辑如下

- 首先尝试从成员变量Map<Method, AsyncTaskExecutor> executors查找是否存在，如果存在则返回
- 然后从AsyncExecutionAspectSupport.getExecutorQualifier()获取专属于该Method的AsyncExecutor的Bean名称，如果存在，则向BeanFactory获取类型为Executor、Bean名称为该名称的Executor并返回
- 从成员变量SingletonSupplier<Executor>获取，如果存在则返回
- 如果经过上述几步查找仍然无法找到那么就返回空
- 如果经过上述几步找到了Executor，判断Executor的类型
  - 如果是AsyncListenableTaskExecutor，将其强制转换为AsyncListenableTaskExecutor后放入到成员变量executors中
  - 如果不是AsyncListenableTaskExecutor，通过TaskExecutorAdapter包装一个concurrentExecutor然后放入到成员变量executors中



# 异步事务

在@Async标注的方法，同时也适用了@Transactional进行了标注；在其调用数据库操作时，将无法产生事务管理的控制，原因就在于其是基于异步处理的操作。
   那该如何给这些操作添加事务管理呢？可以将需要事务管理操作的方法放置到异步方法内部，在内部被调用的方法上添加@Transactional.
  例如：  方法A，同时使用了@Async/@Transactional来标注，但是无法产生事务控制的目的。
         方法B，使用了@Async来标注，  B中调用了方法C、D，方法C、D分别使用@Transactional做了标注，则可实现事务控制的目的。