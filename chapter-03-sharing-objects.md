# Chapter 3: Sharing Objects
* The Last chapter was about how we can avoid thread from accessing shared object via synchronized blocks, in this chapter we will study techniques for sharing and publishing objects so that they can be accessed by multiple threads. 
* Also we will make sure that `memory visibility` is also there with along with sync on critical blocks. 



### Visibility 
```java
public class NoVisibility {

    private static boolean ready;
    private static int number;

    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready) {
                Thread.yield();
            }
            System.out.println(number);
        }
    }

    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```
* Above class might look simple & correct, but in real it's not. 
* This code does not guarantee visibility and ordering. As ReaderThread can keep on reading from cache (register's cache) and never see updated results. 
* Even if it see updated results, it's possible that it sees `ready = true` before `number = 42`. So this code lacks both ordering and visibility. 

#### Ways to solve this problem
**1. synchronized keyword (intrinsic lock)**
```java
public class NoVisibility {

    private static boolean ready;
    private static int number;
    private static final Object lock = new Object(); // common lock for proper synchronization

    private static class ReaderThread extends Thread {
        public void run() {
            while (true) {
                synchronized (lock) {
                    if (ready) {
                        System.out.println(number);
                        break;
                    }
                }
                Thread.yield();
            }
        }
    }

    public static void main(String[] args) {
        new ReaderThread().start();
        synchronized (lock) {
            number = 42;
            ready = true;
        }
    }
}
```

**2. Volatile Variable** 
* When a field is set to volatile, the compiler and runtime are notified that these variable operations should not be reordered with other memory operations. 
* They no longer cached on register, always most recently written by any thread will be returned for these variables. 
* Book suggested to not use volatile over locks where visibility is required for reasoning. 
* The problem with volatile is that even though it guranttees no operation reordering and visibility, but it do not guarantee for a atomic change on variable when multiple threads are changing it. 
* The most common usecases for volatile are: status flags, interruption, shutdown checks, etc. 
> Locking -> Visibility + Atomicity
> 
> Volatile -> Visibility only

### Publication and Escape 
* **Publication**: Making object available to other parts of code, either by retuning from method or passing in parameters. 
* **Escape**: When an object is published when it shouldn't be, like accidentally making a private variable accessible to the outside world 

* Making your variable `public static` is the easiest way for your object to escape. Everybody have access to it now. 
* Suppose private array of strings as member, but your method returns that object as `public String[] getStates()` this lets client change the state. Make sure to send shallow or deep copy. 
* Always keep your constructor clean from any other initialization like eventListener or some other thread getting started. This can lead to importer usage of a partially constricted object. 

### Thread confinement
# Sharing Objects
* The Last chapter was about how we can avoid thread from accessing shared object via synchronized blocks, in this chapter we will study techniques for sharing and publishing objects so that they can be accessed by multiple threads.
* Also we will make sure that `memory visibility` is also there with along with sync on critical blocks.



### Visibility
```java
public class NoVisibility {

    private static boolean ready;
    private static int number;

    private static class ReaderThread extends Thread {
        public void run() {
            while (!ready) {
                Thread.yield();
            }
            System.out.println(number);
        }
    }

    public static void main(String[] args) {
        new ReaderThread().start();
        number = 42;
        ready = true;
    }
}
```
* Above class might look simple & correct, but in real it's not.
* This code does not guarantee visibility and ordering. As ReaderThread can keep on reading from cache (register's cache) and never see updated results.
* Even if it see updated results, it's possible that it sees `ready = true` before `number = 42`. So this code lacks both ordering and visibility.

#### Ways to solve this problem
**1. synchronized keyword (intrinsic lock)**
```java
public class NoVisibility {

    private static boolean ready;
    private static int number;
    private static final Object lock = new Object(); // common lock for proper synchronization

    private static class ReaderThread extends Thread {
        public void run() {
            while (true) {
                synchronized (lock) {
                    if (ready) {
                        System.out.println(number);
                        break;
                    }
                }
                Thread.yield();
            }
        }
    }

    public static void main(String[] args) {
        new ReaderThread().start();
        synchronized (lock) {
            number = 42;
            ready = true;
        }
    }
}
```

**2. Volatile Variable**
* When a field is set to volatile, the compiler and runtime are notified that these variable operations should not be reordered with other memory operations.
* They no longer cached on register, always most recently written by any thread will be returned for these variables.
* Book suggested to not use volatile over locks where visibility is required for reasoning.
* The problem with volatile is that even though it guranttees no operation reordering and visibility, but it do not guarantee for a atomic change on variable when multiple threads are changing it.
* The most common usecases for volatile are: status flags, interruption, shutdown checks, etc.
> Locking -> Visibility + Atomicity
>
> Volatile -> Visibility only

### Publication and Escape
* **Publication**: Making object available to other parts of code, either by retuning from method or passing in parameters.
* **Escape**: When an object is published when it shouldn't be, like accidentally making a private variable accessible to the outside world

* Making your variable `public static` is the easiest way for your object to escape. Everybody have access to it now.
* Suppose private array of strings as member, but your method returns that object as `public String[] getStates()` this lets client change the state. Make sure to send shallow or deep copy.
* Always keep your constructor clean from any other initialization like eventListener or some other thread getting started. This can lead to importer usage of a partially constricted object.

### Thread confinement
* Thread Confinement is a technique where you eliminate the room of object sharing among threads. 
* When we confine (attach or add) a data member to just one thread, it becomes thread safe. 
* Very similar pattern is used in JDBC connection pools, where one thread pick that connection and returns that to pool only when the response is sent to client, this pool object confinement with the thread makes pool object thread safe. 
* Primarily 2 types of confinement Stack & ThreadLocal. 
* **Stack Confinement**: All threads have local stack with them. When a thread executes a method, local variables created in that method get stored in thread's local stack, there is no way other thread can access those members.
```java
public int countPairs(int[] arr) {

    int count = 0;   // stack-confined
    int i = 0;

    while (i < arr.length - 1) {
        count++;     // take two elements as a pair
        i += 2;
    }

    return count;
}
```
* **ThreadLocal**: A class which allows you to associate value-holding object with your concerned thread. 
* Used when a object is need on per thread basis and you don't want to allocate memory or initialize the object again & again. 
* Heavily used as ContextHolders, for example in SpringSecurity the authenticated object gets stored in the SecurityContextHolder and gets removed (in finally block) when request is served. 
* ThreadLocal can be visualized as Map<Thread, T> but in reality, its implementation is very clever. 
```java
public class Thread implements Runnable {
    // ... other thread stuff ...

    /* * This is CRUX! Every thread carries its own personal Map.
     * The keys are the ThreadLocal objects, and the values are your data.
     */
    ThreadLocal.ThreadLocalMap threadLocals = null;
}
```
* So whenever you call `myThreadLocal.set("User123")`, internally OS searches for the thread with this call and in that thread's map it sets this value. Same happens while fetching the data from ThreadLocal.
* You can have as many as ThreadLocal objects and store value for your thread in that. 
```java
public class RequestContext {
    // We create THREE different ThreadLocal objects
    public static final ThreadLocal<String> userId = new ThreadLocal<>();
    public static final ThreadLocal<String> transactionId = new ThreadLocal<>();
    public static final ThreadLocal<Integer> securityLevel = new ThreadLocal<>();
}
```
Now a thread comes and executes. 
```java
RequestContext.userId.set("User123");
RequestContext.transactionId.set("Tx-999");
RequestContext.securityLevel.set(5);
```
Hence, we can have many ThreadLocals for that thread. 
* You can also create custom Class to store multiple info. 


### Immutability
* Immutable objects are always thread safe as they are available for read only. Hence, no inconsistent state. 
* Defining just data members as final won't make your class immutable. As `private final List<String> list` is private and final but with a setter one can add elements to it. 
> What makes a class immutable? 
> 
> * Have all it's fields private and final 
> * Make class final 
> * No setters 
> * Initialize all fields via constructor only
> * Defensive copying in constructor 
> * Return the deep copies so that client won't change object. 

> Always make all your fields private: unless you need greater visibility 
> 
> Always make all your fields final: unless they need to be mutable. 


### Safe Publication 
* Let's first see the case of unsafe publication. 
```java
// Unsafe publication example

public class UnsafePublication {
    public Holder holder;  // shared without synchronization

    public void initialize() {
        holder = new Holder(42);
    }
}

// Holder class
class Holder {
    private int n;
    public Holder(int n) {
        this.n = n;
    }

    public void assertSanity() {
        if (n != n) {
            throw new AssertionError("This statement is false.");
        }
    }
}
```
* Do you think `n != n` can ever be true? It can!
* Suppose there are 2 threads getting involved in construction of object UnsafePublication
1. Thread A creates `Holder(42)` without safe publication
2. Thread B calls `assertSanity()`
3. First read of `n` → sees stale value `0`
4. Value `42` becomes visible afterward
5. Second read of `n` → sees `42`
6 Comparison `0 != 42` → true
7. `AssertionError` thrown
> This happened because when threadB first read n, threadA changes might not have flushed to main memory, so threadB reads 0 as value, in between threadA's update gets flused to mainMemory and threadB reads42

#### How to fix this? 
* Make your class immutable, if the class involves mutable final fields, you need synchronization otherwise it's not needed. 
* For Mutable class, you have to do synchronization no other way around. 
> Rules for safely publish object: 
> 1. Storing reference in volatile field or AtomicReference 
> 2. Storing reference in the final field of a properly constructed object. 
> 3. Storing reference to it into a field that is properly guarded by lock, specially for mutable fields. 
> 4. Or use Concurrent Thread Safe collections. 

> Safest way if to do static initialization `public static Holder holder = new Holder(42`, here JVM ensures the thread safe initialization of the object. 


### Client Side Locking 
* Suppose in class `Collections.synchronizedList` I want to add a new method for `putIfAbsent` and it should be thread safe. 
* Implementation should look like
```java
//Not Thread Safe
public class ListHelper<E> {
    public final List<E> list = Collections.synchronizedList(new ArrayList<E>());

    //This gives the illusion of thread safety, but it is broken!
    public synchronized boolean putIfAbsent(E x) {
        boolean absent = !list.contains(x);
        if (absent) {
            list.add(x);
        }
        return absent;
    }
}
```
* Actually the Concurrent Collections have it's own synchronization implementations so `synchronized` on method won't know which monitor lock to aquire. 
* We have to acquire the very same lock which `Collections.synchronizedList` is having. 
* As per Java doc `Collections.synchronizedList` uses returned list itself as mutex lock. 
```java
//Thread Safe
public class ListHelper<E> {
    public final List<E> list = Collections.synchronizedList(new ArrayList<E>());

    //We are locking on the same lock the list uses internally.
    public boolean putIfAbsent(E x) {
        synchronized (list) { 
            boolean absent = !list.contains(x);
            if (absent) {
                list.add(x);
            }
            return absent;
        }
    }
}
```
* We call this client side locking as client (the ListHelper class) is locking the method as per the lock which the internal list uses. 
* We can also kinda avoid this by making list as private and handle other functions in it in synchronized fashion. 


