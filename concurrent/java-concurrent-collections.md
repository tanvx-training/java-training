JDK provides most of these containers in the `java.util.concurrent` package.

- **`ConcurrentHashMap`**: A thread-safe `HashMap`.
- **`CopyOnWriteArrayList`**: A thread-safe `List` that performs exceptionally well in read-heavy and write-light scenarios, far better than `Vector`.
- **`ConcurrentLinkedQueue`**: An efficient concurrent queue implemented using a linked list. It can be considered a thread-safe `LinkedList` and is a non-blocking queue.
- **`BlockingQueue`**: This is an interface, and the JDK implements this interface using various methods such as linked lists and arrays. It represents a blocking queue, making it very suitable for use as a data-sharing channel.
- **`ConcurrentSkipListMap`**: An implementation of a skip list. This is a Map that uses the skip list data structure for fast lookups.

## ConcurrentHashMap

We know that `HashMap` is not thread-safe. In concurrent scenarios, one feasible way to ensure thread safety is to use the `Collections.synchronizedMap()` method to wrap our `HashMap`. However, this approach synchronizes concurrent access between different threads using a global lock, which brings significant performance issues.

Thus, the thread-safe version of `HashMap`—`ConcurrentHashMap`—was born.

In JDK 1.7, `ConcurrentHashMap` segmented the entire bucket array into segments, with each lock only locking a part of the container's data (see the diagram below). When multiple threads access different data segments in the container, there is no lock contention, improving concurrent access rates.

By JDK 1.8, `ConcurrentHashMap` had abandoned the concept of segments, instead using a data structure of `Node` arrays + linked lists + red-black trees to achieve concurrency control, employing `synchronized` and CAS operations (synchronized locks have been significantly optimized since JDK 1.6). It essentially looks like an optimized and thread-safe `HashMap`. Although the `Segment` data structure can still be seen in JDK 1.8, it has been simplified in attributes, only to maintain compatibility with older versions.

For a detailed introduction to `ConcurrentHashMap`, please refer to the article I wrote: [`ConcurrentHashMap` Source Code Analysis](./../collection/concurrent-hash-map-source-code.md).

## CopyOnWriteArrayList

Before JDK 1.5, if you wanted to use a thread-safe `List`, you could only choose `Vector`. However, `Vector` is an outdated collection and has been deprecated. `Vector` adds `synchronized` to its methods for adding, deleting, updating, and querying. Although this approach ensures synchronization, it essentially puts a big lock on the entire `Vector`, meaning every method execution requires acquiring the lock, leading to very poor performance.

JDK 1.5 introduced the `java.util.concurrent` (JUC) package, which provides many thread-safe and high-performance concurrent containers, among which the only thread-safe `List` implementation is `CopyOnWriteArrayList`.

For most business scenarios, read operations far outnumber write operations. Since read operations do not modify the existing data, locking for each read operation is a waste of resources. In contrast, we should allow multiple threads to access the internal data of the `List` simultaneously, as read operations are inherently safe.

This approach is very similar to the design philosophy of the `ReentrantReadWriteLock` read-write lock, where read-read operations are not mutually exclusive, read-write operations are mutually exclusive, and write-write operations are mutually exclusive (only read-read operations are not mutually exclusive). `CopyOnWriteArrayList` takes this idea further. To maximize read performance, read operations in `CopyOnWriteArrayList` do not require locking at all. Even better, write operations do not block read operations; only write-write operations are mutually exclusive. This significantly enhances the performance of read operations.

The core of `CopyOnWriteArrayList`'s thread safety lies in its **copy-on-write** strategy, which is evident from its name.

When modifying (`add`, `set`, `remove` operations) the content of `CopyOnWriteArrayList`, it does not directly modify the original array. Instead, it first creates a copy of the underlying array, modifies the copy, and then assigns the modified array back, ensuring that write operations do not affect read operations.

For a detailed introduction to `CopyOnWriteArrayList`, please refer to the article I wrote: [`CopyOnWriteArrayList` Source Code Analysis](./../collection/copyonwritearraylist-source-code.md).

## ConcurrentLinkedQueue

Java provides thread-safe `Queue`s which can be divided into **blocking queues** and **non-blocking queues**. The typical example of a blocking queue is `BlockingQueue`, and the typical example of a non-blocking queue is `ConcurrentLinkedQueue`. In practical applications, you should choose between a blocking queue or a non-blocking queue based on your actual needs. **Blocking queues can be implemented using locks, while non-blocking queues can be implemented using CAS operations.**

As the name suggests, `ConcurrentLinkedQueue` uses a linked list as its data structure. `ConcurrentLinkedQueue` is considered one of the best-performing queues in a highly concurrent environment. Its excellent performance is due to its complex internal implementation.

We won't analyze the internal code of `ConcurrentLinkedQueue` here; it suffices to know that `ConcurrentLinkedQueue` achieves thread safety mainly using CAS non-blocking algorithms.

`ConcurrentLinkedQueue` is suitable for scenarios requiring high performance and involving multiple threads reading and writing to the queue simultaneously. If the cost of locking the queue is high, it is suitable to use the lock-free `ConcurrentLinkedQueue` as a replacement.

## BlockingQueue

### Introduction to BlockingQueue

Earlier, we mentioned `ConcurrentLinkedQueue` as a high-performance non-blocking queue. Now, let's discuss the blocking queue—`BlockingQueue`. Blocking queues (`BlockingQueue`) are widely used in the "producer-consumer" problem because `BlockingQueue` provides methods for blocking insertion and removal. When the queue container is full, the producer thread will be blocked until the queue is not full; when the queue container is empty, the consumer thread will be blocked until the queue is not empty.

`BlockingQueue` is an interface that extends `Queue`, so its implementation classes can also be used as implementations of `Queue`, and `Queue` extends the `Collection` interface. Below are the implementation classes related to `BlockingQueue`:

![BlockingQueue Implementation Classes](https://oss.javaguide.cn/github/javaguide/java/51622268.jpg)

Next, we will introduce three common `BlockingQueue` implementation classes: `ArrayBlockingQueue`, `LinkedBlockingQueue`, and `PriorityBlockingQueue`.

### ArrayBlockingQueue

`ArrayBlockingQueue` is a bounded queue implementation class of the `BlockingQueue` interface, implemented using an array.

```java
public class ArrayBlockingQueue<E>
extends AbstractQueue<E>
implements BlockingQueue<E>, Serializable{}
```

Once created, the capacity of `ArrayBlockingQueue` cannot be changed. Its concurrency control uses a reentrant lock (`ReentrantLock`), meaning that both insertion and reading operations require acquiring the lock. When the queue is full, attempting to insert elements will block the operation; similarly, trying to retrieve an element from an empty queue will also block the operation.

By default, `ArrayBlockingQueue` does not guarantee fairness in thread access. Fairness means strictly following the order of absolute wait times for threads, ensuring that the thread that has waited the longest can access the `ArrayBlockingQueue` first. Non-fairness means the access order does not follow strict time order, and it is possible for a thread that has been blocked for a long time to still be unable to access the `ArrayBlockingQueue` when it becomes accessible. Ensuring fairness usually reduces throughput. If a fair `ArrayBlockingQueue` is needed, you can use the following code:

```java
private static ArrayBlockingQueue<Integer> blockingQueue = new ArrayBlockingQueue<Integer>(10, true);
```

### LinkedBlockingQueue

`LinkedBlockingQueue` is a blocking queue implemented using a **singly linked list**. It can be used as both a bounded and unbounded queue and also satisfies FIFO characteristics. Compared to `ArrayBlockingQueue`, it has higher throughput. To prevent `LinkedBlockingQueue` from rapidly increasing in size and consuming a large amount of memory, it is common to specify its size when creating a `LinkedBlockingQueue` object. If not specified, the capacity is equal to `Integer.MAX_VALUE`.

**Relevant constructors:**

```java
/**
 * In a sense, an unbounded queue
 * Creates a {@code LinkedBlockingQueue} with a capacity of
 * {@link Integer#MAX_VALUE}.
 */
public LinkedBlockingQueue() {
    this(Integer.MAX_VALUE);
}

/**
 * Bounded queue
 * Creates a {@code LinkedBlockingQueue} with the given (fixed) capacity.
 *
 * @param capacity the capacity of this queue
 * @throws IllegalArgumentException if {@code capacity} is not greater
 *         than zero
 */
public LinkedBlockingQueue(int capacity) {
    if (capacity <= 0) throw new IllegalArgumentException();
    this.capacity = capacity;
    last = head = new Node<E>(null);
}
```

### PriorityBlockingQueue

`PriorityBlockingQueue` is an unbounded blocking queue that supports priority. By default, elements are sorted in their natural order, but you can specify the element sorting order by implementing the `compareTo()` method in a custom class or by passing a `Comparator` parameter to the constructor.

`PriorityBlockingQueue` uses a reentrant lock (`ReentrantLock`) for concurrency control. The queue is unbounded (while `ArrayBlockingQueue` is bounded, and `LinkedBlockingQueue` can specify the maximum capacity in the constructor, `PriorityBlockingQueue` can only specify the initial queue size, but **it will automatically expand if space is insufficient** when inserting elements).

In simple terms, it is the thread-safe version of `PriorityQueue`. Null values cannot be inserted, and objects inserted into the queue must be comparable (comparable), otherwise a `ClassCastException` will be thrown. Its insertion operation `put` method will not block because it is an unbounded queue (the `take` method will block when the queue is empty).

**Recommended article:** [Understanding Java Concurrent Queue BlockingQueue](https://javadoop.com/post/java-concurrent-queue)

## ConcurrentSkipListMap

> The following content references the Geeks Time column ["Beauty of Data Structures and Algorithms"](https://time.geekbang.org/column/intro/126?code=zl3GYeAsRI4rEJIBNu5B/km7LSZsPDlGWQEpAYw5Vu0=&utm_term=SPoster "《数据结构与算法之美》") and the book "Practical Java High Concurrency Programming."

To introduce `ConcurrentSkipListMap`, let's first briefly understand the skip list.

For a singly linked list, even if it is sorted, we can only traverse the list from beginning to end to find a specific data element, resulting in very low efficiency. The skip list is different. It is a data structure that allows for fast searches, somewhat similar to a balanced tree. Both can quickly search for elements. However, a significant difference is that inserting or deleting in a balanced tree often requires a global adjustment of the entire tree, whereas inserting or deleting in a skip list only requires local operations. This benefit means that, in high concurrency situations, you would need a global lock to ensure the thread safety of the entire balanced tree. For a skip list, you only need partial locks, resulting in better performance in high concurrency environments. In terms of search performance, the time complexity of a skip list is also **O(log n)**. Therefore, the JDK uses skip lists to implement a Map in concurrent data structures.

A skip list essentially maintains multiple linked lists simultaneously, and these linked lists are layered.

![2-level index skip list](https://oss.javaguide.cn/github/javaguide/java/93666217.jpg)

The lowest level linked list maintains all the elements in the skip list, and each higher-level linked list is a subset of the level below it.

All the elements in each linked list of the skip list are sorted. During a search, you can start from the top-level linked list. Once you find that the element being searched for is greater than the value in the current list, you move down to the next level list to continue the search. This means that the search process is jump-like. As shown in the figure above, searching for element 18 in the skip list.

![Searching for element 18 in the skip list](https://oss.javaguide.cn/github/javaguide/java/32005738.jpg)

When searching for 18, originally requiring 18 traversals, now only requires 7. When dealing with a long linked list, the improvement in search efficiency by building an index becomes very significant.

From the above, it is easy to see that **a skip list is an algorithm that uses space to exchange for time.**

Another difference between using a skip list to implement a Map and using a hash algorithm to implement a Map is that the hash does not preserve the order of elements, whereas all elements in a skip list are sorted. Therefore, when traversing a skip list, you get a sorted result. If your application requires order, then a skip list is your best choice. The class that implements this data structure in the JDK is `ConcurrentSkipListMap`.
