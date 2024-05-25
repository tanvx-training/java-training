## Introduction to AQS

AQS stands for `AbstractQueuedSynchronizer`, which translates to abstract queue synchronizer. This class is part of the `java.util.concurrent.locks` package.

![](https://oss.javaguide.cn/github/javaguide/AQS.png)

AQS is an abstract class primarily used to construct locks and synchronizers.

```java
public abstract class AbstractQueuedSynchronizer extends AbstractOwnableSynchronizer implements java.io.Serializable {
}
```

AQS provides implementations of some common functionalities for constructing locks and synchronizers. Thus, using AQS allows for the simple and efficient construction of widely used synchronizers, such as `ReentrantLock`, `Semaphore`, and others like `ReentrantReadWriteLock` and `SynchronousQueue`, which are all based on AQS.

## Principle of AQS

When asked about concurrency knowledge in interviews, you will often be asked, "Can you explain your understanding of the principle of AQS?" Below is an example for reference. Remember, interviews are not about rote memorization; it's crucial to add your own thoughts. Even if you can't, ensure you can explain it clearly rather than just reciting it.

### Core Idea of AQS

The core idea of AQS is that if the requested shared resource is free, the current thread requesting the resource is set as the valid working thread, and the shared resource is set to a locked state. If the requested shared resource is occupied, a mechanism for thread blocking and lock allocation upon wake-up is needed. AQS implements this mechanism based on the **CLH lock** (Craig, Landin, and Hagersten locks).

The CLH lock is an improvement over the spinlock and is a virtual bidirectional queue (a virtual bidirectional queue means there is no instance of the queue, only the relationship between nodes). Threads that cannot obtain the lock temporarily are added to this queue. AQS encapsulates each thread requesting a shared resource into a node of the CLH queue lock (Node) to implement lock allocation. In the CLH queue lock, a node represents a thread, holding references to the thread, the current node's state in the queue (waitStatus), the predecessor node (prev), and the successor node (next).

The structure of the CLH queue is shown below:

![CLH Queue Structure](https://oss.javaguide.cn/github/javaguide/java/concurrent/clh-queue-structure.png)

For a detailed interpretation of the core data structure of AQS, the CLH lock, I highly recommend reading [Java AQS Core Data Structure - CLH Lock - Qunar Technical Salon](https://mp.weixin.qq.com/s/jEx-4XhNGOFdCo4Nou5tqg).

The core principle diagram of AQS (`AbstractQueuedSynchronizer`):

![CLH Queue](https://oss.javaguide.cn/github/javaguide/java/concurrent/clh-queue-state.png)

AQS uses an **int member variable `state` to represent the synchronization state**, and uses an **FIFO thread wait/waiting queue** to complete the queuing of threads requesting resources.

The `state` variable is modified by `volatile` to ensure visibility across threads.

```java
// Shared variable, modified by volatile to ensure visibility across threads
private volatile int state;
```

Additionally, the state information `state` can be operated on by the `protected` methods `getState()`, `setState()`, and `compareAndSetState()`. These methods are all `final`, so they cannot be overridden in subclasses.

```java
// Returns the current value of the synchronization state
protected final int getState() {
     return state;
}
// Sets the value of the synchronization state
protected final void setState(int newState) {
     state = newState;
}
// Atomically sets the synchronization state to the given value if the current state value equals the expected value
protected final boolean compareAndSetState(int expect, int update) {
      return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

Take the reentrant mutex lock `ReentrantLock` as an example, it maintains a `state` variable internally to represent the lock's occupancy state. The initial value of `state` is 0, indicating the lock is unlocked. When thread A calls the `lock()` method, it will try to acquire the lock exclusively through the `tryAcquire()` method and increase the value of `state` by 1. If successful, thread A acquires the lock. If it fails, thread A will be added to a waiting queue (CLH queue) until other threads release the lock. Suppose thread A successfully acquires the lock; it can reacquire the lock before releasing it (`state` will accumulate). This demonstrates reentrancy: a thread can acquire the same lock multiple times without being blocked. However, this also means that a thread must release the lock the same number of times it acquired it to return the `state` value to 0, unlocking the lock. Only then can other waiting threads have a chance to acquire the lock.

The process of thread A trying to acquire the lock is shown below (source: [ReentrantLock Implementation and AQS Principle and Application - Meituan Technical Team](./reentrantlock.md)):

![AQS Exclusive Mode Acquire Lock](https://oss.javaguide.cn/github/javaguide/java/concurrent/aqs-exclusive-mode-acquire-lock.png)

Similarly, in the countdown latch `CountDownLatch`, tasks are divided into N sub-threads to execute, and `state` is initialized to N (note that N must match the number of threads). These N sub-threads begin executing tasks, and each sub-thread calls the `countDown()` method once it completes. This method will attempt to decrease the value of `state` by 1 using the CAS (Compare and Swap) operation. When all sub-threads are finished (i.e., the value of `state` becomes 0), `CountDownLatch` will call the `unpark()` method to wake up the main thread. At this point, the main thread can return from the `await()` method (the `await()` method in `CountDownLatch`, not in AQS) and continue executing subsequent operations.

### Resource Sharing Mode of AQS

AQS defines two resource sharing modes: `Exclusive` (only one thread can execute, such as `ReentrantLock`) and `Share` (multiple threads can execute simultaneously, such as `Semaphore`/`CountDownLatch`).

Generally, the sharing mode of a custom synchronizer is either exclusive or shared, and they only need to implement one of `tryAcquire-tryRelease` or `tryAcquireShared-tryReleaseShared`. However, AQS also supports custom synchronizers implementing both exclusive and shared modes, such as `ReentrantReadWriteLock`.

### Custom Synchronizer

The design of the synchronizer is based on the template method pattern. To customize a synchronizer, the typical approach is:

1. The user inherits `AbstractQueuedSynchronizer` and overrides the specified methods.
2. The user incorporates AQS in the implementation of custom synchronization components and calls its template methods, which in turn call the overridden methods by the user.

This is quite different from the usual way of implementing interfaces. This is a classic use of the template method pattern.

**AQS uses the template method pattern. To customize a synchronizer, you need to override the following hook methods provided by AQS:**

```java
// Exclusive mode. Attempts to acquire the resource, returning true if successful, false otherwise.
protected boolean tryAcquire(int)
// Exclusive mode. Attempts to release the resource, returning true if successful, false otherwise.
protected boolean tryRelease(int)
// Shared mode. Attempts to acquire the resource. A negative value indicates failure; zero indicates success but no available resources left; a positive value indicates success and remaining resources.
protected int tryAcquireShared(int)
// Shared mode. Attempts to release the resource, returning true if successful, false otherwise.
protected boolean tryReleaseShared(int)
// Whether the thread is exclusively holding the resource. Only needs to be implemented if conditions are used.
protected boolean isHeldExclusively()
```

**What is a hook method?** A hook method is a method declared in an abstract class, typically protected, that can be an empty method (implemented by subclasses) or a default implementation. The template design pattern controls the implementation of fixed steps through hook methods.

Due to space constraints, the template method pattern will not be detailed here. Those unfamiliar with it can refer to this article: [Java 8 Transformation of the Template Method Pattern is Truly Awesome!](https://mp.weixin.qq.com/s/zpScSCktFpnSWHWIQem2jg).

Besides the hook methods mentioned above, other methods in the AQS class are `final` and cannot be overridden by other classes.

## Common Synchronization Utilities

Here, we introduce several common synchronization utilities based on AQS.

### Semaphore

#### Introduction

Both `synchronized` and `ReentrantLock` allow only one thread to access a resource at a time. In contrast, a `Semaphore` can control the number of threads that simultaneously access a particular resource.

Using a `Semaphore` is straightforward. Suppose we have `N (N > 5)` threads trying to acquire shared resources from a `Semaphore`. The following code ensures that at most 5 threads can acquire the shared resources at the same time. Other threads will be blocked until some threads release the resources, allowing blocked threads to acquire them.

```java
// Initial number of shared resources
final Semaphore semaphore = new Semaphore(5);
// Acquire 1 permit
semaphore.acquire();
// Release 1 permit
semaphore.release();
```

When the initial number of resources is 1, the `Semaphore` degenerates into an exclusive lock.

A `Semaphore` has two modes:

- **Fair mode:** The order of calling `acquire()` is the order of obtaining permits, following FIFO.
- **Unfair mode:** Permits are obtained preemptively.

The corresponding constructors for `Semaphore` are as follows:

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

**Both constructors must specify the number of permits. The second constructor can specify fair or unfair mode, with the default being unfair mode.**

`Semaphore` is commonly used in scenarios where resource access has a clear quantity limit, such as rate limiting (for single-machine mode, using Redis + Lua is recommended for rate limiting in actual projects).

#### Principle

A `Semaphore` is an implementation of a shared lock. It initializes the AQS `state` value as `permits`, where `permits` represents the number of available permits. Only threads that obtain permits can execute.

Take the parameterless `acquire` method as an example. When calling `semaphore.acquire()`, the thread tries to acquire a permit. If `state > 0`, it indicates a successful acquisition. If `state <= 0`, it indicates insufficient permits, resulting in a failed acquisition.

If successful (`state > 0`), the thread uses a CAS operation to modify the `state` value (`state = state - 1`). If unsuccessful, a Node is created and added to the wait queue, suspending the current thread.

```java
// Acquire 1 permit
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

// Acquire one or more permits
public void acquire(int permits) throws InterruptedException {
    if (permits < 0) throw new IllegalArgumentException();
    sync.acquireSharedInterruptibly(permits);
}
```

The `acquireSharedInterruptibly` method is the default implementation in `AbstractQueuedSynchronizer`.

```java
// In shared mode, acquire permits. If successful, return; if failed, add to the wait queue and suspend the thread
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
      throw new InterruptedException();
        // Try to acquire permits; if failed, create a node, add it to the wait queue, and suspend the thread
    if (tryAcquireShared(arg) < 0)
      doAcquireSharedInterruptibly(arg);
}
```

For example, in the unfair mode (`NonfairSync`), the implementation of the `tryAcquireShared` method is as follows:

```java
// In shared mode, try to acquire resources (permits in Semaphore):
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

// Unfair shared mode acquisition of permits
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        // Current available permits
        int available = getState();
        /*
         * Try to acquire permits. If the current number of available permits is <= 0, return a negative value indicating failure.
         * If the current number of available permits is > 0, acquire permits successfully. If CAS fails, loop to get the latest value and retry.
         */
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

Take the parameterless `release` method as an example. When calling `semaphore.release();`, the thread tries to release a permit and uses a CAS operation to modify the `state` value (`state = state + 1`). After successfully releasing the permit, it will also wake up one thread from the wait queue. The awakened thread will retry to modify the `state` value (`state = state - 1`). If `state > 0`, the thread successfully acquires the permit; otherwise, it re-enters the wait queue and suspends.

```java
// Release 1 permit
public void release() {
    sync.releaseShared(1);
}

// Release one or more permits
public void release(int permits) {
    if (permits < 0) throw new IllegalArgumentException();
    sync.releaseShared(permits);
}
```

The `releaseShared` method is the default implementation in `AbstractQueuedSynchronizer`.

```java
// Release the shared lock. If tryReleaseShared returns true, wake up one or more threads in the wait queue.
public final boolean releaseShared(int arg) {
    // Release the shared lock
    if (tryReleaseShared(arg)) {
      // Release the next waiting node
      doReleaseShared();
      return true;
    }
    return false;
}
```

The `tryReleaseShared` method is overridden in the internal `Sync` class of `Semaphore`, as the default implementation in `AbstractQueuedSynchronizer` only throws an `UnsupportedOperationException`.

```java
// Overridden in the internal Sync class. Attempt to release resources
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        // Increase available permits by 1
        int next = current + releases;
        if (next < current) // overflow
            throw new Error("Maximum permit count exceeded");
         // CAS modify the state value
        if (compareAndSetState(current, next))
            return true;
    }
}
```

As seen, the underlying implementations of the methods mentioned above are mostly through the synchronizer `sync`. `Sync` is an internal class of `Semaphore`, inheriting `AbstractQueuedSynchronizer` and overriding some of its methods. `Sync` also has two subclasses, `NonfairSync` (for unfair mode) and `FairSync` (for fair mode).

```java
private static final class Sync extends AbstractQueuedSynchronizer {
  // ...
}
static final class NonfairSync extends Sync {
  // ...
}
static final class FairSync extends Sync {
  // ...
}
```

### Practical Example

```java
public class SemaphoreExample {
  // Number of requests
  private static final int threadCount = 550;

  public static void main(String[] args) throws InterruptedException {
    // Create a thread pool with a fixed number of threads (if the thread count here is too low, you will find execution very slow)
    ExecutorService threadPool = Executors.newFixedThreadPool(300);
    // Initial number of permits
    final Semaphore semaphore = new Semaphore(20);

    for (int i = 0; i < threadCount; i++) {
      final int threadnum = i;
      threadPool.execute(() -> { // Use of Lambda expression
        try {
          semaphore.acquire(); // Acquire a permit, so the number of runnable threads is 20/1=20
          test(threadnum);
          semaphore.release(); // Release a permit
        } catch (InterruptedException e) {
          e.printStackTrace();
        }
      });
    }
    threadPool.shutdown();
    System.out.println("finish");
  }

  public static void test(int threadnum) throws InterruptedException {
    Thread.sleep(1000); // Simulate the time-consuming operation of a request
    System.out.println("threadnum:" + threadnum);
    Thread.sleep(1000); // Simulate the time-consuming operation of a request
  }
}
```

The `acquire()` method blocks until a permit is available, and then it takes one permit. Each `release()` method increases the permit count, potentially releasing a blocked `acquire()` method. However, there are no actual permit objects; the `Semaphore` just maintains a count of available permits. `Semaphore` is often used to limit the number of threads accessing a certain resource.

You can also acquire and release multiple permits at once, though it's generally not necessary:

```java
semaphore.acquire(5); // Acquire 5 permits, so the number of runnable threads is 20/5=4
test(threadnum);
semaphore.release(5); // Release 5 permits
```

Besides the `acquire()` method, another commonly used method is `tryAcquire()`, which immediately returns false if it cannot obtain a permit.

[Issue 645 Additional Content](https://github.com/Snailclimb/JavaGuide/issues/645):

> Like `CountDownLatch`, `Semaphore` is also an implementation of a shared lock. It initializes the `state` of AQS (AbstractQueuedSynchronizer) to `permits`. When the number of threads executing tasks exceeds `permits`, the extra threads will be placed in a waiting queue (`Park`) and will spin to check if `state` is greater than 0. Only when `state` is greater than 0 can the blocked threads continue to execute. At this point, the threads executing the task continue to execute the `release()` method, which increments the `state` variable by 1, allowing the spinning threads to proceed.
> Thus, at most, the number of threads that can spin successfully is limited to the `permits` count, controlling the number of threads executing the task.

### CountDownLatch (Countdown Timer)

#### Introduction

`CountDownLatch` allows `count` threads to block at one point until all threads have finished their tasks.

`CountDownLatch` is one-time use; its counter can only be initialized once in the constructor, with no mechanism to reset it. Once `CountDownLatch` is used, it cannot be reused.

#### Principle

`CountDownLatch` is a shared lock implementation, initializing the AQS `state` to `count`, as seen in its constructor.

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}

private static final class Sync extends AbstractQueuedSynchronizer {
    Sync(int count) {
        setState(count);
    }
  //...
}
```

When a thread calls `countDown()`, it uses the `tryReleaseShared` method with a CAS operation to decrease the `state`, until `state` is 0. When `state` is 0, it means all threads have called `countDown`, and the threads waiting on the `CountDownLatch` will be awakened to continue execution.

```java
public void countDown() {
    sync.releaseShared(1);
}
```

The `releaseShared` method is the default implementation in `AbstractQueuedSynchronizer`.

```java
// Release shared lock
// If tryReleaseShared returns true, it wakes up one or more threads in the waiting queue.
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
      doReleaseShared();
      return true;
    }
    return false;
}
```

The `tryReleaseShared` method is overridden in the `Sync` inner class of `CountDownLatch`, whereas the default implementation in `AbstractQueuedSynchronizer` throws `UnsupportedOperationException`.

```java
// Decrease state until it reaches 0;
// Only when count decreases to 0, countDown returns true.
protected boolean tryReleaseShared(int releases) {
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

Taking the parameterless `await` method as an example, when `await()` is called, if `state` is not 0, it means the tasks are not completed yet, and `await()` will block (adding the main thread to the waiting queue, the CLH queue). Then, `CountDownLatch` will spin with CAS to check `state == 0`; if `state == 0`, it will release all waiting threads, allowing the code following `await()` to execute.

```java
// Wait (can also be called lock)
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
// Wait with a timeout
public boolean await(long timeout, TimeUnit unit)
    throws InterruptedException {
    return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
}
```

The `acquireSharedInterruptibly` method is the default implementation in `AbstractQueuedSynchronizer`.

```java
// Try to acquire lock, if successful return, otherwise add to waiting queue and suspend thread
public final void acquireSharedInterruptibly(int arg)
    throws InterruptedException {
    if (Thread.interrupted())
      throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
      doAcquireSharedInterruptibly(arg);
}
```

The `tryAcquireShared` method is overridden in the `Sync` inner class of `CountDownLatch`, determining if `state` is 0. If true, it returns 1; otherwise, it returns -1.

```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
    }
```

### Practical Example

**Two Typical Uses of CountDownLatch**:

1. **A thread waits for n threads to complete before starting:** Initialize the `CountDownLatch` counter to n (`new CountDownLatch(n)`), and each time a task thread completes, decrement the counter by 1 (`countDownLatch.countDown()`). When the counter reaches 0, the thread waiting on the `CountDownLatch` (`await()`) is awakened. A typical use case is when starting a service, the main thread needs to wait for multiple components to load before continuing.
2. **Achieving maximum parallelism for task execution by multiple threads:** Note that this is about parallelism, not concurrency, emphasizing multiple threads starting at the exact same time. It's like a race where all threads are at the starting line waiting for the gun to fire, then all start simultaneously. To achieve this, initialize a shared `CountDownLatch` object with the counter set to 1 (`new CountDownLatch(1)`). All threads call `countDownLatch.await()` before starting their tasks. When the main thread calls `countDown()`, the counter becomes 0, and all threads are awakened simultaneously.

**CountDownLatch Code Example**:

```java
public class CountDownLatchExample {
  // Number of requests
  private static final int THREAD_COUNT = 550;

  public static void main(String[] args) throws InterruptedException {
    // Create a thread pool with a fixed number of threads (execution will be slow if the thread count is too low)
    // This is just for testing; in a real scenario, manually set thread pool parameters
    ExecutorService threadPool = Executors.newFixedThreadPool(300);
    final CountDownLatch countDownLatch = new CountDownLatch(THREAD_COUNT);
    for (int i = 0; i < THREAD_COUNT; i++) {
      final int threadNum = i;
      threadPool.execute(() -> {
        try {
          test(threadNum);
        } catch (InterruptedException e) {
          e.printStackTrace();
        } finally {
          // Indicate that a request has been completed
          countDownLatch.countDown();
        }
      });
    }
    countDownLatch.await();
    threadPool.shutdown();
    System.out.println("finish");
  }

  public static void test(int threadnum) throws InterruptedException {
    Thread.sleep(1000);
    System.out.println("threadNum:" + threadnum);
    Thread.sleep(1000);
  }
}
```

In the above code, we defined the number of requests as 550. The message `System.out.println("finish");` will only be printed after all 550 requests have been processed.

The first interaction with `CountDownLatch` is the main thread waiting for other threads. The main thread must call `CountDownLatch.await()` immediately after starting other threads. This causes the main thread to block on this method until other threads complete their tasks.

The other N threads must reference the latch object because they need to notify the `CountDownLatch` object that they have completed their tasks. This notification mechanism is achieved by calling `CountDownLatch.countDown()`. Each time this method is called, the count initialized in the constructor decreases by 1. When all N threads have called this method, the count reaches 0, and the main thread, having called `await()`, resumes its execution.

**Caution:** Misusing the `await()` method of `CountDownLatch` can easily lead to deadlock. For example, if we change the for loop in the above code to:

```java
for (int i = 0; i < threadCount - 1; i++) {
  // .......
}
```

This change would prevent the count from reaching 0, resulting in indefinite waiting.

### CyclicBarrier (Cyclic Barrier)

#### Introduction

`CyclicBarrier` is very similar to `CountDownLatch` in that it also allows threads to wait for each other. However, it is more complex and powerful than `CountDownLatch`. The main application scenarios are similar to those of `CountDownLatch`.

> While `CountDownLatch` is based on AQS (AbstractQueuedSynchronizer), `CyclicBarrier` is based on `ReentrantLock` (which also belongs to AQS synchronizers) and `Condition`.

The literal meaning of `CyclicBarrier` is a barrier (Barrier) that can be reused cyclically (Cyclic). What it does is block a group of threads when they reach a barrier (also called a synchronization point) until the last thread arrives, at which point the barrier opens, and all the blocked threads continue their work.

#### Principle

Inside `CyclicBarrier`, a `count` variable is used as a counter. The initial value of `count` is set to the initialization value of the `parties` attribute. Each time a thread reaches the barrier, the counter is decremented by 1. If the `count` value reaches 0, it indicates that the current generation's last thread has arrived at the barrier, and the task specified in the constructor is attempted to be executed.

```java
// Number of threads to intercept each time
private final int parties;
// Counter
private int count;
```

Let's briefly look at the source code.

1. The default constructor for `CyclicBarrier` is `CyclicBarrier(int parties)`, where the parameter represents the number of threads to intercept. Each thread calls the `await()` method to inform `CyclicBarrier` that it has reached the barrier, and then the current thread is blocked.

```java
public CyclicBarrier(int parties) {
    this(parties, null);
}

public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}
```

Here, `parties` represents the number of threads to be intercepted. When the number of intercepted threads reaches this value, the barrier opens, allowing all threads to pass through.

2. When a `CyclicBarrier` object calls the `await()` method, it actually calls the `dowait(false, 0L)` method. The `await()` method acts like setting up a barrier, blocking the thread. When the number of blocked threads reaches the value of `parties`, the barrier opens, and the threads proceed.

```java
public int await() throws InterruptedException, BrokenBarrierException {
  try {
      return dowait(false, 0L);
  } catch (TimeoutException toe) {
      throw new Error(toe); // cannot happen
  }
}
```

The source code analysis of the `dowait(false, 0L)` method is as follows:

```java
// When the number of threads or requests reaches the count, the method after await will be executed. In the example above, the value of count is 5.
private int count;
/**
* Main barrier code, covering the various policies.
  */
  private int dowait(boolean timed, long nanos)
  throws InterruptedException, BrokenBarrierException, TimeoutException {
  final ReentrantLock lock = this.lock;
  // Lock
  lock.lock();
  try {
  final Generation g = generation;

       if (g.broken)
           throw new BrokenBarrierException();

       // If the thread is interrupted, throw an exception
       if (Thread.interrupted()) {
           breakBarrier();
           throw new InterruptedException();
       }
       // Decrement count
       int index = --count;
       // When count is decremented to 0, it indicates the last thread has reached the barrier, satisfying the condition to execute the method after await
       if (index == 0) {  // tripped
           boolean ranAction = false;
           try {
               final Runnable command = barrierCommand;
               if (command != null)
                   command.run();
               ranAction = true;
               // Reset count to the initial value of parties
               // Wake up previously waiting threads
               // Start the next execution cycle
               nextGeneration();
               return 0;
           } finally {
               if (!ranAction)
                   breakBarrier();
           }
       }

       // Loop until tripped, broken, interrupted, or timed out
       for (;;) {
           try {
               if (!timed)
                   trip.await();
               else if (nanos > 0L)
                   nanos = trip.awaitNanos(nanos);
           } catch (InterruptedException ie) {
               if (g == generation && !g.broken) {
                   breakBarrier();
                   throw ie;
               } else {
                   // We're about to finish waiting even if we had not
                   // been interrupted, so this interrupt is deemed to
                   // "belong" to subsequent execution.
                   Thread.currentThread().interrupt();
               }
           }

           if (g.broken)
               throw new BrokenBarrierException();

           if (g != generation)
               return index;

           if (timed && nanos <= 0L) {
               breakBarrier();
               throw new TimeoutException();
           }
       }
      } finally {
          lock.unlock();
          }
      }
```

### Practical Application

#### Example 1:

```java
public class CyclicBarrierExample1 {
  // Number of requests
  private static final int threadCount = 550;
  // Number of threads required for synchronization
  private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5);

  public static void main(String[] args) throws InterruptedException {
    // Create thread pool
    ExecutorService threadPool = Executors.newFixedThreadPool(10);

    for (int i = 0; i < threadCount; i++) {
      final int threadNum = i;
      Thread.sleep(1000);
      threadPool.execute(() -> {
        try {
          test(threadNum);
        } catch (InterruptedException e) {
          e.printStackTrace();
        } catch (BrokenBarrierException e) {
          e.printStackTrace();
        }
      });
    }
    threadPool.shutdown();
  }

  public static void test(int threadnum) throws InterruptedException, BrokenBarrierException {
    System.out.println("threadnum: " + threadnum + " is ready");
    try {
      /** Wait for 60 seconds to ensure the threads fully complete execution */
      cyclicBarrier.await(60, TimeUnit.SECONDS);
    } catch (Exception e) {
      System.out.println("-----CyclicBarrierException------");
    }
    System.out.println("threadnum: " + threadnum + " is finish");
  }
}
```

Output:

```plain
threadnum: 0 is ready
threadnum: 1 is ready
threadnum: 2 is ready
threadnum: 3 is ready
threadnum: 4 is ready
threadnum: 4 is finish
threadnum: 0 is finish
threadnum: 1 is finish
threadnum: 2 is finish
threadnum: 3 is finish
threadnum: 5 is ready
threadnum: 6 is ready
threadnum: 7 is ready
threadnum: 8 is ready
threadnum: 9 is ready
threadnum: 9 is finish
threadnum: 5 is finish
threadnum: 8 is finish
threadnum: 7 is finish
threadnum: 6 is finish
...
```

We can observe that when the number of threads, which is also the number of requests, reaches the defined 5, the methods following the `await()` call are executed.

Additionally, `CyclicBarrier` provides a more advanced constructor `CyclicBarrier(int parties, Runnable barrierAction)` that allows a `barrierAction` to be executed first when the threads reach the barrier, facilitating more complex business scenarios.

#### Example 2:

```java
public class CyclicBarrierExample2 {
  // Number of requests
  private static final int threadCount = 550;
  // Number of threads required for synchronization
  private static final CyclicBarrier cyclicBarrier = new CyclicBarrier(5, () -> {
    System.out.println("------Priority execution when the number of threads reaches the barrier------");
  });

  public static void main(String[] args) throws InterruptedException {
    // Create thread pool
    ExecutorService threadPool = Executors.newFixedThreadPool(10);

    for (int i = 0; i < threadCount; i++) {
      final int threadNum = i;
      Thread.sleep(1000);
      threadPool.execute(() -> {
        try {
          test(threadNum);
        } catch (InterruptedException e) {
          e.printStackTrace();
        } catch (BrokenBarrierException e) {
          e.printStackTrace();
        }
      });
    }
    threadPool.shutdown();
  }

  public static void test(int threadnum) throws InterruptedException, BrokenBarrierException {
    System.out.println("threadnum: " + threadnum + " is ready");
    cyclicBarrier.await();
    System.out.println("threadnum: " + threadnum + " is finish");
  }
}
```

Output:

```plain
threadnum: 0 is ready
threadnum: 1 is ready
threadnum: 2 is ready
threadnum: 3 is ready
threadnum: 4 is ready
------Priority execution when the number of threads reaches the barrier------
threadnum: 4 is finish
threadnum: 0 is finish
threadnum: 2 is finish
threadnum: 1 is finish
threadnum: 3 is finish
threadnum: 5 is ready
threadnum: 6 is ready
threadnum: 7 is ready
threadnum: 8 is ready
threadnum: 9 is ready
------Priority execution when the number of threads reaches the barrier------
threadnum: 9 is finish
threadnum: 5 is finish
threadnum: 6 is finish
threadnum: 8 is finish
threadnum: 7 is finish
...

