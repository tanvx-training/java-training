### Preface

![ThreadLocal](./images/thread-local/1.png)

**This article contains over 10,000 words and 31 images. It took a considerable amount of time and
effort to create. Original content is not easy, so please show your support by giving it a read.
Thank you.**

When it comes to `ThreadLocal`, your first reaction might be: "It's simple, right? It's a
thread-local variable, each thread has its own copy." But here are a few questions for you to
ponder:

- `ThreadLocal` keys are **weak references**, so when a **GC (Garbage Collection)** occurs after
  calling `ThreadLocal.get()`, is the key **null**?
- What's the **data structure** of `ThreadLocalMap` within `ThreadLocal`?
- What's the **hash algorithm** used in `ThreadLocalMap`?
- How are **hash collisions** resolved in `ThreadLocalMap`?
- What's the **resizing mechanism** of `ThreadLocalMap`?
- How does `ThreadLocalMap` handle **cleaning up expired keys**? What are the processes of **probing
  cleaning** and **heuristic cleaning**?
- What's the implementation principle of the `ThreadLocalMap.set()` method?
- What's the implementation principle of the `ThreadLocalMap.get()` method?
- How is `ThreadLocal` used in projects? What pitfalls have been encountered?
- ...

Have you mastered all of the above questions clearly? This article will delve into the intricacies
of `ThreadLocal` through visual illustrations and explanations.

### Table of Contents

**Note:** The source code in this article is based on `JDK 1.8`.

### Demonstrating `ThreadLocal` with Code

Let's start by looking at an example of using `ThreadLocal`:

```java
public class ThreadLocalTest {

  private List<String> messages = Lists.newArrayList();

  public static final ThreadLocal<ThreadLocalTest> holder = ThreadLocal.withInitial(
      ThreadLocalTest::new);

  public static void add(String message) {
    holder.get().messages.add(message);
  }

  public static List<String> clear() {
    List<String> messages = holder.get().messages;
    holder.remove();

    System.out.println("size: " + holder.get().messages.size());
    return messages;
  }

  public static void main(String[] args) {
    ThreadLocalTest.add("Is a single flower romantic?");
    System.out.println(holder.get().messages);
    ThreadLocalTest.clear();
  }
}
```

The output is as follows:

```java
[Is a single flower romantic?]
    size:0
```

The `ThreadLocal` object provides thread-local variables, where each `Thread` has its own **copy of
the variable**, and multiple threads do not interfere with each other.

### Data Structure of `ThreadLocal`

![](./images/thread-local/2.png)

Each `Thread` object has an instance variable of type `ThreadLocal.ThreadLocalMap`
named `threadLocals`. This means that each thread has its own `ThreadLocalMap`.

`ThreadLocalMap` has its own independent implementation. You can think of its keys as `ThreadLocal`
instances, and the values are the values stored in the code (actually, the key is not `ThreadLocal`
itself, but rather a **weak reference** to it).

When a thread puts a value into `ThreadLocal`, it stores it in its own `ThreadLocalMap`. Reading is
also done using `ThreadLocal` as a reference, searching for the corresponding key in its own map,
thus achieving **thread isolation**.

`ThreadLocalMap` is somewhat similar to the structure of `HashMap`, except that `HashMap` is
implemented using an **array + linked list**, while `ThreadLocalMap` does not have a **linked list**
structure.

We also need to pay attention to `Entry`, whose `key` is `ThreadLocal<?> k`, inherited
from `WeakReference`, which is commonly referred to as a weak reference type.

### Is the Key Null after GC?

In response to the question at the beginning, since the `ThreadLocal` key is a weak reference, is
the key **null** after a `GC` when calling `ThreadLocal.get()`?

To understand this question, we need to understand the **four types of references** in Java:

- **Strong Reference**: Objects created with `new` are typically strong references. As long as the
  strong reference exists, the garbage collector will never collect the referenced object, even when
  memory is low.
- **Soft Reference**: Objects decorated with `SoftReference` are called soft references. Objects
  pointed to by soft references are reclaimed when memory is about to overflow.
- **Weak Reference**: Objects decorated with `WeakReference` are called weak references. Objects
  pointed to by weak references are reclaimed whenever garbage collection occurs, if the object is
  only referenced by weak references.
- **Phantom Reference**: Phantom references are the weakest type of reference, defined in Java
  using `PhantomReference`. The only purpose of phantom references is to receive notifications that
  the object is about to die.

Now, let's take a look at the code. We will use reflection to see the state of the data
in `ThreadLocal` after `GC`. (The code snippet below is sourced
from: <https://blog.csdn.net/thewindkee/article/details/103726942> Run it locally to demonstrate the
GC scenario)

```java
public class ThreadLocalDemo {

  public static void main(String[] args)
      throws NoSuchFieldException, IllegalAccessException, InterruptedException {
    Thread t = new Thread(() -> test("abc", false));
    t.start();
    t.join();
    System.out.println("-- After GC --");
    Thread t2 = new Thread(() -> test("def", true));
    t2.start();
    t2.join();
  }

  private static void test(String s, boolean isGC) {
    try {
      new ThreadLocal<>().set(s);
      if (isGC) {
        System.gc();
      }
      Thread t = Thread.currentThread();
      Class<? extends Thread> clz = t.getClass();
      Field field = clz.getDeclaredField("threadLocals");
      field.setAccessible(true);
      Object ThreadLocalMap = field.get(t);
      Class<?> tlmClass = ThreadLocalMap.getClass();
      Field tableField = tlmClass.getDeclaredField("table");
      tableField.setAccessible(true);
      Object[] arr = (Object[]) tableField.get(ThreadLocalMap);
      for (Object o : arr) {
        if (o != null) {
          Class<?> entryClass = o.getClass();
          Field valueField = entryClass.getDeclaredField("value");
          Field referenceField = entryClass.getSuperclass().getSuperclass()
              .getDeclaredField("referent");
          valueField.setAccessible(true);
          referenceField.setAccessible(true);
          System.out.println(String.format("Weak reference key:%s,value:%s", referenceField.get(o),
              valueField.get(o)));
        }
      }
    } catch (Exception e) {
      e.printStackTrace();
    }
  }
}
```

The output is as follows:

```java
Weak reference key:java.lang.ThreadLocal@433619b6,value:abc
    Weak reference key:java.lang.ThreadLocal@418a15e3,value:java.lang.ref.SoftReference@bf97a12
--After GC--
    Weak reference key:null,value:def
```

As shown in the output, since the `ThreadLocal` created here does not point to any value:

```java
new ThreadLocal<>().set(s);
```

After `GC`, the key will be collected, and we can see from the debugging output that `referent=null`
. If there are no strong references, the key will be collected, which means that the value will not
be collected, resulting in the value persisting forever and causing a memory leak.

However, if we modify the code slightly:

![](./images/thread-local/4.png)

At first glance, if you don't think too deeply about **weak references** and **garbage collection**,
you might assume that the key is `null`.

Actually, this is not correct. Because the question mentioned doing `ThreadLocal.get()` operation,
indicating that there is still a **strong reference** existing. Therefore, the `key` is not `null`,
as shown in the above figure, the **strong reference** of `ThreadLocal` still exists.

If there were no **strong references**, then the `key` would be collected, resulting in the
situation where the `value` is not collected, but the `key` is, leading to the `value` persisting
indefinitely, causing a memory leak.

### Detailed Analysis of the `ThreadLocal.set()` Method

![](./images/thread-local/6.png)

The `set` method in `ThreadLocal` works as depicted in the above diagram. It's quite simple, mainly
checking if the `ThreadLocalMap` exists, and then using the `set` method in `ThreadLocal` to process
the data.

Here's the code:

```java
public void set(T value){
    Thread t=Thread.currentThread();
    ThreadLocalMap map=getMap(t);
    if(map!=null)
    map.set(this,value);
    else
    createMap(t,value);
    }

    void createMap(Thread t,T firstValue){
    t.threadLocals=new ThreadLocalMap(this,firstValue);
    }
```

The core logic lies in `ThreadLocalMap`. Let's delve into it step by step, with more detailed
analysis to follow.

### Hash Algorithm in `ThreadLocalMap`

Since it's a `Map` structure, `ThreadLocalMap` needs to implement its own hash algorithm to resolve
collisions in the hash table array.

```java
int i=key.threadLocalHashCode&(len-1);
```

The hash algorithm in `ThreadLocalMap` is quite simple. Here, `i` represents the current key's array
index position in the hash table.

The most crucial part is the calculation of the `threadLocalHashCode` value. In `ThreadLocal`,
there's a property called `HASH_INCREMENT = 0x61c88647`.

```java
public class ThreadLocal<T> {

  private final int threadLocalHashCode = nextHashCode();

  private static AtomicInteger nextHashCode = new AtomicInteger();

  private static final int HASH_INCREMENT = 0x61c88647;

  private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
  }

  static class ThreadLocalMap {

    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
      table = new Entry[INITIAL_CAPACITY];
      int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

      table[i] = new Entry(firstKey, firstValue);
      size = 1;
      setThreshold(INITIAL_CAPACITY);
    }
  }
}
```

Every time a `ThreadLocal` object is created, the `ThreadLocal.nextHashCode` value increases
by `0x61c88647`.

This value is quite special; it's a **Fibonacci number**, also known as the **golden ratio**. The
hash increment is this number, and the benefit it brings is that the hash distribution is **very
uniform**.

Let's try it ourselves:

![](./images/thread-local/8.png)

As we can see, the distribution of generated hash codes is quite uniform. I won't delve into the
specifics of the Fibonacci algorithm here. If you're interested, you can search for relevant
information.

### `ThreadLocalMap` Hash Conflict

> **Note:** In all the example images below, the **green blocks** represent **valid data** in
> the `Entry`, the **gray blocks** represent `Entry` with a `null` `key`, which **has been garbage
collected**, and the **white blocks** represent `null` `Entry`.

Although `ThreadLocalMap` uses the **golden ratio** as the hashing factor, greatly reducing the
probability of hash conflicts, conflicts can still occur.

In `HashMap`, conflicts are resolved by constructing a **linked list** structure on the array, with
conflicting data attached to the linked list. If the length of the linked list exceeds a certain
threshold, it is converted into a **red-black tree**.

However, `ThreadLocalMap` does not have a linked list structure, so the approach used in `HashMap`
cannot be applied here.

![](./images/thread-local/7.png)

As shown in the diagram above, if we insert data with `value=27`, according to the hash calculation,
it should go into slot 4. However, slot 4 already has `Entry` data.

In this case, a linear search is performed backward until an empty slot with a `null` `Entry` is
found, where the current element is then placed. During iteration, there are other scenarios
encountered, such as encountering an `Entry` that is not `null` and has an equal `key` value, or
encountering an `Entry` with a `null` `key`, each requiring different handling, which will be
explained in detail later.

Additionally, there is an `Entry` with a `null` `key` (represented by the **gray block data**
in `Entry=2`). This is because the `key` value is of **weak reference** type, so such data may
exist. During the `set` process, if an `Entry` with an expired `key` is encountered, a round of **
probing cleanup** operation is actually performed, with the specific operation explained later.

### Detailed Explanation of `ThreadLocalMap.set()`

#### Principle Illustration of `ThreadLocalMap.set()`

After understanding the hash algorithm used in `ThreadLocal`, let's now dive into how `set` works.

**First Scenario:** The slot corresponding to the calculated hash is empty:

![](./images/thread-local/9.png)

In this case, the data is simply placed into the slot.

**Second Scenario:** The slot is not empty, and the `key` value matches the `ThreadLocal`'s `key`
obtained through hash calculation:

![](./images/thread-local/10.png)

In this case, the data in the slot is updated directly.

**Third Scenario:** The slot is not empty, and during the linear search, no expired `Entry` is
encountered before finding a `null` `Entry` slot:

![](./images/thread-local/11.png)

The array is traversed linearly, searching backward. If a `null` `Entry` slot is found, the data is
placed into that slot. Alternatively, if during the traversal, an `Entry` with an equal `key` value
is encountered, it is updated directly.

**Fourth Scenario:** The slot is not empty, and during the linear search, an expired `Entry` is
encountered before finding a `null` `Entry` slot. For example, during the traversal process,
the `Entry` at index 7 has a `null` `key`:

![](./images/thread-local/12.png)

The `replaceStaleEntry()` method is executed, which signifies the logic to replace expired data.
Starting from index 7, a probing cleanup process is initiated.

The start position for scanning expired data is initialized as
follows: `slotToExpunge = staleSlot = 7`

Iterating backward from the current `staleSlot`, other expired data is searched for, updating the
starting scan index `slotToExpunge`. The `for` loop continues until encountering a `null` `Entry`.

If expired data is found, the iteration continues backward until encountering a `null` `Entry` slot,
at which point the iteration stops. In the example shown in the diagram, `slotToExpunge` is updated
to 0.

The backward iteration is performed to update the starting index `slotToExpunge` for probing cleanup
of expired data. This value will be explained later and is used to determine if there are any
expired elements before the current expired slot `staleSlot`.

Next, starting from the `staleSlot` position (index 7), if an `Entry` with the same key value is
found while iterating forward:

![](./images/thread-local/14.png)

From the current `staleSlot`, search for `Entry` elements with equal key values. Once found, update
the `Entry` value, swap the positions of the `staleSlot` element (which is expired), update
the `Entry` data, and then begin cleaning up the expired `Entry`, as shown in the diagram below:

![](https://oss.javaguide.cn/java-guide-blog/view.png)During the forward traversal, if no `Entry`
with the same key value is found:

![](./images/thread-local/15.png)

From the current `staleSlot`, search forward for `Entry` elements with equal key values until
encountering a `null` `Entry`, at which point the search stops. As illustrated in the diagram, there
are no `Entry` elements with the same key value at this time.

Create a new `Entry` and replace `table[staleSlot]`:

![](./images/thread-local/16.png)

After the replacement, expired element cleanup is performed. The cleanup mainly involves two
methods: `expungeStaleEntry()` and `cleanSomeSlots()`. Further details will be explained later on.

### Detailed Explanation of `ThreadLocalMap.set()` Method

After providing an illustrated explanation of the `set()` method, let's delve into the source code:

`java.lang.ThreadLocal.ThreadLocalMap.set()`:

```java
private void set(ThreadLocal<?> key,Object value){
    Entry[]tab=table;
    int len=tab.length;
    int i=key.threadLocalHashCode&(len-1);

    for(Entry e=tab[i];
    e!=null;
    e=tab[i=nextIndex(i,len)]){
    ThreadLocal<?> k=e.get();

    if(k==key){
    e.value=value;
    return;
    }

    if(k==null){
    replaceStaleEntry(key,value,i);
    return;
    }
    }

    tab[i]=new Entry(key,value);
    int sz=++size;
    if(!cleanSomeSlots(i,sz)&&sz>=threshold)
    rehash();
    }
```

In this method, the position in the hash table corresponding to the key is calculated, and then the
bucket corresponding to the current key is searched backward to find an available bucket.

```java
Entry[]tab=table;
    int len=tab.length;
    int i=key.threadLocalHashCode&(len-1);
```

Under what circumstances is a bucket considered available?

1. If `k = key`, it indicates a replacement operation and is considered available.
2. If `key = null`, it indicates that the `Entry` at the bucket position is expired.
   The `replaceStaleEntry()` method (core method) is executed, and then the operation returns.
3. If during the loop, an `Entry` with a `null` `key` is encountered, it means that the bucket is
   available for use.

Next, the method proceeds with a `for` loop traversal, searching backward. Let's take a look at
the `nextIndex()` and `prevIndex()` methods:

```java
private static int nextIndex(int i,int len){
    return((i+1<len)?i+1:0);
    }

private static int prevIndex(int i,int len){
    return((i-1>=0)?i-1:len-1);
    }
```

Now, let's examine the remaining logic within the `for` loop:

1. If during the traversal, the `Entry` at the position corresponding to the key value is null, it
   means there is no data conflict in the hash table array. The loop exits, and the data is set
   directly into the corresponding bucket.
2. If the `Entry` at the position corresponding to the key value is not null:  
   2.1 If `k = key`, it indicates a replacement operation. The data is updated, and the loop
   returns.  
   2.2 If `key = null`, it indicates that the `Entry` at the bucket position is expired.
   The `replaceStaleEntry()` method (core method) is executed, and then the operation returns.
3. If the `for` loop completes without finding an `Entry` with a `null` key, it means a null `Entry`
   has been encountered during the backward traversal.  
   3.1 A new `Entry` object is created in the bucket where the `Entry` is null.  
   3.2 `++size` operation is performed.
4. The `cleanSomeSlots()` method is called to perform heuristic cleanup of expired
   key-related `Entry` data.  
   4.1 If no data is cleaned up and `size` exceeds the threshold (2/3 of array length), a `rehash()`
   operation is performed.  
   4.2 Within `rehash()`, a probing cleanup is first performed to clear expired keys. After cleanup,
   if **size >= threshold - threshold / 4**, the actual resizing logic is executed (resizing logic
   will be explained later).

Now, let's focus on the `replaceStaleEntry()` method, which provides functionality to replace
expired data. We can revisit the principle diagram of the **Fourth Scenario** and correlate it with
the following code:

`java.lang.ThreadLocal.ThreadLocalMap.replaceStaleEntry()`:

```java
private void replaceStaleEntry(ThreadLocal<?> key,Object value,
    int staleSlot){
    Entry[]tab=table;
    int len=tab.length;
    Entry e;

    int slotToExpunge=staleSlot;
    for(int i=prevIndex(staleSlot,len);
    (e=tab[i])!=null;
    i=prevIndex(i,len))

    if(e.get()==null)
    slotToExpunge=i;

    for(int i=nextIndex(staleSlot,len);
    (e=tab[i])!=null;
    i=nextIndex(i,len)){

    ThreadLocal<?> k=e.get();

    if(k==key){
    e.value=value;

    tab[i]=tab[staleSlot];
    tab[staleSlot]=e;

    if(slotToExpunge==staleSlot)
    slotToExpunge=i;
    cleanSomeSlots(expungeStaleEntry(slotToExpunge),len);
    return;
    }

    if(k==null&&slotToExpunge==staleSlot)
    slotToExpunge=i;
    }

    tab[staleSlot].value=null;
    tab[staleSlot]=new Entry(key,value);

    if(slotToExpunge!=staleSlot)
    cleanSomeSlots(expungeStaleEntry(slotToExpunge),len);
    }
```

The `slotToExpunge` variable represents the starting index for probing cleanup of expired data,
initially set to the current `staleSlot`. The loop starts from `staleSlot` and iterates backward to
find non-expired data. The loop continues until encountering a null `Entry`, at which
point `slotToExpunge` is updated to this index.

```java
for(int i=prevIndex(staleSlot,len);
    (e=tab[i])!=null;
    i=prevIndex(i,len)){

    if(e.get()==null){
    slotToExpunge=i;
    }
    }
```

Then, the method proceeds with a forward search from `staleSlot`, also ending when encountering a
null `Entry`. If, during the iteration, `k == key` is found, it indicates a replacement logic. The
data is updated, and the current `staleSlot` position is swapped. If `slotToExpunge == staleSlot`,
it indicates that no expired `Entry` was found during the initial backward search. In this case, the
start index for probing cleanup of expired data is updated to the current loop index,
i.e., `slotToExpunge = i`. Finally, the `cleanSomeSlots(expungeStaleEntry(slotToExpunge), len);`
method is called for heuristic cleanup of expired data.

```java
if(k==key){
    e.value=value;

    tab[i]=tab[staleSlot];
    tab[staleSlot]=e;

    if(slotToExpunge==staleSlot)
    slotToExpunge=i;

    cleanSomeSlots(expungeStaleEntry(slotToExpunge),len);
```

### Detailed Explanation of `ThreadLocalMap`'s Probe Cleanup Process for Expired Keys

Earlier, we mentioned two methods for cleaning up expired key data in `ThreadLocalMap`: **probe
cleanup** and **heuristic cleanup**.

Let's first discuss probe cleanup, which involves the `expungeStaleEntry` method. This method
traverses the hash table array, probing forward from the starting position to clean up expired data.
It sets the `Entry` of expired data to `null`. Along the way, if it encounters unexpired data, it
rehashes the data and repositions it in the `table` array. If the repositioned location already
contains data, it moves the unexpired data to the nearest `Entry=null` bucket, making the `Entry`
data closer to its correct bucket after rehashing. Here's how it works:

![](./images/thread-local/18.png)

In the diagram above, let's assume `expungeStaleEntry(3)` is called. We can see the state of
the `table` data in the `ThreadLocalMap`. Then, the cleanup operation is executed:

![](./images/thread-local/21.png)

The first step is to clear the data at the current `staleSlot` position, making the `Entry`
at `index=3` null. Then, it continues to probe forward:

![](./images/thread-local/22.png)

After the second step, the element at `index=4` is moved to the `index=3` slot.

Continuing the iteration, it encounters normal data. It calculates whether the position of this data
has shifted. If it has shifted, it recalculates the slot position to ensure that normal data is
placed as close as possible to its correct position or a position closer to it.

![](./images/thread-local/23.png)

During the iteration forward, if it encounters an empty slot, the probing stops. This completes one
round of probe cleanup. Now, let's look at the implementation source code:

```java
private int expungeStaleEntry(int staleSlot){
    Entry[]tab=table;
    int len=tab.length;

    tab[staleSlot].value=null;
    tab[staleSlot]=null;
    size--;

    Entry e;
    int i;
    for(i=nextIndex(staleSlot,len);
    (e=tab[i])!=null;
    i=nextIndex(i,len)){
    ThreadLocal<?> k=e.get();
    if(k==null){
    e.value=null;
    tab[i]=null;
    size--;
    }else{
    int h=k.threadLocalHashCode&(len-1);
    if(h!=i){
    tab[i]=null;

    while(tab[h]!=null)
    h=nextIndex(h,len);
    tab[h]=e;
    }
    }
    }
    return i;
    }
```

Let's continue using `staleSlot=3` as an example. First, it clears the data at `tab[staleSlot]`,
making it null, and then decrements `size`.

```java
ThreadLocal<?> k=e.get();

    if(k==null){
    e.value=null;
    tab[i]=null;
    size--;
    }
```

Next, it iterates forward from `staleSlot`. If it encounters expired data where `k == null`, it
clears the data at that slot, decrements `size`, and continues.

```java
int h=k.threadLocalHashCode&(len-1);
    if(h!=i){
    tab[i]=null;

    while(tab[h]!=null)
    h=nextIndex(h,len);

    tab[h]=e;
    }
```

If the key has not expired, it recalculates the index position for the key. If it's different from
the current index position (`h != i`), it means a hash collision has occurred. In this case, it sets
the current `tab[i]` to null and finds the nearest position to store the `Entry` data after
rehashing. After this iteration, `Entry` positions with hash collisions are closer to their correct
positions, which improves query efficiency.

### `ThreadLocalMap` Resizing Mechanism

In the `ThreadLocalMap.set()` method, after completing heuristic cleaning and if no data was
cleaned, and if the number of `Entry` elements in the current hash table has reached the resizing
threshold `(len*2/3)`, the `rehash()` logic is triggered:

```java
if(!cleanSomeSlots(i,sz)&&sz>=threshold)
    rehash();
```

Let's delve into the specific implementation of `rehash()`:

```java
private void rehash(){
    expungeStaleEntries();

    if(size>=threshold-threshold/4)
    resize();
    }

private void expungeStaleEntries(){
    Entry[]tab=table;
    int len=tab.length;
    for(int j=0;j<len; j++){
    Entry e=tab[j];
    if(e!=null&&e.get()==null)
    expungeStaleEntry(j);
    }
    }
```

Firstly, it performs heuristic cleaning from the beginning of the `table`. The detailed cleaning
process is analyzed above. After cleaning, there might be some `Entry` data with `null` keys in
the `table`. Hence, by checking if `size >= threshold - threshold / 4`, which is equivalent
to `size >= threshold * 3/4`, the decision to resize is made.

Remember that the threshold for triggering `rehash()` is `size >= threshold`. So, when discussing
the resizing mechanism of `ThreadLocalMap`, it's essential to explain these two steps clearly:

![](./images/thread-local/24.png)

Now let's examine the specific implementation of the `resize()` method, using `oldTab.len=8` as an
example:

![](./images/thread-local/25.png)

After resizing, the size of `tab` becomes `oldLen * 2`. Then, it iterates through the old hash
table, recalculates the hash positions, and places the entries in the new `tab` array. If a hash
conflict occurs, it searches for the nearest slot with a `null` entry. Once the iteration is
complete, all the `entry` data from `oldTab` has been transferred to the new `tab`. It recalculates
the threshold for the next resizing. The specific code is as follows:

```java
private void resize(){
    Entry[]oldTab=table;
    int oldLen=oldTab.length;
    int newLen=oldLen*2;
    Entry[]newTab=new Entry[newLen];
    int count=0;

    for(int j=0;j<oldLen; ++j){
    Entry e=oldTab[j];
    if(e!=null){
    ThreadLocal<?> k=e.get();
    if(k==null){
    e.value=null;
    }else{
    int h=k.threadLocalHashCode&(newLen-1);
    while(newTab[h]!=null)
    h=nextIndex(h,newLen);
    newTab[h]=e;
    count++;
    }
    }
    }

    setThreshold(newLen);
    size=count;
    table=newTab;
    }
```

### Understanding `ThreadLocalMap.get()`

We've already examined the source code for the `set()` method, including operations like setting
data, cleaning data, and optimizing the positions of data buckets. Now, let's explore the principles
behind the `get()` operation.

#### Graphical Explanation of `ThreadLocalMap.get()`

**First Scenario:** The `ThreadLocal` key's hash calculation leads directly to a slot in the hash
table, and the key in that slot matches the one being searched for. In this case, it returns the
corresponding entry.

![](./images/thread-local/26.png)

**Second Scenario:** The key in the slot does not match the one being searched for.

![](./images/thread-local/27.png)

For example, let's consider `get(ThreadLocal1)`. After calculating the hash, the correct slot
position should be 4. However, the slot at `index=4` already contains data, and its key does not
match `ThreadLocal1`. Thus, it needs to continue iterating.

When it reaches the data at `index=5`, the key is `null`, triggering a single round of heuristic
data recovery by executing the `expungeStaleEntry()` method. After this, the data at `index 5`
and `index 8` will be recovered, while the data at `index 6` and `index 7` will move forward.
Continuing the iteration from `index=5`, it finds the `Entry` with a matching key at `index=6`, as
shown below:

![](./images/thread-local/28.png)

#### Detailed Explanation of `ThreadLocalMap.get()` Source Code

`java.lang.ThreadLocal.ThreadLocalMap.getEntry()`:

```java
private Entry getEntry(ThreadLocal<?> key){
    int i=key.threadLocalHashCode&(table.length-1);
    Entry e=table[i];
    if(e!=null&&e.get()==key)
    return e;
    else
    return getEntryAfterMiss(key,i,e);
    }

private Entry getEntryAfterMiss(ThreadLocal<?> key,int i,Entry e){
    Entry[]tab=table;
    int len=tab.length;

    while(e!=null){
    ThreadLocal<?> k=e.get();
    if(k==key)
    return e;
    if(k==null)
    expungeStaleEntry(i);
    else
    i=nextIndex(i,len);
    e=tab[i];
    }
    return null;
    }
```

### Heuristic Cleanup Process for Expired Keys in `ThreadLocalMap`

We've mentioned two methods for cleaning up expired keys in `ThreadLocalMap`: **Probe-based Cleanup (expungeStaleEntry())** and **Heuristic Cleanup (cleanSomeSlots())**.

Probe-based cleanup involves linearly traversing the entries from the current entry onwards until a null value is encountered, thus cleaning the entries.

Heuristic cleanup, as defined by the author, involves "Heuristically scan some cells looking for stale entries".

Here's the specific code for heuristic cleanup:

```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ((n >>>= 1) != 0);
    return removed;
}
```

### `InheritableThreadLocal`

When using `ThreadLocal`, it's not possible to share the thread-local data created in the parent thread with child threads, especially in asynchronous scenarios.

To address this issue, the JDK provides the `InheritableThreadLocal` class. Here's an example:

```java
public class InheritableThreadLocalDemo {

    public static void main(String[] args) {
        ThreadLocal<String> threadLocal = new ThreadLocal<>();
        ThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();
        threadLocal.set("Parent ThreadLocal Data");
        inheritableThreadLocal.set("Parent InheritableThreadLocal Data");

        new Thread(new Runnable() {
            @Override
            public void run() {
                System.out.println("Child thread accesses parent ThreadLocal data: " + threadLocal.get());
                System.out.println("Child thread accesses parent InheritableThreadLocal data: " + inheritableThreadLocal.get());
            }
        }).start();
    }
}
```

Printed result:

```java
Child thread accesses parent ThreadLocal data: null
Child thread accesses parent InheritableThreadLocal data: Parent InheritableThreadLocal Data
```

The implementation principle involves copying data from the parent thread to the child thread. This copying occurs in the `init()` method of the `Thread` class, called when a new thread is created using `new Thread()`:

```java
private void init(ThreadGroup g, Runnable target, String name, long stackSize, AccessControlContext acc, boolean inheritThreadLocals) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }

    if (inheritThreadLocals && parent.inheritableThreadLocals != null) {
        this.inheritableThreadLocals = ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    }
    this.stackSize = stackSize;
    tid = nextThreadID();
}
```

However, `InheritableThreadLocal` has limitations, particularly when dealing with thread pools where threads are reused. In such cases, `InheritableThreadLocal` may not behave as expected.

To address this issue, Alibaba has open-sourced a component called `TransmittableThreadLocal`, which resolves the problems associated with `InheritableThreadLocal`. Further details on this can be explored independently if interested.

### Practical Use of `ThreadLocal` in Projects

#### Scenarios for Using `ThreadLocal`

In our project, we use ELK (Elasticsearch, Logstash, and Kibana) for logging. Logs are sent to Logstash and then visualized and queried in Kibana.

As we're dealing with distributed systems where services communicate with each other, we need a way to pass a `traceId` between different services to correlate requests. We achieve this using `org.slf4j.MDC`, which internally utilizes `ThreadLocal`. Here's how it works:

When a request is received by **Service A**, it generates a `traceId` (similar to a UUID) and stores it in the current thread's `ThreadLocal`. When **Service A** calls **Service B**, it includes the `traceId` in the request's header. Upon receiving the request, **Service B** checks if the `traceId` is present in the header. If it is, it stores it in its own thread's `ThreadLocal`.

![](./images/thread-local/30.png)

The `requestId` shown in the diagram is the `traceId` used to correlate different service calls. With this `requestId`, we can trace the entire chain of service invocations. Additionally, there are other scenarios to consider:

![](./images/thread-local/31.png)

For these scenarios, we have corresponding solutions, as outlined below.

#### Solution for Feign Remote Calls

**Service Making the Request:**

```java
@Component
@Slf4j
public class FeignInvokeInterceptor implements RequestInterceptor {

  @Override
  public void apply(RequestTemplate template) {
    String requestId = MDC.get("requestId");
    if (StringUtils.isNotBlank(requestId)) {
      template.header("requestId", requestId);
    }
  }
}
```

**Service Receiving the Request:**

```java
@Slf4j
@Component
public class LogInterceptor extends HandlerInterceptorAdapter {

  @Override
  public void afterCompletion(HttpServletRequest arg0, HttpServletResponse arg1, Object arg2,
      Exception arg3) {
    MDC.remove("requestId");
  }

  @Override
  public void postHandle(HttpServletRequest arg0, HttpServletResponse arg1, Object arg2,
      ModelAndView arg3) {
  }

  @Override
  public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler)
      throws Exception {

    String requestId = request.getHeader(BaseConstant.REQUEST_ID_KEY);
    if (StringUtils.isBlank(requestId)) {
      requestId = UUID.randomUUID().toString().replace("-", "");
    }
    MDC.put("requestId", requestId);
    return true;
  }
}
```

#### Passing `requestId` in Asynchronous Thread Pool Calls

Because `MDC` relies on `ThreadLocal`, child threads in asynchronous processes cannot access data stored in the parent thread's `ThreadLocal`. To address this, we can customize the thread pool executor and modify its `run()` method:

```java
public class MyThreadPoolTaskExecutor extends ThreadPoolTaskExecutor {

  @Override
  public void execute(Runnable runnable) {
    Map<String, String> context = MDC.getCopyOfContextMap();
    super.execute(() -> run(runnable, context));
  }

  @Override
  private void run(Runnable runnable, Map<String, String> context) {
    if (context != null) {
      MDC.setContextMap(context);
    }
    try {
      runnable.run();
    } finally {
      MDC.remove();
    }
  }
}
```
