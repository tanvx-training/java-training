## 1. Correctly Declare Thread Pool

**Thread pools must be declared manually using the constructor of `ThreadPoolExecutor` to avoid using the `Executors` class to create thread pools, which may pose an OOM risk.**

The disadvantages of using `Executors` to return thread pool objects are as follows (will be detailed later):

- **`FixedThreadPool` and `SingleThreadExecutor`**: Use an unbounded `LinkedBlockingQueue`, with a maximum task queue length of `Integer.MAX_VALUE`, which may accumulate a large number of requests, leading to OOM.
- **`CachedThreadPool`**: Uses a synchronous queue `SynchronousQueue`, allowing the creation of up to `Integer.MAX_VALUE` threads, which may create a large number of threads, leading to OOM.
- **`ScheduledThreadPool` and `SingleThreadScheduledExecutor`**: Use an unbounded delayed blocking queue `DelayedWorkQueue`, with a maximum task queue length of `Integer.MAX_VALUE`, which may accumulate a large number of requests, leading to OOM.

In short: **use bounded queues to control the number of thread creations.**

Apart from avoiding OOM, the reasons for not recommending the two shortcut thread pools provided by `Executors` are:

- In practical usage, thread pool parameters such as core thread count, task queue usage, saturation policy, etc., need to be manually configured according to the performance of the machine and business scenarios.
- We should explicitly name our thread pools, which helps us to locate issues.

## 2. Monitoring Thread Pool Running Status

You can use various methods to monitor the running status of a thread pool, such as the Actuator component in Spring Boot.

In addition to this, we can also use related APIs of `ThreadPoolExecutor` to create a rudimentary monitor. From the following image, you can see that `ThreadPoolExecutor` provides information such as current thread count, active thread count, completed task count, and queued task count.

![ThreadPoolExecutor Methods Information](https://oss.javaguide.cn/github/javaguide/java/concurrent/threadpool-methods-information.png)

Here's a simple demo. `printThreadPoolStatus()` will print the thread pool's thread count, active thread count, completed task count, and queued task count every second.

```java
/**
 * Prints the status of the thread pool
 *
 * @param threadPool The thread pool object
 */
public static void printThreadPoolStatus(ThreadPoolExecutor threadPool) {
    ScheduledExecutorService scheduledExecutorService = new ScheduledThreadPoolExecutor(1, createThreadFactory("print-images/thread-pool-status", false));
    scheduledExecutorService.scheduleAtFixedRate(() -> {
        log.info("=========================");
        log.info("ThreadPool Size: [{}]", threadPool.getPoolSize());
        log.info("Active Threads: {}", threadPool.getActiveCount());
        log.info("Number of Tasks : {}", threadPool.getCompletedTaskCount());
        log.info("Number of Tasks in Queue: {}", threadPool.getQueue().size());
        log.info("=========================");
    }, 0, 1, TimeUnit.SECONDS);
}
```

## 3. Suggest Using Different Thread Pools for Different Categories of Business

Many people encounter a similar problem in real projects: **multiple businesses in my project need to use thread pools. Should I define one for each thread pool or define a common one?**

Generally, it is recommended to use different thread pools for different businesses and configure the current thread pool according to the current business situation. This is because the concurrency and resource usage of different businesses are different, focusing on optimizing performance bottlenecks related to system performance.

**Let's look at a real accident case!** (This case is from: [《Improper Use of Thread Pool Causing an Online Incident》](https://heapdump.cn/article/646639), a wonderful case)

![Overview of the Case Code](https://oss.javaguide.cn/github/javaguide/java/concurrent/production-accident-threadpool-sharing-example.png)

The above code may lead to a deadlock situation. Why? Let's illustrate it with a diagram.

Imagine an extreme situation: suppose the core thread count of our thread pool is **n**, and there are **n** parent tasks (deduction tasks), each parent task has two sub-tasks (sub-tasks under the deduction task), one of which has been executed and the other is placed in the task queue. Since the parent task has used up the thread pool's core thread resources, the sub-task cannot obtain thread resources and cannot be executed normally, remaining blocked in the queue. The parent task waits for the sub-task to complete execution, while the sub-task waits for the parent task to release the thread pool resources, leading to a **"deadlock"**.

![Improper Use of Thread Pool Leading to Deadlock](https://oss.javaguide.cn/github/javaguide/java/concurrent/production-accident-threadpool-sharing-deadlock.png)

The solution is simple: add a separate thread pool for executing sub-tasks to avoid deadlock.

## 4. Don't Forget to Name Your Thread Pools

When initializing a thread pool, it's important to explicitly name it (by setting a thread pool name prefix), which aids in issue localization.

By default, thread names created resemble `pool-1-thread-n`, lacking any business significance, which makes problem localization difficult for us.

There are typically two ways to name threads in a thread pool:

**1. Utilizing Guava's `ThreadFactoryBuilder`**

```java
ThreadFactory threadFactory = new ThreadFactoryBuilder()
                        .setNameFormat(threadNamePrefix + "-%d")
                        .setDaemon(true).build();
ExecutorService threadPool = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, TimeUnit.MINUTES, workQueue, threadFactory)
```

**2. Implementing Your Own `ThreadFactory`.**

```java
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.atomic.AtomicInteger;

/**
 * Thread factory that sets thread names, aiding in issue localization.
 */
public final class NamingThreadFactory implements ThreadFactory {

  private final AtomicInteger threadNum = new AtomicInteger();
  private final String name;

  /**
   * Create a named thread pool production factory.
   */
  public NamingThreadFactory(String name) {
    this.name = name;
  }

  @Override
  public Thread newThread(Runnable r) {
    Thread t = new Thread(r);
    t.setName(name + " [#" + threadNum.incrementAndGet() + "]");
    return t;
  }
}
```

## 5. Correctly Configure Thread Pool Parameters

When it comes to configuring thread pool parameters, the ingenious approach by Meituan remains unforgettable even to this day (as mentioned later)!

Let's first examine the conventional methods recommended in various books and blogs for configuring thread pool parameters, which can serve as references.

### Conventional Approach

Many people might even think that setting the thread pool size slightly larger is better! I believe this is clearly problematic. Let's take a very common example from our lives: **having more people doesn't necessarily mean getting things done better; it increases communication costs. If a task only requires 3 people, but you bring in 6, would it improve efficiency? I doubt it.** The impact of having too many threads is similar to allocating too many people to a task; it primarily increases the **context switching** cost in the multi-threading scenario. If you're not familiar with what context switching is, you can see my explanation below.

> Context Switching:
>
> In multi-threaded programming, the number of threads is generally greater than the number of CPU cores. However, a CPU core can only be used by one thread at any given time. To ensure that all threads receive effective execution, the CPU employs a strategy of allocating time slices to each thread and rotating them. When a thread's time slice is exhausted, it returns to a ready state to allow other threads to use the CPU. This process is called a context switch. In summary, it involves saving the state of the current task before switching to another task, so that the task's state can be loaded again when switching back. **The process of saving and loading the task's state is called a context switch**.
>
> Context switching is typically resource-intensive. In other words, it requires a significant amount of processor time. In dozens or hundreds of switches per second, each switch may require a nanosecond-level time. Therefore, context switching consumes a considerable amount of CPU time and may be the most time-consuming operation in an operating system.
>
> Linux has many advantages compared to other operating systems (including other Unix-like systems), one of which is that its context switching and mode switching consume very little time.

Analogous to human cooperation in the real world to accomplish a task, it's certain that setting the thread pool size too large or too small will both cause issues. The most suitable size is the best.

- If we set the thread pool size too small, and there are a large number of tasks/requests to handle at the same time, it may lead to a situation where a large number of tasks/requests are queued in the task queue waiting to be executed. This could even lead to situations where tasks/requests cannot be processed after the task queue is full, or a large number of tasks accumulate in the task queue, leading to OOM. Clearly, this is problematic as the CPU is not being fully utilized.
- If we set the number of threads too large, many threads may contend for CPU resources simultaneously, resulting in a large number of context switches and thus increasing the execution time of threads, affecting overall execution efficiency.

There is a simple and widely applicable formula:

- **CPU Intensive Tasks (N+1):** For tasks that primarily consume CPU resources, the number of threads can be set to N (the number of CPU cores) + 1. The additional thread beyond the number of CPU cores is to prevent occasional interruptions due to page faults or other reasons. Once a task is paused, the CPU remains idle, and the extra thread can make full use of the CPU's idle time.
- **I/O Intensive Tasks (2N):** For such tasks, the system spends most of its time processing I/O interactions, and threads do not occupy the CPU for processing during I/O operations. Therefore, in I/O intensive task applications, more threads can be configured. The specific calculation method is 2N.

**How to determine whether a task is CPU intensive or I/O intensive?**

CPU intensive tasks are simply tasks that utilize CPU computing power, such as sorting a large amount of data in memory. Tasks involving network reads or file reads are I/O intensive. These tasks spend minimal time on CPU computation compared to waiting for I/O operations to complete.

🌈 Let's expand a bit (see: [issue#1737](https://github.com/Snailclimb/JavaGuide/issues/1737)):

A more rigorous method of calculating the number of threads is: `Optimal Thread Count = N (CPU core count) * (1 + WT (Thread Wait Time) / ST (Thread Computing Time))`, where `WT (Thread Wait Time) = Total Thread Runtime - ST (Thread Computing Time)`.

The higher the proportion of thread wait time, the more threads are needed. The higher the proportion of thread computing time, the fewer threads are needed.

We can use the JDK's built-in tool VisualVM to view the `WT/ST` ratio.

For CPU-intensive tasks, the `WT/ST` ratio is close to or equal to 0. Therefore, the number of threads can be set to N (the number of CPU cores) * (1 + 0) = N, which is similar to what we mentioned earlier, N (the number of CPU cores) + 1.

For I/O intensive tasks, almost all time is spent waiting for threads, so theoretically, you can set the number of threads to 2N (in theory, the result of WT/ST should be relatively large. The reason for choosing 2N here is probably to avoid creating too many threads).

**Note**: The formula mentioned above is just a reference. In actual projects, it's unlikely that thread pool parameters will be set directly according to the formula, as different business scenarios have different requirements. Specific adjustments should be based on the actual runtime situation of the project. The following introduction of Meituan's dynamic thread pool parameter configuration solution is very good and practical!

### Meituan's Ingenious Approach

The Meituan technology team introduced in the article ["Java Thread Pool Implementation Principles and Its Practice in Meituan Business"](https://tech.meituan.com/2020/04/02/java-pooling-pratice-in-meituan.html) the idea and method of implementing customizable configuration of thread pool parameters.

The Meituan technology team's approach primarily involves customizing the configuration of key thread pool parameters. These three key parameters are:

- **`corePoolSize`:** This defines the minimum number of threads that can run simultaneously.
- **`maximumPoolSize`:** When the number of tasks in the queue reaches the queue capacity, the number of threads that can run simultaneously becomes the maximum number of threads.
- **`workQueue`:** When a new task arrives, it first checks whether the current number of running threads exceeds the core thread count. If it does, the new task is placed in the queue.

**Why these three parameters?**

As I mentioned in this summary of ["Thread Pool Learning for Beginners"](https://mp.weixin.qq.com/s?__biz=Mzg2OTA0Njk0OA==&mid=2247485808&idx=1&sn=1013253533d73450cef673aee13267ab&chksm=cea246bbf9d5cfad1c21316340a0ef1609a7457fea4113a1f8d69e8c91e7d9cd6285f5ee1490&token=510053261&lang=zh_CN&scene=21#wechat_redirect), these three parameters are the most important parameters of `ThreadPoolExecutor`, and they basically determine the thread pool's task processing strategy.

**How to support dynamic configuration of parameters?** Take a look at the methods provided by `ThreadPoolExecutor` below.

![ThreadPoolExecutor Methods](https://oss.javaguide.cn/github/javaguide/java/concurrent/threadpoolexecutor-methods.png)

Of particular note is `corePoolSize`. During program execution, if we call the `setCorePoolSize()` method, the thread pool first checks whether the current number of working threads is greater than `corePoolSize`. If it is, it will reclaim working threads.

Additionally, you might notice that there is no method to dynamically specify the queue length above. Meituan's approach is to customize a queue called `ResizableCapacityLinkedBlockIngQueue` (mainly by removing the `final` keyword from the `capacity` field of `LinkedBlockingQueue`, making it mutable).

The final effect of dynamically modifying thread pool parameters is as follows. 👏👏👏

![Dynamically Configuring Thread Pool Parameters](https://oss.javaguide.cn/github/javaguide/java/concurrent/meituan-dynamically-configuring-thread-pool-parameters.png)

If our project also wants to achieve this effect, we can leverage existing open-source projects:

- **[Hippo4j](https://github.com/opengoofy/hippo4j)**: An asynchronous thread pool framework that supports dynamic changes to thread pools, monitoring, and alerts without modifying code. It supports multiple usage patterns and is easy to integrate, dedicated to improving system operation guarantee capabilities.
- **[Dynamic TP](https://github.com/dromara/dynamic-tp)**: Lightweight dynamic thread pool with built-in monitoring and alerting functions, integrating third-party middleware thread pool management, based on mainstream configuration centers (currently supports Nacos, Apollo, Zookeeper, Consul, Etcd, and can be customized via SPI).

## 6. Don't Forget to Shut Down the Thread Pool

When a thread pool is no longer needed, it should be explicitly shut down to release thread resources.

The thread pool provides two shutdown methods:

- **`shutdown()`**: Shuts down the thread pool, changing the thread pool's state to `SHUTDOWN`. The thread pool will no longer accept new tasks, but tasks in the queue will be executed.
- **`shutdownNow()`**: Shuts down the thread pool, changing the thread pool's state to `STOP`. The thread pool will terminate currently running tasks, stop processing queued tasks, and return a list of tasks waiting to be executed.

After calling the `shutdownNow()` and `shutdown()` methods, it does not mean that the thread pool has completed the shutdown operation. It just asynchronously notifies the thread pool to shut down. If you want to wait synchronously until the thread pool is completely shut down before continuing execution, you need to call the `awaitTermination()` method for synchronous waiting.

When calling the `awaitTermination()` method, a reasonable timeout should be set to avoid long-term blocking of the program, leading to performance issues. Additionally, since tasks in the thread pool may be canceled or throw exceptions, exception handling is required when using the `awaitTermination()` method. The `awaitTermination()` method throws an `InterruptedException` exception, which needs to be caught and handled to avoid program crashes or abnormal exits.

```java
// ...
// Shut down the thread pool
executor.shutdown();
try {
    // Wait for the thread pool to shut down, maximum wait time is 5 minutes
    if (!executor.awaitTermination(5, TimeUnit.MINUTES)) {
        // If timeout occurs, print log
        System.err.println("The thread pool did not fully shut down within 5 minutes");
    }
} catch (InterruptedException e) {
    // Exception handling
}
```

## 7. Avoid Putting Time-consuming Tasks in the Thread Pool

The purpose of a thread pool itself is to improve task execution efficiency and avoid the performance overhead of frequent thread creation and destruction. If time-consuming tasks are submitted to the thread pool for execution, it may occupy the threads in the thread pool for a long time, making it unable to respond to other tasks in a timely manner, and even causing the thread pool to crash or the program to hang.

Therefore, when using a thread pool, we should try to avoid submitting time-consuming tasks to it. For some relatively time-consuming operations, such as network requests, file I/O, etc., asynchronous operations should be used to handle them to avoid blocking threads in the thread pool.

## 8. Some Pitfalls in Using Thread Pools

### Pitfall of Creating Thread Pools Repeatedly

Thread pools are reusable, so never create thread pools frequently, such as creating a new thread pool for each user request.

```java
@GetMapping("wrong")
public String wrong() throws InterruptedException {
    // Custom thread pool
    ThreadPoolExecutor executor = new ThreadPoolExecutor(5,10,1L,TimeUnit.SECONDS,new ArrayBlockingQueue<>(100),new ThreadPoolExecutor.CallerRunsPolicy());

    //  Process tasks
    executor.execute(() -> {
      // ......
    }
    return "OK";
}
```

The reason for this problem is still a lack of understanding of thread pools, and there's a need to strengthen basic knowledge of thread pools.

### Pitfall of Spring Internal Thread Pool

When using Spring's internal thread pool, always manually customize the thread pool and configure reasonable parameters, otherwise, production problems may occur (creating a thread for each request).

```java
@Configuration
@EnableAsync
public class ThreadPoolExecutorConfig {

    @Bean(name="threadPoolExecutor")
    public Executor threadPoolExecutor(){
        ThreadPoolTaskExecutor threadPoolExecutor = new ThreadPoolTaskExecutor();
        int processNum = Runtime.getRuntime().availableProcessors(); // Returns the number of processors available to the Java virtual machine
        int corePoolSize = (int) (processNum / (1 - 0.2));
        int maxPoolSize = (int) (processNum / (1 - 0.5));
        threadPoolExecutor.setCorePoolSize(corePoolSize); // Core pool size
        threadPoolExecutor.setMaxPoolSize(maxPoolSize); // Maximum number of threads
        threadPoolExecutor.setQueueCapacity(maxPoolSize * 1000); // Queue length
        threadPoolExecutor.setThreadPriority(Thread.MAX_PRIORITY);
        threadPoolExecutor.setDaemon(false);
        threadPoolExecutor.setKeepAliveSeconds(300);// Thread idle time
        threadPoolExecutor.setThreadNamePrefix("test-Executor-"); // Thread name prefix
        return threadPoolExecutor;
    }
}
```

### Pitfall of Sharing Thread Pool with ThreadLocal

Sharing a thread pool with `ThreadLocal` may lead to threads obtaining old/dirty data from `ThreadLocal`. This is because the thread pool will reuse thread objects, and `ThreadLocal` variables bound to thread objects will also be reused, causing a thread to possibly obtain the `ThreadLocal` value of another thread.

Don't assume that if the code doesn't explicitly use a thread pool, there is no thread pool. For example, commonly used web servers like Tomcat use thread pools to improve concurrency, and they use custom thread pools based on native Java thread pools.

Of course, you can set Tomcat to handle tasks with a single thread. However, this is not appropriate and will severely affect the speed of task processing.

```properties
server.tomcat.max-threads=1
```

A recommended solution to the above problem is to use Alibaba's open-source `TransmittableThreadLocal` (`TTL`). The `TransmittableThreadLocal` class extends and enhances the JDK's built-in `InheritableThreadLocal` class. In scenarios such as using thread pools that pool and reuse threads, it provides the functionality of transmitting `ThreadLocal` values to solve the problem of context propagation during asynchronous execution.

`TransmittableThreadLocal` project address: [https://github.com/alibaba/transmittable-thread-local](https://github.com/alibaba/transmittable-thread-local).