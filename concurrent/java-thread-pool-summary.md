## Introduction to Thread Pools

As the name suggests, a thread pool is a resource pool that manages a series of threads and provides a way to limit and manage thread resources. Each thread pool also maintains some basic statistical information, such as the number of tasks completed.

Here, we borrow some content from the book "Java Concurrency in Practice" to summarize the benefits of using thread pools:

- **Reduce Resource Consumption**: By reusing already created threads, you reduce the overhead caused by thread creation and destruction.
- **Improve Response Speed**: When a task arrives, it can be executed immediately without waiting for a thread to be created.
- **Enhance Thread Manageability**: Threads are scarce resources. Unlimited creation of threads not only consumes system resources but also reduces system stability. Using a thread pool allows for unified allocation, tuning, and monitoring.

**Thread pools are generally used to execute multiple unrelated time-consuming tasks. Without multithreading, tasks are executed sequentially. Using a thread pool allows multiple unrelated tasks to be executed simultaneously.**

## Introduction to the Executor Framework

The `Executor` framework was introduced after Java 5. After Java 5, starting a thread using the `Executor` is better than using the `Thread`'s `start` method. Besides being easier to manage and more efficient (implemented with thread pools to save overhead), it also helps to avoid the "this escape" problem.

> "This escape" refers to a situation where another thread holds a reference to an object before its constructor has finished executing, potentially leading to confusing errors when calling methods on an incompletely constructed object.

The `Executor` framework not only includes the management of thread pools but also provides thread factories, queues, and rejection policies, making concurrent programming much simpler.

The `Executor` framework consists of three main parts:

**1. Tasks (`Runnable` / `Callable`)**

Tasks to be executed need to implement the **`Runnable` interface** or **`Callable` interface**. Classes implementing the **`Runnable` interface** or **`Callable` interface** can be executed by **`ThreadPoolExecutor`** or **`ScheduledThreadPoolExecutor`**.

**2. Task Execution (`Executor`)**

As shown in the diagram below, this includes the core interface for task execution **`Executor`**, and the **`ExecutorService` interface** which extends from the `Executor` interface. The two key classes **`ThreadPoolExecutor`** and **`ScheduledThreadPoolExecutor`** implement the **`ExecutorService`** interface.

![Executor Class Diagram](https://oss.javaguide.cn/github/javaguide/java/concurrent/executor-class-diagram.png)

Many underlying class relationships are mentioned here, but in practice, we need to focus more on the `ThreadPoolExecutor` class, which is frequently used when working with thread pools.

**Note:** By looking at the source code of `ScheduledThreadPoolExecutor`, we find that `ScheduledThreadPoolExecutor` actually extends `ThreadPoolExecutor` and implements `ScheduledExecutorService`, and `ScheduledExecutorService` in turn implements `ExecutorService`, as shown in the class relationship diagram above.

Description of the `ThreadPoolExecutor` class:

```java
// AbstractExecutorService implements the ExecutorService interface
public class ThreadPoolExecutor extends AbstractExecutorService
```

Description of the `ScheduledThreadPoolExecutor` class:

```java
// ScheduledExecutorService inherits the ExecutorService interface
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService
```

**3. Asynchronous Computation Results (`Future`)**

The **`Future`** interface and its implementation class **`FutureTask`** can represent the result of asynchronous computations.

When we submit an implementation class of the **`Runnable` interface** or **`Callable` interface** to **`ThreadPoolExecutor`** or **`ScheduledThreadPoolExecutor`** for execution (calling the `submit()` method), it will return a **`FutureTask`** object.

**Usage Diagram of the `Executor` Framework:**

![Usage Diagram of the Executor Framework](./images/java-thread-pool-summary/Executor框架的使用示意图.png)

1. The main thread first creates a task object implementing the `Runnable` or `Callable` interface.
2. The created `Runnable` or `Callable` object is directly handed over to the `ExecutorService` for execution: `ExecutorService.execute(Runnable command)` or submitted for execution: `ExecutorService.submit(Runnable task)` or `ExecutorService.submit(Callable<T> task)`.
3. If `ExecutorService.submit(...)` is used, `ExecutorService` will return an object implementing the `Future` interface (as mentioned earlier, the `submit()` method returns a `FutureTask` object). Since `FutureTask` implements `Runnable`, we can also create a `FutureTask` and hand it over to `ExecutorService` for execution.
4. Finally, the main thread can execute the `FutureTask.get()` method to wait for the task to complete. The main thread can also execute `FutureTask.cancel(boolean mayInterruptIfRunning)` to cancel the task's execution.

## Introduction to the `ThreadPoolExecutor` Class (Important)

The thread pool implementation class `ThreadPoolExecutor` is the core class of the `Executor` framework.

### Analysis of Thread Pool Parameters

The `ThreadPoolExecutor` class provides four constructors. Let's look at the longest one, as the other three are based on this constructor (the other constructors simply provide default parameters, such as default rejection policies).

```java
    /**
     * Creates a new ThreadPoolExecutor with the given initial parameters.
     */
    public ThreadPoolExecutor(int corePoolSize, // Number of core threads in the thread pool
                              int maximumPoolSize, // Maximum number of threads in the thread pool
                              long keepAliveTime, // Maximum time that excess idle threads will wait for new tasks before terminating
                              TimeUnit unit, // Time unit for the keepAliveTime parameter
                              BlockingQueue<Runnable> workQueue, // Task queue for storing waiting tasks
                              ThreadFactory threadFactory, // Factory for creating new threads, generally default
                              RejectedExecutionHandler handler // Rejection policy for handling tasks that cannot be executed
                               ) {
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

These parameters are very important and you will definitely use them when working with thread pools! So, make sure to take note of them.

The three most important parameters of `ThreadPoolExecutor`:

- `corePoolSize`: The maximum number of threads that can run simultaneously when the task queue has not reached its capacity.
- `maximumPoolSize`: The maximum number of threads that can run simultaneously when the task queue is full.
- `workQueue`: When a new task arrives, it first checks whether the number of running threads has reached the core thread count. If it has, the new task will be stored in the queue.

Other common parameters of `ThreadPoolExecutor`:

- `keepAliveTime`: When the number of threads in the thread pool exceeds `corePoolSize`, if no new tasks are submitted, the excess idle threads will not be immediately destroyed but will wait until the wait time exceeds `keepAliveTime` before being recycled.
- `unit`: The time unit for the `keepAliveTime` parameter.
- `threadFactory`: Used by the executor to create new threads.
- `handler`: Rejection policy (will be described in detail later).

The following diagram can help you understand the relationship between the various parameters in the thread pool (image source: "Java Performance Tuning In Action"):

![Relationship between thread pool parameters](https://oss.javaguide.cn/github/javaguide/java/concurrent/relationship-between-thread-pool-parameters.png)

**`ThreadPoolExecutor` Rejection Policies:**

When the current number of running threads reaches the maximum number and the queue is also full, `ThreadPoolExecutor` defines some strategies:

- `ThreadPoolExecutor.AbortPolicy`: Throws a `RejectedExecutionException` to reject new tasks.
- `ThreadPoolExecutor.CallerRunsPolicy`: Uses the thread that calls `execute` to run the task. This means the task will be executed in the calling thread if the thread pool is shut down, the task will be discarded. This strategy reduces the rate of task submission and affects overall performance. If your application can tolerate this delay and requires every task request to be executed, you can choose this strategy.
- `ThreadPoolExecutor.DiscardPolicy`: Discards the new task without processing it.
- `ThreadPoolExecutor.DiscardOldestPolicy`: Discards the earliest unhandled task request.

Example:

When creating a thread pool using `ThreadPoolTaskExecutor` in Spring or directly using the `ThreadPoolExecutor` constructor, if you do not specify a `RejectedExecutionHandler` rejection policy, the default is `AbortPolicy`. Under this rejection policy, if the queue is full, `ThreadPoolExecutor` will throw a `RejectedExecutionException` to reject new tasks, meaning you will lose the opportunity to handle this task. If you do not want to lose tasks, you can use `CallerRunsPolicy`. Unlike other strategies, `CallerRunsPolicy` does not discard tasks or throw exceptions but instead runs the task in the calling thread.

```java
public static class CallerRunsPolicy implements RejectedExecutionHandler {

    public CallerRunsPolicy() { }

    public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
        if (!e.isShutdown()) {
            // Execute directly in the main thread, not in the thread pool
            r.run();
        }
    }
}
```

### Two Ways to Create a Thread Pool

**Method 1: Create through the `ThreadPoolExecutor` constructor (recommended).**

![Create through constructor](./images/java-thread-pool-summary/threadpoolexecutor-constructor.png)

**Method 2: Create through the `Executors` utility class of the `Executor` framework.**

The `Executors` utility class provides methods for creating thread pools as shown in the following image:

![Executors utility class methods for creating thread pools](https://oss.javaguide.cn/github/javaguide/java/concurrent/executors-new-thread-pool-methods.png)

As can be seen, the `Executors` utility class can create various types of thread pools, including:

- `FixedThreadPool`: A thread pool with a fixed number of threads. The number of threads in this thread pool remains constant. When a new task is submitted, if there is an idle thread in the thread pool, it is immediately executed. If not, the new task will be temporarily stored in a task queue, waiting for an idle thread to process the tasks in the queue.
- `SingleThreadExecutor`: A thread pool with only one thread. If more than one task is submitted to this thread pool, the tasks will be stored in a task queue, and executed in a first-in-first-out order when the thread becomes idle.
- `CachedThreadPool`: A thread pool that adjusts the number of threads according to the actual situation. The number of threads in the thread pool is not fixed, but if there are idle threads that can be reused, they will be used first. If all threads are working and a new task is submitted, a new thread will be created to handle the task. After all threads complete the current task, they will return to the thread pool for reuse.
- `ScheduledThreadPool`: A thread pool for running tasks after a given delay or periodically.

The "Alibaba Java Development Manual" enforces the rule that thread pools should not be created using `Executors` but through the `ThreadPoolExecutor` constructor. This way, the writer has a clearer understanding of the thread pool's running rules and avoids the risk of resource exhaustion.

The drawbacks of the thread pools returned by `Executors` are as follows (detailed later):

- `FixedThreadPool` and `SingleThreadExecutor`: Use an unbounded `LinkedBlockingQueue`, where the task queue's maximum length is `Integer.MAX_VALUE`, which can accumulate a large number of requests, potentially leading to OOM (Out Of Memory).
- `CachedThreadPool`: Uses a synchronous queue `SynchronousQueue`, allowing the number of created threads to be `Integer.MAX_VALUE`. If the number of tasks is excessive and their execution speed is slow, a large number of threads may be created, leading to OOM.
- `ScheduledThreadPool` and `SingleThreadScheduledExecutor`: Use an unbounded delayed blocking queue `DelayedWorkQueue`, where the task queue's maximum length is `Integer.MAX_VALUE`, which can accumulate a large number of requests, potentially leading to OOM.

```java
// Unbounded queue LinkedBlockingQueue
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>());
}

// Unbounded queue LinkedBlockingQueue
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService (new ThreadPoolExecutor(1, 1,0L, TimeUnit.MILLISECONDS,new LinkedBlockingQueue<Runnable>()));
}

// Synchronous queue SynchronousQueue, no capacity, maximum thread count is Integer.MAX_VALUE
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,60L, TimeUnit.SECONDS,new SynchronousQueue<Runnable>());
}

// Delayed blocking queue DelayedWorkQueue
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

### Summary of Common Blocking Queues Used in Thread Pools

When a new task arrives, it first checks whether the number of running threads has reached the core thread count. If it has, the new task will be stored in the queue.

Different thread pools use different blocking queues, and we can analyze them based on the built-in thread pools.

- Unbounded queue `LinkedBlockingQueue` with a capacity of `Integer.MAX_VALUE`: Used by `FixedThreadPool` and `SingleThreadExecutor`. `FixedThreadPool` can create up to the core thread count (core thread count and maximum thread count are equal), and `SingleThreadExecutor` can create only one thread (core

thread count and maximum thread count are both 1). Their task queues will never be filled.
- Synchronous queue `SynchronousQueue`: Used by `CachedThreadPool`. `SynchronousQueue` has no capacity and does not store elements. Its purpose is to ensure that for submitted tasks, if there is an idle thread, it uses the idle thread to handle the task; otherwise, it creates a new thread to handle the task. In other words, the maximum thread count for `CachedThreadPool` is `Integer.MAX_VALUE`, which means the number of threads can be infinitely extended, potentially leading to OOM.
- Delayed blocking queue `DelayedWorkQueue`: Used by `ScheduledThreadPool` and `SingleThreadScheduledExecutor`. `DelayedWorkQueue` does not sort internal elements by the time they are added but sorts tasks by the length of the delay. It uses a heap data structure to ensure that each time a task is dequeued, it is the task with the shortest execution time. `DelayedWorkQueue` automatically expands to half its original capacity when full, never blocking and expanding up to `Integer.MAX_VALUE`, so it can only create up to the core thread count.

## Analysis of ThreadPool Principles (Important)

We've covered the `Executor` framework and the `ThreadPoolExecutor` class. Now, let's put that knowledge into practice by writing a small demo of `ThreadPoolExecutor` to review the concepts discussed above.

### ThreadPool Example Code

First, let's create a class that implements the `Runnable` interface (or you can use the `Callable` interface; we'll discuss the differences later).

`MyRunnable.java`

```java
import java.util.Date;

/**
 * This is a simple Runnable class that takes approximately 5 seconds to execute its task.
 * @author shuang.kou
 */
public class MyRunnable implements Runnable {

    private String command;

    public MyRunnable(String s) {
        this.command = s;
    }

    @Override
    public void run() {
        System.out.println(Thread.currentThread().getName() + " Start. Time = " + new Date());
        processCommand();
        System.out.println(Thread.currentThread().getName() + " End. Time = " + new Date());
    }

    private void processCommand() {
        try {
            Thread.sleep(5000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    @Override
    public String toString() {
        return this.command;
    }
}

```

Now, let's write a test program. Here, we'll follow Alibaba's recommendation to create a thread pool using the `ThreadPoolExecutor` constructor with custom parameters.

`ThreadPoolExecutorDemo.java`

```java
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

public class ThreadPoolExecutorDemo {

    private static final int CORE_POOL_SIZE = 5;
    private static final int MAX_POOL_SIZE = 10;
    private static final int QUEUE_CAPACITY = 100;
    private static final Long KEEP_ALIVE_TIME = 1L;
    public static void main(String[] args) {

        // Using the recommended way to create a thread pool by Alibaba
        // Creating a thread pool with custom parameters via ThreadPoolExecutor constructor
        ThreadPoolExecutor executor = new ThreadPoolExecutor(
                CORE_POOL_SIZE,
                MAX_POOL_SIZE,
                KEEP_ALIVE_TIME,
                TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(QUEUE_CAPACITY),
                new ThreadPoolExecutor.CallerRunsPolicy());

        for (int i = 0; i < 10; i++) {
            // Creating WorkerThread objects (WorkerThread class implements the Runnable interface)
            Runnable worker = new MyRunnable("" + i);
            // Executing the Runnable
            executor.execute(worker);
        }
        // Shutting down the thread pool
        executor.shutdown();
        while (!executor.isTerminated()) {
        }
        System.out.println("Finished all threads");
    }
}

```

In this code, we've specified:

- `corePoolSize`: 5 core threads.
- `maximumPoolSize`: Maximum of 10 threads.
- `keepAliveTime`: Waiting time of 1L.
- `unit`: Time unit for keepAliveTime as TimeUnit.SECONDS.
- `workQueue`: Task queue as `ArrayBlockingQueue` with a capacity of 100.
- `handler`: Rejection policy as `CallerRunsPolicy`.

**Output Structure**:

```plain
pool-1-thread-3 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-5 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-2 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-1 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-4 Start. Time = Sun Apr 12 11:14:37 CST 2020
pool-1-thread-3 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-4 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-1 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-5 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-1 Start. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-2 End. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-5 Start. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-4 Start. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-3 Start. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-2 Start. Time = Sun Apr 12 11:14:42 CST 2020
pool-1-thread-1 End. Time = Sun Apr 12 11:14:47 CST 2020
pool-1-thread-4 End. Time = Sun Apr 12 11:14:47 CST 2020
pool-1-thread-5 End. Time = Sun Apr 12 11:14:47 CST 2020
pool-1-thread-3 End. Time = Sun Apr 12 11:14:47 CST 2020
pool-1-thread-2 End. Time = Sun Apr 12 11:14:47 CST 2020
Finished all threads  // It exits only after all tasks are executed, as executor.isTerminated()
```

### Analysis of Thread Pool Principles

From the output results of the previous code, we can observe: **the thread pool first executes 5 tasks, and then if any of these tasks are completed, it will fetch new tasks to execute.** You can analyze what exactly is happening based on the explanation above (take some time to think independently).

Now, let's delve into analyzing the output content to understand the principles of the thread pool.

To understand the thread pool's principles, we need to first analyze the `execute` method. In the sample code, we use `executor.execute(worker)` to submit a task to the thread pool.

This method is crucial. Let's take a look at its source code:

```java
   // Stores the running state (runState) and the number of active threads in the thread pool (workerCount)
   private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));

    private static int workerCountOf(int c) {
        return c & CAPACITY;
    }
    // Task queue
    private final BlockingQueue<Runnable> workQueue;

    public void execute(Runnable command) {
        // Throw an exception if the task is null.
        if (command == null)
            throw new NullPointerException();
        // ctl stores some current status information of the thread pool.
        int c = ctl.get();

        // The following involves 3 steps
        // 1. First, check if the number of tasks executed in the current thread pool is less than corePoolSize.
        // If it is less, a new thread is created through addWorker(command, true), and the task (command) is added to that thread. Then, start the thread to execute the task.
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }
        // 2. If the current number of tasks executed is greater than or equal to corePoolSize, it will come here, indicating that creating a new thread has failed.
        // Check the thread pool state with the isRunning method. The task will only be added if the thread pool is in the RUNNING state and the queue can accept tasks.
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            // Check the thread pool state again. If the thread pool state is not RUNNING, the task needs to be removed from the task queue, and try to determine if all threads have finished execution. Meanwhile, execute the rejection policy.
            if (!isRunning(recheck) && remove(command))
                reject(command);
                // If the current number of working threads is 0, create a new thread and execute it.
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //3. Create a new thread through addWorker(command, false), and add the task (command) to the thread. Then, start the thread to execute the task.
        // Passing false indicates that when adding a thread, it checks whether the current number of threads is less than maxPoolSize.
        // If addWorker(command, false) fails, execute the corresponding rejection policy through reject().
        else if (!addWorker(command, false))
            reject(command);
    }
```

Here's a brief analysis of the entire process (the logic has been simplified for better understanding):

1. If the current number of running threads is less than the corePoolSize, a new thread will be created to execute the task.
2. If the current number of running threads is equal to or greater than the corePoolSize, but less than the maximumPoolSize, the task will be added to the task queue for later execution.
3. If adding the task to the task queue fails (because the queue is full), but the current number of running threads is less than the maximumPoolSize, a new thread will be created to execute the task.
4. If the current number of running threads equals the maximumPoolSize, creating a new thread would exceed the maximum number of threads allowed. In this case, the current task will be rejected, and the rejection policy will be executed, such as throwing an exception or discarding the task.

In the `execute` method, the `addWorker` method is called multiple times. The `addWorker` method is mainly used to create new worker threads. If it returns true, it means the worker thread creation and startup were successful; otherwise, it returns false.

```java
    // Global lock, essential for concurrent operations
private final ReentrantLock mainLock = new ReentrantLock();
// Tracks the maximum size of the thread pool, accessed only when holding the global lock mainLock
private int largestPoolSize;
// Worker thread collection, stores all (active) worker threads in the thread pool, accessed only when holding the global lock mainLock
private final HashSet<Worker> workers = new HashSet<>();
//Get the thread pool state
private static int runStateOf(int c)     { return c & ~CAPACITY; }
//Check if the thread pool state is Running
private static boolean isRunning(int c) {
    return c < SHUTDOWN;
    }


/**
 * Add a new worker thread to the thread pool
 * @param firstTask Task to be executed
 * @param core If true, it means using the basic size of the thread pool; if false, it uses the maximum size of the thread pool
 * @return Returns true if added successfully; otherwise, false
 */
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
    //These two lines are used to get the state of the thread pool
    int c = ctl.get();
    int rs = runStateOf(c);

    // Check if the queue is empty only if necessary.
    if (rs >= SHUTDOWN &&
    ! (rs == SHUTDOWN &&
    firstTask == null &&
    ! workQueue.isEmpty()))
    return false;

    for (;;) {
    //Get the number of working threads in the thread pool
    int wc = workerCountOf(c);
    // If core is false, it means the queue is also full, and the size of the thread pool becomes maximumPoolSize
    if (wc >= CAPACITY ||
    wc >= (core ? corePoolSize : maximumPoolSize))
    return false;
    //Atomically increment the number of workcounts
    if (compareAndIncrementWorkerCount(c))
    break retry;
    // If the thread state changes, perform the above operation again
    c = ctl.get();
    if (runStateOf(c) != rs)
    continue retry;
    // else CAS failed due to workerCount change; retry inner loop
    }
    }
    // Mark whether the worker thread is successfully started
    boolean workerStarted = false;
    // Mark whether the worker thread is successfully created
    boolean workerAdded = false;
    Worker w = null;
    try {

    w = new Worker(firstTask);
final Thread t = w.thread;
    if (t != null) {
// Lock
final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
    //Get the state of the thread pool
    int rs = runStateOf(ctl.get());
    //rs < SHUTDOWN If the thread pool
    Certainly, let's continue with the translation:

```java
                   // rs < SHUTDOWN If the thread pool state is still RUNNING, and the thread state is alive, the worker thread will be added to the worker thread collection
                  // (rs = SHUTDOWN && firstTask == null) If the thread pool state is less than STOP, which is either RUNNING or SHUTDOWN, and the passed task instance firstTask is null, it needs to be added to the worker thread collection and a new Worker needs to be started.
                   // The firstTask == null means only a new thread is created without executing any task
                    if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {
                        if (t.isAlive()) // precheck that t is startable
                            throw new IllegalThreadStateException();
                        workers.add(w);
                       // Update the current maximum capacity of working threads
                        int s = workers.size();
                        if (s > largestPoolSize)
                            largestPoolSize = s;
                      // Whether the worker thread is successfully started
                        workerAdded = true;
                    }
                } finally {
                    // Unlock
                    mainLock.unlock();
                }
                //// If a worker thread is successfully added, then call the Thread#start() method of the actual thread instance t within the Worker internally to start the real thread instance
                if (workerAdded) {
                    t.start();
                  /// Mark the thread as successfully started
                    workerStarted = true;
                }
            }
        } finally {
           // If the thread fails to start, the corresponding Worker needs to be removed from the worker threads
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }
```

For further understanding of thread pool source code analysis, I recommend this article: Hardcore Analysis: [Analyzing the Implementation Principle of JUC Thread Pool ThreadPoolExecutor from the Source Code (40,000 Words)](https://www.throwx.cn/2020/08/23/java-concurrency-thread-pool-executor/)

Now, let's go back to the example code. By now, you should find it easy to understand its principles, right?

If you haven't grasped it yet, don't worry. You can take a look at my analysis:

> In our code, we simulated 10 tasks. We configured the core thread pool size to be 5, and the queue capacity to be 100. Therefore, only 5 tasks can be executed simultaneously at any given time, and the remaining 5 tasks will be placed in the waiting queue. If any of the current 5 tasks are completed, the thread pool will fetch new tasks to execute.

### Several Common Comparisons

#### `Runnable` vs `Callable`

`Runnable` has been around since Java 1.0, while `Callable` was introduced in Java 1.5 to handle scenarios not supported by `Runnable`. The **`Runnable` interface** does not return a result or throw checked exceptions, whereas the **`Callable` interface** can do both. Therefore, if a task does not need to return a result or throw an exception, it's recommended to use the **`Runnable` interface** for cleaner code.

The `Executors` utility class can convert `Runnable` objects into `Callable` objects using the methods `Executors.callable(Runnable task)` or `Executors.callable(Runnable task, Object result)`.

`Runnable.java`

```java
@FunctionalInterface
public interface Runnable {
   /**
    * Executed by a thread, with no return value and no ability to throw exceptions.
    */
    public abstract void run();
}
```

`Callable.java`

```java
@FunctionalInterface
public interface Callable<V> {
    /**
     * Computes a result, or throws an exception if unable to do so.
     * @return the computed result
     * @throws an exception if unable to compute a result
     */
    V call() throws Exception;
}

```

#### `execute()` vs `submit()`

- The `execute()` method is used to submit tasks that do not need to return a value, so it cannot determine whether the task was successfully executed by the thread pool.
- The `submit()` method is used to submit tasks that need to return a value. The thread pool returns a `Future` object, through which you can determine whether the task was executed successfully. You can also use the `get()` method of `Future` to obtain the return value. The `get()` method blocks the current thread until the task is completed. If you use the `get(long timeout, TimeUnit unit)` method, and the task is not completed within the specified `timeout`, a `java.util.concurrent.TimeoutException` is thrown.

It's recommended to use the `ThreadPoolExecutor` constructor to create thread pools for practical usage.

Example 1: Obtaining the return value using the `get()` method.

```java
ExecutorService executorService = Executors.newFixedThreadPool(3);

Future<String> submit = executorService.submit(() -> {
    try {
        Thread.sleep(5000L);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "abc";
});

String s = submit.get();
System.out.println(s);
executorService.shutdown();
```

Output:

```plaintext
abc
```

Example 2: Obtaining the return value using the `get(long timeout, TimeUnit unit)` method.

```java
ExecutorService executorService = Executors.newFixedThreadPool(3);

Future<String> submit = executorService.submit(() -> {
    try {
        Thread.sleep(5000L);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "abc";
});

String s = submit.get(3, TimeUnit.SECONDS);
System.out.println(s);
executorService.shutdown();
```

Output:

```plaintext
Exception in thread "main" java.util.concurrent.TimeoutException
  at java.util.concurrent.FutureTask.get(FutureTask.java:205)
```

#### `shutdown()` vs `shutdownNow()`

- **`shutdown()`**: Shuts down the thread pool, changing its state to `SHUTDOWN`. The thread pool no longer accepts new tasks, but tasks in the queue are allowed to complete.
- **`shutdownNow()`**: Shuts down the thread pool, changing its state to `STOP`. The thread pool terminates the currently running tasks and stops processing queued tasks, returning a list of tasks waiting for execution.

#### `isTerminated()` vs `isShutdown()`

- **`isShutDown`**: Returns true after calling the `shutdown()` method.
- **`isTerminated`**: Returns true after calling the `shutdown()` method and after all submitted tasks have completed.

## Several Common Built-in Thread Pools

### FixedThreadPool

#### Introduction

`FixedThreadPool` is a thread pool with a fixed number of threads that can be reused. Let's take a look at the implementation of `FixedThreadPool` through the relevant source code in the `Executors` class:

```java
   /**
     * Creates a thread pool that reuses a fixed number of threads.
     */
    public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
    }
```

There is another implementation method for `FixedThreadPool` that is similar to the one above, so we won't elaborate on it here.

From the above source code, we can see that the newly created `FixedThreadPool` has both `corePoolSize` and `maximumPoolSize` set to `nThreads`, which is a parameter passed by ourselves when using it.

Even if the value of `maximumPoolSize` is greater than `corePoolSize`, only `corePoolSize` threads will be created at most. This is because `FixedThreadPool` uses a capacity `Integer.MAX_VALUE` `LinkedBlockingQueue` (unbounded queue), which will never be filled.

#### Task Execution Process

The execution process of the `execute()` method in `FixedThreadPool` (image source: "Java Concurrency in Practice"):

![Execution process of the execute() method in FixedThreadPool](https://i.imgur.com/uek1Y3X.png)

**Explanation:**

1. If the current number of running threads is less than `corePoolSize`, a new thread is created to execute the task if a new task arrives.
2. When the current number of running threads equals `corePoolSize`, if a new task arrives, it is added to the `LinkedBlockingQueue`.
3. After the threads in the thread pool finish executing the current tasks, they repeatedly retrieve tasks from the `LinkedBlockingQueue` in a loop.

#### Why Not Recommended to Use `FixedThreadPool`?

`FixedThreadPool` uses an unbounded queue `LinkedBlockingQueue` (queue capacity is `Integer.MAX_VALUE`) as the thread pool's work queue. This choice can have the following impacts:

1. Once the number of threads in the thread pool reaches `corePoolSize`, new tasks will wait in the unbounded queue indefinitely. Therefore, the number of threads in the thread pool will not exceed `corePoolSize`.
2. Since an unbounded queue is used, the `maximumPoolSize` parameter is effectively irrelevant because the queue can never be filled. Thus, as seen in the source code for creating `FixedThreadPool`, both `corePoolSize` and `maximumPoolSize` are set to the same value.
3. Due to points 1 and 2, the `keepAliveTime` parameter is also irrelevant when using an unbounded queue.
4. The running `FixedThreadPool` (not executed `shutdown()` or `shutdownNow()`) will not reject tasks, which can lead to an OutOfMemoryError (OOM) when there are too many tasks.

### SingleThreadExecutor

#### Introduction

`SingleThreadExecutor` is a thread pool with only one thread. Let's take a look at the implementation of `SingleThreadExecutor` through the source code:

```java
   /**
     * Creates a single-threaded executor that uses the provided
     * thread factory to create a new thread when needed.
     */
    public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

The newly created `SingleThreadExecutor` has both `corePoolSize` and `maximumPoolSize` set to 1, and other parameters are the same as `FixedThreadPool`.

#### Task Execution Process

The execution process of the `SingleThreadExecutor` (image source: "Java Concurrency in Practice"):

![Execution process of the SingleThreadExecutor](https://i.imgur.com/WLsKamw.png)

**Explanation:**

1. If the current number of running threads is less than `corePoolSize`, a new thread is created to execute the task.
2. After there is one running thread in the thread pool, tasks are added to the `LinkedBlockingQueue`.
3. After the thread finishes executing the current task, it repeatedly retrieves tasks from the `LinkedBlockingQueue` in a loop.

#### Why Not Recommended to Use `SingleThreadExecutor`?

Similar to `FixedThreadPool`, `SingleThreadExecutor` also uses an unbounded queue `LinkedBlockingQueue`, which may lead to an OOM when there are too many tasks.

### CachedThreadPool

#### Introduction

`CachedThreadPool` is a thread pool that creates new threads as needed. Let's take a look at the implementation of `CachedThreadPool` through the source code:

```java
    /**
     * Creates a thread pool that creates new threads as needed, but
     * will reuse previously constructed threads when they are
     * available.
     */
    public static ExecutorService newCachedThreadPool(ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>(),
                                      threadFactory);
    }

```

```java
    public static ExecutorService newCachedThreadPool() {
        return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                      60L, TimeUnit.SECONDS,
                                      new SynchronousQueue<Runnable>());
    }
```

In `CachedThreadPool`, both `corePoolSize` and `maximumPoolSize` are set to 0 and `Integer.MAX_VALUE`, respectively. This means it has an unbounded capacity, indicating that if the rate at which tasks are submitted exceeds the rate at which they can be processed, `CachedThreadPool` will continuously create new threads. In extreme cases, this could lead to CPU and memory resource exhaustion.

#### Task Execution Process

The execution process of the `execute()` method in `CachedThreadPool` (image source: "Java Concurrency in Practice"):

![Execution process of the execute() method in CachedThreadPool](https://i.imgur.com/taMzvVR.png)

**Explanation:**

1. Initially, the task is submitted to the `SynchronousQueue` using the `offer(Runnable task)` method. If there are idle threads in the `maximumPool`, and one of them is executing the `poll(keepAliveTime, TimeUnit.NANOSECONDS)` operation, the offer operation by the main thread and the poll operation by the idle thread pair successfully. The main thread hands over the task to the idle thread for execution, and the `execute()` method completes. Otherwise, proceed to step 2.
2. If the initial `maximumPool` is empty or there are no idle threads in the `maximumPool`, there will be no thread executing `SynchronousQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS)`. In this case, step 1 fails, and `CachedThreadPool` creates a new thread to execute the task, and the `execute()` method completes.

#### Why Not Recommended to Use `CachedThreadPool`?

`CachedThreadPool` uses a synchronous queue `SynchronousQueue`, which allows the creation of up to `Integer.MAX_VALUE` threads. This could potentially create a large number of threads, leading to CPU and memory resource exhaustion.

### ScheduledThreadPool

#### Introduction

`ScheduledThreadPool` is used to execute tasks after a given delay or to execute tasks periodically. This is rarely used in real projects and is not recommended. Let's take a look at the implementation:

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
          new DelayedWorkQueue());
}
```

`ScheduledThreadPool` is created through `ScheduledThreadPoolExecutor`, using a `DelayedWorkQueue` (a delayed blocking queue) as the thread pool's task queue.

The internal elements of `DelayedWorkQueue` are not sorted based on insertion time, but rather based on the delay time of tasks. It uses a "heap" data structure internally, ensuring that each dequeue task is the one with the earliest execution time in the current queue. `DelayedWorkQueue` automatically expands its capacity by 1/2 when it becomes full, meaning it never blocks. The maximum expansion can reach `Integer.MAX_VALUE`, so it can only create threads up to the number of core threads.

`ScheduledThreadPoolExecutor` inherits from `ThreadPoolExecutor`, so creating a `ScheduledThreadExecutor` essentially creates a `ThreadPoolExecutor` thread pool, with different parameters passed in.

```java
public class ScheduledThreadPoolExecutor
        extends ThreadPoolExecutor
        implements ScheduledExecutorService
```

#### Comparison Between ScheduledThreadPoolExecutor and Timer

- `Timer` is sensitive to changes in the system clock, while `ScheduledThreadPoolExecutor` is not.
- `Timer` has only one execution thread, so long-running tasks can delay other tasks. `ScheduledThreadPoolExecutor` can be configured with any number of threads. Additionally, you can have complete control over the threads created (via `ThreadFactory`).
- If a `TimerTask` throws a runtime exception, it will kill a thread, causing the `Timer` to stop running, i.e., scheduled tasks will no longer run. `ScheduledThreadPoolExecutor` not only captures runtime exceptions but also allows you to handle them when needed (by overriding the `afterExecute` method of `ThreadPoolExecutor`). Tasks that throw exceptions will be canceled, but other tasks will continue to run.

For a detailed explanation of scheduled tasks, you can refer to this article: [Java Scheduled Task Explained](https://javaguide.cn/system-design/schedule-task.html).                                      