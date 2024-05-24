## Introduction to Atomic Classes

The word "Atomic" translates to "原子" in Chinese, meaning the smallest unit of matter in chemistry, which is indivisible in a chemical reaction. In our context, "Atomic" means an operation that cannot be interrupted. Even when multiple threads execute together, once an operation starts, it will not be interrupted by other threads.

So, in simple terms, an atomic class is a class that has atomic operation characteristics.

The atomic classes in the concurrent package `java.util.concurrent` are all located in `java.util.concurrent.atomic`, as shown below.

![Overview of JUC Atomic Classes](https://oss.javaguide.cn/github/javaguide/java/JUC%E5%8E%9F%E5%AD%90%E7%B1%BB%E6%A6%82%E8%A7%88.png)

Based on the type of data being operated on, the atomic classes in the JUC package can be divided into four categories:

### Basic Types

Using atomic methods to update basic types:

- `AtomicInteger`: Atomic class for integers
- `AtomicLong`: Atomic class for long integers
- `AtomicBoolean`: Atomic class for booleans

### Array Types

Using atomic methods to update elements in an array:

- `AtomicIntegerArray`: Atomic class for integer arrays
- `AtomicLongArray`: Atomic class for long integer arrays
- `AtomicReferenceArray`: Atomic class for reference type arrays

### Reference Types

- `AtomicReference`: Atomic class for reference types
- `AtomicMarkableReference`: Atomic class for updating reference types with a mark. This class associates a boolean mark with the reference, ~~which can also address the ABA problem that may occur when using CAS for atomic updates~~.
- `AtomicStampedReference`: Atomic class for updating reference types with a version number. This class associates an integer value with the reference, which can be used to solve the problem of atomic updates of data and data versions, effectively addressing the ABA problem when using CAS for atomic updates.

**🐛 Correction (refer to: [issue#626](https://github.com/Snailclimb/JavaGuide/issues/626))**: `AtomicMarkableReference` cannot solve the ABA problem.

### Object Property Update Types

- `AtomicIntegerFieldUpdater`: Updater for atomic updates of integer fields
- `AtomicLongFieldUpdater`: Updater for atomic updates of long integer fields
- `AtomicReferenceFieldUpdater`: Updater for atomic updates of fields with reference types

## Atomic Classes for Basic Types

Atomic classes for updating basic types:

- `AtomicInteger`: Atomic class for integers
- `AtomicLong`: Atomic class for long integers
- `AtomicBoolean`: Atomic class for booleans

The methods provided by these three classes are almost the same, so we will use `AtomicInteger` as an example to introduce them.

### Common Methods of the `AtomicInteger` Class

```java
public final int get() // Get the current value
public final int getAndSet(int newValue) // Get the current value and set a new value
public final int getAndIncrement() // Get the current value and increment
public final int getAndDecrement() // Get the current value and decrement
public final int getAndAdd(int delta) // Get the current value and add the expected value
boolean compareAndSet(int expect, int update) // Atomically set the value to the given updated value if the current value equals the expected value
public final void lazySet(int newValue) // Eventually sets to the newValue, using lazySet may cause other threads to read the old value for a short time afterwards
```

### Example Usage of the `AtomicInteger` Class

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicIntegerTest {

    public static void main(String[] args) {
        int tempValue = 0;
        AtomicInteger i = new AtomicInteger(0);
        tempValue = i.getAndSet(3);
        System.out.println("tempValue: " + tempValue + ";  i: " + i); // tempValue: 0;  i: 3
        tempValue = i.getAndIncrement();
        System.out.println("tempValue: " + tempValue + ";  i: " + i); // tempValue: 3;  i: 4
        tempValue = i.getAndAdd(5);
        System.out.println("tempValue: " + tempValue + ";  i: " + i); // tempValue: 4;  i: 9
    }

}
```

### Advantages of Atomic Classes for Basic Types

Let's look at a simple example to understand the advantages of atomic classes for basic data types.

**1. Ensuring thread safety in a multi-threaded environment without using atomic classes (basic data types)**

```java
class Test {
    private volatile int count = 0;
    // To ensure thread safety when performing count++, locking is needed
    public synchronized void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}
```

**2. Ensuring thread safety in a multi-threaded environment using atomic classes (basic data types)**

```java
class Test2 {
    private AtomicInteger count = new AtomicInteger();

    public void increment() {
        count.incrementAndGet();
    }
    // Using AtomicInteger, no need for locks, and thread safety is achieved.
    public int getCount() {
        return count.get();
    }
}
```

### Simple Analysis of AtomicInteger Thread-Safety Principle

Part of the source code of the `AtomicInteger` class:

```java
// setup to use Unsafe.compareAndSwapInt for updates
private static final Unsafe unsafe = Unsafe.getUnsafe();
private static final long valueOffset;

static {
    try {
        valueOffset = unsafe.objectFieldOffset
            (AtomicInteger.class.getDeclaredField("value"));
    } catch (Exception ex) {
        throw new Error(ex);
    }
}

private volatile int value;
```

The `AtomicInteger` class mainly utilizes CAS (compare and swap) + volatile and native methods to ensure atomic operations, thereby avoiding the high overhead of `synchronized` and significantly improving execution efficiency.

The principle of CAS is to compare the expected value with the original value; if they are the same, the original value is updated to the new value. The `Unsafe` class's `objectFieldOffset()` method is a native method used to get the memory address of the "original value." Additionally, value is a volatile variable, making it visible in memory, so the JVM can ensure that any thread can always access the latest value of this variable at any given time.

## Array Type Atomic Classes

Updating elements in an array atomically is facilitated by:

- `AtomicIntegerArray`: Atomic integer array class
- `AtomicLongArray`: Atomic long integer array class
- `AtomicReferenceArray`: Atomic reference type array class

Since the methods provided by these three classes are almost identical, let's use `AtomicIntegerArray` as an example to introduce them.

**Common Methods in the `AtomicIntegerArray` Class**:

```java
public final int get(int i) // Get the value at index = i
public final int getAndSet(int i, int newValue) // Return the current value at index = i and set it to newValue
public final int getAndIncrement(int i) // Get the value at index = i and increment it
public final int getAndDecrement(int i) // Get the value at index = i and decrement it
public final int getAndAdd(int i, int delta) // Get the value at index = i and add the expected value
boolean compareAndSet(int i, int expect, int update) // If the input value equals the expected value, atomically set the value at index = i to the input value (update)
public final void lazySet(int i, int newValue) // Set the value at index = i to newValue; using lazySet may allow other threads to still read the old value for a short time afterwards.
```

**Example of Using the `AtomicIntegerArray` Class**:

```java
import java.util.concurrent.atomic.AtomicIntegerArray;

public class AtomicIntegerArrayTest {

  public static void main(String[] args) {
    int temvalue = 0;
    int[] nums = { 1, 2, 3, 4, 5, 6 };
    AtomicIntegerArray i = new AtomicIntegerArray(nums);
    for (int j = 0; j < nums.length; j++) {
      System.out.println(i.get(j));
    }
    temvalue = i.getAndSet(0, 2);
    System.out.println("temvalue:" + temvalue + ";  i:" + i);
    temvalue = i.getAndIncrement(0);
    System.out.println("temvalue:" + temvalue + ";  i:" + i);
    temvalue = i.getAndAdd(0, 5);
    System.out.println("temvalue:" + temvalue + ";  i:" + i);
  }

}
```

## Reference Type Atomic Classes

Basic type atomic classes can only update a single variable. If atomic updates for multiple variables are needed, reference type atomic classes are used.

- `AtomicReference`: Atomic reference type class
- `AtomicStampedReference`: Atomic reference type class with version number. This class associates an integer value with a reference, useful for resolving atomic updates of data and data version numbers, thus addressing potential ABA problems that may occur when using CAS for atomic updates.
- `AtomicMarkableReference`: Atomic reference type class with mark. This class associates a boolean mark with a reference. ~~It can also solve potential ABA problems that may occur when using CAS for atomic updates.~~

Since the methods provided by these three classes are almost identical, let's use `AtomicReference` as an example to introduce them.

**Example of Using the `AtomicReference` Class** :

```java
import java.util.concurrent.atomic.AtomicReference;

public class AtomicReferenceTest {

    public static void main(String[] args) {
        AtomicReference<Person> ar = new AtomicReference<>();
        Person person = new Person("SnailClimb", 22);
        ar.set(person);
        Person updatePerson = new Person("Daisy", 20);
        ar.compareAndSet(person, updatePerson);

        System.out.println(ar.get().getName());
        System.out.println(ar.get().getAge());
    }
}

class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        super();
        this.name = name;
        this.age = age;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public int getAge() {
        return age;
    }

    public void setAge(int age) {
        this.age = age;
    }

}
```

In the above code, a `Person` object is created first, then the `Person` object is set into the `AtomicReference` object, and finally the `compareAndSet` method is called. This method sets `ar` through a CAS operation. If the value of `ar` is `person`, it will be set to `updatePerson`. The underlying principle is the same as the `compareAndSet` method in the `AtomicInteger` class. After running the above code, the output is as follows:

```plaintext
Daisy
20
```

**Usage Example of `AtomicStampedReference` Class**:

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class AtomicStampedReferenceDemo {
    public static void main(String[] args) {
        // Instantiate, get current value, and get stamp value
        final Integer initialRef = 0, initialStamp = 0;
        final AtomicStampedReference<Integer> asr = new AtomicStampedReference<>(initialRef, initialStamp);
        System.out.println("currentValue=" + asr.getReference() + ", currentStamp=" + asr.getStamp());

        // Compare and set
        final Integer newReference = 666, newStamp = 999;
        final boolean casResult = asr.compareAndSet(initialRef, newReference, initialStamp, newStamp);
        System.out.println("currentValue=" + asr.getReference()
                + ", currentStamp=" + asr.getStamp()
                + ", casResult=" + casResult);

        // Get current value and current stamp value
        int[] arr = new int[1];
        final Integer currentValue = asr.get(arr);
        final int currentStamp = arr[0];
        System.out.println("currentValue=" + currentValue + ", currentStamp=" + currentStamp);

        // Set stamp value alone
        final boolean attemptStampResult = asr.attemptStamp(newReference, 88);
        System.out.println("currentValue=" + asr.getReference()
                + ", currentStamp=" + asr.getStamp()
                + ", attemptStampResult=" + attemptStampResult);

        // Reset current value and stamp value
        asr.set(initialRef, initialStamp);
        System.out.println("currentValue=" + asr.getReference() + ", currentStamp=" + asr.getStamp());

        // [Not recommended for use unless you understand the meaning of the comments]
        // Weak compare and set
        // Confusing! The weakCompareAndSet method ultimately calls the compareAndSet method. [Version: jdk-8u191]
        // But the comment says "May fail spuriously and does not provide ordering guarantees,
        // so is only rarely an appropriate alternative to compareAndSet."
        // todo It seems possible that the JVM forwards the method in the native method through the method name
        final boolean wCasResult = asr.weakCompareAndSet(initialRef, newReference, initialStamp, newStamp);
        System.out.println("currentValue=" + asr.getReference()
                + ", currentStamp=" + asr.getStamp()
                + ", wCasResult=" + wCasResult);
    }
}
```

The output result is as follows:

```plaintext
currentValue=0, currentStamp=0
currentValue=666, currentStamp=999, casResult=true
currentValue=666, currentStamp=999
currentValue=666, currentStamp=88, attemptStampResult=true
currentValue=0, currentStamp=0
currentValue=666, currentStamp=999, wCasResult=true
```

**Usage Example of `AtomicMarkableReference` Class**:

```java
import java.util.concurrent.atomic.AtomicMarkableReference;

public class AtomicMarkableReferenceDemo {
    public static void main(String[] args) {
        // Instantiate, get current value, and get mark value
        final Boolean initialRef = null, initialMark = false;
        final AtomicMarkableReference<Boolean> amr = new AtomicMarkableReference<>(initialRef, initialMark);
        System.out.println("currentValue=" + amr.getReference() + ", currentMark=" + amr.isMarked());

        // Compare and set
        final Boolean newReference1 = true, newMark1 = true;
        final boolean casResult = amr.compareAndSet(initialRef, newReference1, initialMark, newMark1);
        System.out.println("currentValue=" + amr.getReference()
                + ", currentMark=" + amr.isMarked()
                + ", casResult=" + casResult);

        // Get current value and current mark value
        boolean[] arr = new boolean[1];
        final Boolean currentValue = amr.get(arr);
        final boolean currentMark = arr[0];
        System.out.println("currentValue=" + currentValue + ", currentMark=" + currentMark);

        // Set mark value alone
        final boolean attemptMarkResult = amr.attemptMark(newReference1, false);
        System.out.println("currentValue=" + amr.getReference()
                + ", currentMark=" + amr.isMarked()
                + ", attemptMarkResult=" + attemptMarkResult);

        // Reset current value and mark value
        amr.set(initialRef, initialMark);
        System.out.println("currentValue=" + amr.getReference() + ", currentMark=" + amr.isMarked());

        // [Not recommended for use unless you understand the meaning of the comments]
        // Weak compare and set
        // Confusing! The weakCompareAndSet method ultimately calls the compareAndSet method. [Version: jdk-8u191]
        // But the comment says "May fail spuriously and does not provide ordering guarantees,
        // so is only rarely an appropriate alternative to compareAndSet."
        // todo It seems possible that the JVM forwards the method in the native method through the method name
        final boolean wCasResult = amr.weakCompareAndSet(initialRef, newReference1, initialMark, newMark1);
        System.out.println("currentValue=" + amr.getReference()
                + ", currentMark=" + amr.isMarked()
                + ", wCasResult=" + wCasResult);
    }
}
```

The output result is as follows:

```plaintext
currentValue=null, currentMark=false
currentValue=true, currentMark=true, casResult=true
currentValue=true, currentMark=true
currentValue=true, currentMark=false, attemptMarkResult=true
currentValue=null, currentMark=false
currentValue=true, currentMark=true, wCasResult=true
```

## Object Field Modifier Type Atomic Classes

When you need to atomically update a field within an object, you need to use object field modifier type atomic classes.

- `AtomicIntegerFieldUpdater`: Atomically updates an integer field updater.
- `AtomicLongFieldUpdater`: Atomically updates a long integer field updater.
- `AtomicReferenceFieldUpdater`: Atomically updates a reference type field within an object.

To atomically update an object's property, you need to follow two steps. First, since the object field modifier type atomic classes are abstract classes, you must use the static method `newUpdater()` each time to create an updater, and you need to specify the class and the field you want to update. Second, the field of the object being updated must be marked with the `public volatile` modifier.

The methods provided by the above three classes are almost identical, so here we use `AtomicIntegerFieldUpdater` as an example to illustrate.

**Usage Example of `AtomicIntegerFieldUpdater` Class**:

```java
import java.util.concurrent.atomic.AtomicIntegerFieldUpdater;

public class AtomicIntegerFieldUpdaterTest {
  public static void main(String[] args) {
    AtomicIntegerFieldUpdater<User> a = AtomicIntegerFieldUpdater.newUpdater(User.class, "age");

    User user = new User("Java", 22);
    System.out.println(a.getAndIncrement(user));// 22
    System.out.println(a.get(user));// 23
  }
}

class User {
  private String name;
  public volatile int age;

  public User(String name, int age) {
    super();
    this.name = name;
    this.age = age;
  }

  public String getName() {
    return name;
  }

  public void setName(String name) {
    this.name = name;
  }

  public int getAge() {
    return age;
  }

  public void setAge(int age) {
    this.age = age;
  }

}
```

Output:

```plaintext
22
23
```

