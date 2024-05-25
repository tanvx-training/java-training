In a project development scenario, it's quite common for an interface to need to call the interfaces of N other services. For example, when a user requests to retrieve order information, it may be necessary to call interfaces such as user information, product details, logistics information, and product recommendations, and then consolidate the data for unified return.

If executed serially (performing each task one after the other in order), the response speed of the interface will be very slow. Considering that most of these interfaces are **not sequentially dependent**, they can be **executed in parallel**. For instance, when calling to get product details, logistics information can be retrieved simultaneously. By executing multiple tasks in parallel, the response speed of the interface can be greatly optimized.

For interface calls with sequential dependencies, arrangements can be made as shown in the diagram below:

![serial-to-parallel](https://oss.javaguide.cn/github/javaguide/high-performance/serial-to-parallel.png)

1. User information must be obtained before calling the interfaces for product details and logistics information.
2. Product recommendations can only be called after successfully obtaining product details and logistics information.

For Java programs, `CompletableFuture`, introduced in Java 8, can help us orchestrate multiple tasks, and its functionality is very powerful.

This article serves as a simple introduction to `CompletableFuture`, taking you through the commonly used APIs.

## Introduction to Future

The `Future` class is a typical application of asynchronous thinking, mainly used in scenarios where time-consuming tasks need to be executed to avoid the program waiting idly for the completion of these tasks, which would result in low efficiency. Specifically, when we execute a time-consuming task, we can delegate this task to a separate thread for asynchronous execution, while we can proceed with other tasks without waiting for the time-consuming task to complete. After finishing our tasks, we can retrieve the result of the time-consuming task through the `Future` class. This significantly improves the efficiency of program execution.

This is actually the classic **Future pattern** in multi-threading, which can be seen as a design pattern. Its core idea is asynchronous invocation, primarily used in the multi-threading domain and not exclusive to the Java language.

In Java, the `Future` class is just a generic interface located in the `java.util.concurrent` package, defining five methods, mainly including these four functionalities:

- Cancelling a task
- Checking if a task has been cancelled
- Checking if a task has completed execution
- Getting the result of the task execution

```java
// V represents the type of the result returned by the Future execution
public interface Future<V> {
    // Cancels the task execution
    // Returns true if successful, otherwise false
    boolean cancel(boolean mayInterruptIfRunning);
    // Checks if the task has been cancelled
    boolean isCancelled();
    // Checks if the task has completed execution
    boolean isDone();
    // Gets the result of the task execution
    V get() throws InterruptedException, ExecutionException;
    // Throws TimeoutException if the result is not returned within the specified time
    V get(long timeout, TimeUnit unit) throws InterruptedException, ExecutionException, TimeoutException;
}
```

In simple terms: I have a task, which I submit to `Future` for processing. During the task execution, I can do anything I want. Additionally, during this period, I can cancel the task and check its execution status. After a while, I can directly retrieve the result of the task from `Future`.

## Introduction to CompletableFuture

In practical usage, `Future` has some limitations, such as lack of support for asynchronous task orchestration and the blocking nature of the `get()` method for obtaining computation results.

Java 8 introduced the `CompletableFuture` class to address these shortcomings of `Future`. In addition to providing more user-friendly and powerful features than `Future`, `CompletableFuture` also offers functional programming and the ability to orchestrate asynchronous tasks (multiple asynchronous tasks can be chained together to form a complete chain of calls).

Let's take a brief look at the definition of the `CompletableFuture` class.

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {
}
```

As you can see, `CompletableFuture` implements both the `Future` and `CompletionStage` interfaces.

![CompletableFuture Class Diagram](https://oss.javaguide.cn/github/javaguide/java/concurrent/completablefuture-class-diagram.jpg)

The `CompletionStage` interface describes a stage of an asynchronous computation. Many computations can be divided into multiple stages or steps. In such cases, all steps can be combined using this interface to form a pipeline of asynchronous computations.

Besides providing more user-friendly and powerful features than `Future`, `CompletableFuture` also offers functional programming capabilities.

![CompletableFuture Functional Capabilities](https://oss.javaguide.cn/javaguide/image-20210902092441434.png)

The `Future` interface has 5 methods:

- `boolean cancel(boolean mayInterruptIfRunning)`: Attempts to cancel the execution of the task.
- `boolean isCancelled()`: Checks if the task has been cancelled.
- `boolean isDone()`: Checks if the task has been completed.
- `get()`: Waits for the task to complete execution and retrieves the computation result.
- `get(long timeout, TimeUnit unit)`: Adds a timeout for waiting.

The `CompletionStage` interface describes a stage of an asynchronous computation. Many computations can be divided into multiple stages or steps. In such cases, all steps can be combined using this interface to form a pipeline of asynchronous computations.

The `CompletionStage` interface has many methods, and the functional capabilities of `CompletableFuture` are provided by this interface. You can see that it heavily utilizes the functional programming features introduced in Java 8 from the method parameters of this interface.

![CompletionStage Functional Capabilities](https://oss.javaguide.cn/javaguide/image-20210902093026059.png)

Due to the numerous methods, I won't explain them all here. In the following sections, I will introduce the usage of most commonly used methods.

### 常见操作

#### Creating CompletableFuture

Common methods for creating `CompletableFuture` objects include:

1. Using the `new` keyword.
2. Utilizing the static factory methods provided by `CompletableFuture`: `runAsync()` and `supplyAsync()`.

##### Using the `new` Keyword

Creating a `CompletableFuture` object using the `new` keyword can be considered using `CompletableFuture` as a `Future`.

I used this approach to create `CompletableFuture` objects in my open-source project [guide-rpc-framework](https://github.com/Snailclimb/guide-rpc-framework).

Here's a simple example:

We create a `CompletableFuture` with a result value type of `RpcResponse<Object>`. You can consider `resultFuture` as a carrier for the asynchronous computation result.

```java
CompletableFuture<RpcResponse<Object>> resultFuture = new CompletableFuture<>();
```

Assuming that at some point in the future, we get the final result. We can then call the `complete()` method to pass the result, indicating that `resultFuture` has been completed.

```java
// The complete() method can only be called once, subsequent calls will be ignored.
resultFuture.complete(rpcResponse);
```

You can check if it's completed using the `isDone()` method.

```java
public boolean isDone() {
    return result != null;
}
```

Retrieving the result of the asynchronous computation is straightforward, just call the `get()` method. The thread calling `get()` will block until the `CompletableFuture` completes its computation.

```java
rpcResponse = completableFuture.get();
```

If you already know the result of the computation, you can use the `completedFuture()` static method to create a `CompletableFuture`.

```java
CompletableFuture<String> future = CompletableFuture.completedFuture("hello!");
assertEquals("hello!", future.get());
```

The `completedFuture()` method internally calls the parameterized `new` method, but this method is not exposed externally.

```java
public static <U> CompletableFuture<U> completedFuture(U value) {
    return new CompletableFuture<U>((value == null) ? NIL : value);
}
```

##### Static Factory Methods

These two methods help us encapsulate the computation logic.

```java
static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);
// Using a custom thread pool (recommended)
static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor);
static CompletableFuture<Void> runAsync(Runnable runnable);
// Using a custom thread pool (recommended)
static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor);
```

The `runAsync()` method accepts a `Runnable` parameter, which is a functional interface that does not allow return values. You can use `runAsync()` when you need asynchronous operation without caring about the result.

```java
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```

The `supplyAsync()` method accepts a `Supplier<U>` parameter, which is also a functional interface. `U` is the type of the result value.

```java
@FunctionalInterface
public interface Supplier<T> {

    /**
     * Gets a result.
     *
     * @return a result
     */
    T get();
}
```

You can use `supplyAsync()` when you need asynchronous operation and care about the result.

```java
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> System.out.println("hello!"));
    future.get(); // Outputs "hello!"
    CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> "hello!");
    assertEquals("hello!", future2.get());
```

### Handling the Result of Asynchronous Computations

After obtaining the result of an asynchronous computation, we can further process it. Some commonly used methods for this purpose are:

- `thenApply()`
- `thenAccept()`
- `thenRun()`
- `whenComplete()`

The `thenApply()` method accepts a `Function` instance to process the result.

```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn);
```

The `thenApply()` method is used as follows:

```java
CompletableFuture<String> future = CompletableFuture.completedFuture("hello!")
        .thenApply(s -> s + "world!");
assertEquals("hello!world!", future.get());
// This call will be ignored.
future.thenApply(s -> s + "nice!");
assertEquals("hello!world!", future.get());
```

You can also chain calls in a **stream-like manner**:

```java
CompletableFuture<String> future = CompletableFuture.completedFuture("hello!")
        .thenApply(s -> s + "world!").thenApply(s -> s + "nice!");
assertEquals("hello!world!nice!", future.get());
```

**If you don't need to retrieve the result from the callback function, you can use `thenAccept()` or `thenRun()`. The difference between these two methods is that `thenRun()` cannot access the result of the asynchronous computation.**

The parameter of the `thenAccept()` method is `Consumer<? super T>`.

```java
public CompletableFuture<Void> thenAccept(Consumer<? super T> action);
```

As the name suggests, `Consumer` is a consumer interface that accepts one input object and then "consumes" it.

```java
@FunctionalInterface
public interface Consumer<T> {

    void accept(T t);

    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

The parameter of the `thenRun()` method is `Runnable`.

```java
public CompletableFuture<Void> thenRun(Runnable action);
```

The `thenAccept()` and `thenRun()` methods are used as follows:

```java
CompletableFuture.completedFuture("hello!")
        .thenApply(s -> s + "world!").thenApply(s -> s + "nice!").thenAccept(System.out::println); // Outputs "hello!world!nice!"

CompletableFuture.completedFuture("hello!")
        .thenApply(s -> s + "world!").thenApply(s -> s + "nice!").thenRun(() -> System.out.println("hello!")); // Outputs "hello!"
```

The `whenComplete()` method's parameter is `BiConsumer<? super T, ? super Throwable>`.

```java
public CompletableFuture<T> whenComplete(BiConsumer<? super T, ? super Throwable> action);
```

Compared to `Consumer`, `BiConsumer` accepts two input objects and then "consumes" them.

```java
@FunctionalInterface
public interface BiConsumer<T, U> {
    void accept(T t, U u);

    default BiConsumer<T, U> andThen(BiConsumer<? super T, ? super U> after) {
        Objects.requireNonNull(after);

        return (l, r) -> {
            accept(l, r);
            after.accept(l, r);
        };
    }
}
```

The `whenComplete()` method is used as follows:

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> "hello!")
        .whenComplete((res, ex) -> {
            // res represents the result returned
            // ex is of type Throwable, representing the thrown exception (if any)
            System.out.println(res);
            // There is no exception thrown here, so it's null
            assertNull(ex);
        });
assertEquals("hello!", future.get());
```

### Exception Handling

You can handle possible exceptions thrown during task execution using the `handle()` method.

```java
public <U> CompletableFuture<U> handle(
    BiFunction<? super T, Throwable, ? extends U> fn) {
    return uniHandleStage(null, fn);
}

public <U> CompletableFuture<U> handleAsync(
    BiFunction<? super T, Throwable, ? extends U> fn) {
    return uniHandleStage(defaultExecutor(), fn);
}

public <U> CompletableFuture<U> handleAsync(
    BiFunction<? super T, Throwable, ? extends U> fn, Executor executor) {
    return uniHandleStage(screenExecutor(executor), fn);
}
```

Here's an example code:

```java
CompletableFuture<String> future
        = CompletableFuture.supplyAsync(() -> {
    if (true) {
        throw new RuntimeException("Computation error!");
    }
    return "hello!";
}).handle((res, ex) -> {
    // res represents the returned result
    // ex is of type Throwable, representing the thrown exception
    return res != null ? res : "world!";
});
assertEquals("world!", future.get());
```

You can also handle exceptions using the `exceptionally()` method.

```java
CompletableFuture<String> future
        = CompletableFuture.supplyAsync(() -> {
    if (true) {
        throw new RuntimeException("Computation error!");
    }
    return "hello!";
}).exceptionally(ex -> {
    System.out.println(ex.toString());// CompletionException
    return "world!";
});
assertEquals("world!", future.get());
```

If you want the result of `CompletableFuture` to be an exception, you can use the `completeExceptionally()` method to assign it.

```java
CompletableFuture<String> completableFuture = new CompletableFuture<>();
// ...
    completableFuture.completeExceptionally(
    new RuntimeException("Calculation failed!"));
// ...
    completableFuture.get(); // ExecutionException
```

### Combining CompletableFuture

You can use `thenCompose()` to sequentially link two `CompletableFuture` objects, creating an asynchronous task chain. Its purpose is to use the result of the previous task as the input parameter for the next task, thus forming a dependency.

```java
public <U> CompletableFuture<U> thenCompose(
    Function<? super T, ? extends CompletionStage<U>> fn) {
    return uniComposeStage(null, fn);
}

public <U> CompletableFuture<U> thenComposeAsync(
    Function<? super T, ? extends CompletionStage<U>> fn) {
    return uniComposeStage(defaultExecutor(), fn);
}

public <U> CompletableFuture<U> thenComposeAsync(
    Function<? super T, ? extends CompletionStage<U>> fn,
    Executor executor) {
    return uniComposeStage(screenExecutor(executor), fn);
}
```

Here's how to use the `thenCompose()` method:

```java
CompletableFuture<String> future
        = CompletableFuture.supplyAsync(() -> "hello!")
        .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + "world!"));
assertEquals("hello!world!", future.get());
```

In practical development, this method is quite useful. For instance, if `task1` and `task2` are both executed asynchronously but `task2` must start after `task1` completes (where `task2` depends on the result of `task1`).

Similar to `thenCompose()`, there's `thenCombine()` method, which also combines two `CompletableFuture` objects.

```java
CompletableFuture<String> completableFuture
        = CompletableFuture.supplyAsync(() -> "hello!")
        .thenCombine(CompletableFuture.supplyAsync(
                () -> "world!"), (s1, s2) -> s1 + s2)
        .thenCompose(s -> CompletableFuture.supplyAsync(() -> s + "nice!"));
assertEquals("hello!world!nice!", completableFuture.get());
```

**So, what's the difference between `thenCompose()` and `thenCombine()`?**

- `thenCompose()` links two `CompletableFuture` objects, using the result of the previous task as the input parameter for the next task, and there's a sequential order between them.
- `thenCombine()` merges the results of two tasks after both have completed. Both tasks run in parallel without a specific ordering between them.

Apart from `thenCompose()` and `thenCombine()`, there are other methods for combining `CompletableFuture` to achieve different effects, meeting various business requirements.

For example, if we want to execute `task3` after either `task1` or `task2` completes, we can use `acceptEither()`.

```java
public CompletableFuture<Void> acceptEither(
    CompletionStage<? extends T> other, Consumer<? super T> action) {
    return orAcceptStage(null, other, action);
}

public CompletableFuture<Void> acceptEitherAsync(
    CompletionStage<? extends T> other, Consumer<? super T> action) {
    return orAcceptStage(asyncPool, other, action);
}
```

Here's a simple example:

```java
CompletableFuture<String> task = CompletableFuture.supplyAsync(() -> {
    System.out.println("Task 1 started, time: " + System.currentTimeMillis());
    try {
        Thread.sleep(500);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("Task 1 finished, time: " + System.currentTimeMillis());
    return "task1";
});

CompletableFuture<String> task2 = CompletableFuture.supplyAsync(() -> {
    System.out.println("Task 2 started, time: " + System.currentTimeMillis());
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    System.out.println("Task 2 finished, time: " + System.currentTimeMillis());
    return "task2";
});

task.acceptEitherAsync(task2, (res) -> {
    System.out.println("Task 3 started, time: " + System.currentTimeMillis());
    System.out.println("Result of the previous task: " + res);
});

// Adding some delay to ensure enough time for asynchronous tasks to complete
try {
    Thread.sleep(2000);
} catch (InterruptedException e) {
    e.printStackTrace();
}
```

Output:

```plain
Task 1 started, time: 1695088058520
Task 2 started, time: 1695088058521
Task 1 finished, time: 1695088059023
Task 3 started, time: 1695088059023
Result of the previous task: task1
Task 2 finished, time: 1695088059523
```

The task combination operation `acceptEitherAsync()` triggers the execution of `task3` when either asynchronous `task1` or `task2` completes, but it's important to note that the trigger timing is indeterminate. If both `task1` and `task2` haven't completed, `task3` won't be executed yet.

### Running Multiple CompletableFuture in Parallel

You can use the `allOf()` static method of `CompletableFuture` to run multiple `CompletableFutures` in parallel.

In real projects, we often need to run multiple unrelated tasks in parallel, where these tasks have no dependency on each other and can run independently.

For instance, if we need to read and process 6 files, where these 6 tasks are independent and have no execution order dependency, but we need to aggregate and summarize the results of processing these files before returning to the user. In such cases, we can use parallel execution of multiple `CompletableFutures` to handle this.

Here's an example code:

```java
CompletableFuture<Void> task1 =
  CompletableFuture.supplyAsync(()->{
    // Custom business operations
  });
// More tasks...
CompletableFuture<Void> task6 =
  CompletableFuture.supplyAsync(()->{
    // Custom business operations
  });

CompletableFuture<Void> allTasksFuture = CompletableFuture.allOf(task1, /*...*/, task6);

try {
    allTasksFuture.join();
} catch (Exception ex) {
    // Handle exceptions
}
System.out.println("All tasks done.");
```

`allOf()` method is often compared with `anyOf()` method.

**`allOf()` method waits for all `CompletableFutures` to complete before returning.**

```java
Random rand = new Random();
CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(1000 + rand.nextInt(1000));
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        System.out.println("Future 1 done...");
    }
    return "abc";
});
CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(1000 + rand.nextInt(1000));
    } catch (InterruptedException e) {
        e.printStackTrace();
    } finally {
        System.out.println("Future 2 done...");
    }
    return "efg";
});
```

Calling `join()` ensures the program waits for both `future1` and `future2` to complete before continuing execution.

```java
CompletableFuture<Void> allFutures = CompletableFuture.allOf(future1, future2);
allFutures.join();
assertTrue(allFutures.isDone());
System.out.println("All futures done...");
```

Output:

```plain
Future 1 done...
Future 2 done...
All futures done...
```

**`anyOf()` method doesn't wait for all `CompletableFutures` to complete, it returns as soon as any one of them completes!**

```java
CompletableFuture<Object> anyOfFuture = CompletableFuture.anyOf(future1, future2);
System.out.println(anyOfFuture.get());
```

Output could be either:

```plain
Future 2 done...
efg
```

or:

```plain
Future 1 done...
abc
```

### Recommendations for Using CompletableFuture

### Use Custom Thread Pool

In our previous code examples, for simplicity, we didn't choose a custom thread pool. However, in real projects, this is not advisable.

`CompletableFuture` by default uses `ForkJoinPool.commonPool()` as the executor, which is a globally shared thread pool and may be occupied by other tasks, leading to performance degradation or starvation. Therefore, it's recommended to use a custom thread pool to execute asynchronous tasks with `CompletableFuture`, which can improve concurrency and flexibility.

```java
private ThreadPoolExecutor executor = new ThreadPoolExecutor(10, 10,
        0L, TimeUnit.MILLISECONDS,
        new LinkedBlockingQueue<Runnable>());

CompletableFuture.runAsync(() -> {
     //...
}, executor);
```

### Avoid Using get() Whenever Possible

The `get()` method of `CompletableFuture` is blocking, so it's best to avoid using it whenever possible. If you must use it, ensure to add a timeout, or else the main thread might wait indefinitely, unable to execute other tasks.

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(10_000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return "Hello, world!";
});

// Get the result of the asynchronous task with a timeout of 5 seconds
try {
    String result = future.get(5, TimeUnit.SECONDS);
    System.out.println(result);
} catch (InterruptedException | ExecutionException | TimeoutException e) {
    // Handle exceptions
    e.printStackTrace();
}
```

The above code throws a `TimeoutException` when calling `get()`. This allows us to handle the exception appropriately, such as canceling the task, retrying the task, logging, etc.

### Proper Exception Handling

When using `CompletableFuture`, always handle exceptions correctly to avoid swallowing or losing exceptions, which could lead to unpredictable issues.

Here are some suggestions:

- Use the `whenComplete` method to trigger a callback when the task completes and handle exceptions properly, rather than swallowing or losing exceptions.
- Use the `exceptionally` method to handle exceptions and rethrow them so that exceptions can propagate to subsequent stages instead of being ignored or terminated.
- Use the `handle` method to handle both normal results and exceptions, returning a new result instead of letting exceptions affect normal business logic.
- Use `CompletableFuture.allOf` to combine multiple `CompletableFutures` and uniformly handle exceptions from all tasks, instead of having overly long or repetitive exception handling.
- ...

### Combine Multiple Asynchronous Tasks Appropriately

Correctly use methods like `thenCompose()`, `thenCombine()`, `acceptEither()`, `allOf()`, `anyOf()`, etc., to combine multiple asynchronous tasks according to the actual needs of the business, improving program execution efficiency.

In actual usage, we can also leverage or refer to existing asynchronous task orchestration frameworks, such as JD's [asyncTool](https://gitee.com/jd-platform-opensource/asyncTool).

![asyncTool README Document](https://oss.javaguide.cn/github/javaguide/java/concurrent/asyncTool-readme.png)