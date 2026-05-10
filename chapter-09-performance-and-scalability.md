# Chapter 9: Performance and Scalability 

* First make your program right, then work on improving it's performance. 
* Improving performance means doing more work with fewer or fixed resources.
* Just increasing thread won't proportionally increase your application's performance, you also have to consider locking, signaling, and memory synchronization, thread creation, teardown, scheduling. If not monitored well even with multiple threads your application could perform worse than a single threaded application. 
* how fast: service time, latency
* how much: capacity, throughput
> **Scalability** describes the ability to improve throughput or capacity when additional computing resources (such as additional CPUs, memory, stor- age, or I/O bandwidth) are added.
* Making a system capable of handling massive scale almost always means making it slightly slower at handling a single task. In simple words suppose your monolith is doing multiple interactions within single request with response time of 900ms. Now we need to split the layers as we have many requests coming and need to scale, now among the microservices the latency will increase due to network IO, but now system is handling more requests with a cost of slightly slower response time.

> **Performance** (How fast): You're trying to reduce latency and service time. By optimizing algo.
> 
> **Scalability** (How much): This is about increasing throughput and capacity,  don't necessarily care if a single task gets faster
> 
> Hence performance and scalability are often at odds, breaking things increases time for unified result or effort. 


* No optimization is universally "faster." Performance is entirely dependent on the context and the dataset. Eg: Bubble sort is faster for tiny dataset than Quick-sort. 
> `Arrays.sort()` uses Tim-sort for array size greater than 32, for 32 and below it uses simple Binary Insertion Sort. 

### Amdahl’s law
Amdahl’s law describes how much a program can theoretically be sped up by additional computing resources, based on the proportion of parallelizable and serial components. If F is the fraction of the calculation that must be executed serially, then Amdahl’s law says that on a machine with N processors, we can achieve a speedup of at most:
$$ \text{Speedup} \le \frac{1}{F + \frac{1-F}{N}} $$

Eg: 10% task must be serial, so if we put infinite threads, the max performance improvement we can get is 10x. 
> Just because your task can run on multiple threads this doesn't mean you can keep on increasing the processors. Threadpool's thread taking task from blocking queue, here taking task there is some level of synchronization. 
> 
> All concurrent applications have some sources of serialization; if you think yours does not, look again.

### Costs introduced by threads
1. **Context Switching**
2. **Memory synchronization**: using `synchronized` & `volatile` forces data to be flushed to main memory (via **memory barriers**) which slow down things. 
3. **Blocking**
* **Uncontended Synchronization**: No other threadB needs the lock (won't wait), for current aquired lock by some threadA. 
* **Contended Synchronization**: Multiple threads try to acquire the same lock simultaneously.

> In Contended Synchronization, JVM have 2 ways to handle thread, first is spin-waiting (keep on checking for lock), another is Suspension (Sleeping).  



### Reducing lock contention
```text
There are three ways to reduce lock contention:
• Reduce the duration for which locks are held;
• Reduce the frequency with which locks are requested; or
• Replace exclusive locks with coordination mechanisms that permit
greater concurrency.
```
1. **Narrowing lock scope**: Hold locks for the shortest possible duration to reduce thread contention and increase throughput.Only use `synchronized` for read, write shared or mutable states, not over full method. 
2. **Reducing lock granularity**: If a lock guards more than one independent state variable, you may be able to improve scalability by splitting it into multiple locks that each guard different variables. This results in each lock being requested less often.
```java
/* 
        Here, as x and y are independent, it's better to use separate locks. 
        This clears reduces lock contention. 
 */
class Example {
    private final Object lockA = new Object();
    private final Object lockB = new Object();
    private int x = 0;
    private int y = 0;

    void incX() {
        synchronized (lockA) {
            x++;
        }
    }
    
    void incY() {
        synchronized (lockB) {
            y++;
        }
    }
}
```

3. **Lock striping**: This involves stripping the lock in partition for a shared data. Just like we did in ConcurrentHashMap, where 16 concurrent writes can be done on it. 
For this we do something like: 
```java
// Synchronization policy: buckets[n] guarded by locks[n%N_LOCKS]
private static final int N_LOCKS = 16; private final Node[] buckets;
private final Object[] locks;

//then in synchrnozied blocks use it as 
synchronized (locks[hash % N_LOCKS]){
    
}
```
> Although this is for old implementation of `ConcurrentHashMap`, not it's done by locking head node using sychronized and CAS operations. 

4. **Avoiding hot fields**: `count` variable in `ConcurrentHashMap` will be hot field, as even though it has stripes each thread needs to increase or decrease count (this all to avoid brute force counting). 
Solution: Use count variable for each stripe and for final count go through count of all buckets. 

5. **Alternatives to exclusive locks**: Using the concurrent collections, read-write locks (good for read-only), immutable objects and atomic variables.

6. **Monitoring CPU utilization**: If CPU is not fully utilized there can be many reasons to it, such as: Insufficient load, I/O bound, Externally bound, lock contention

7. **Just say no to object pooling**: Don't blindly do Object Pooling, as it comes with cost of synchronization, guessing pool size, cleanups and other things, better to do object creation directly. Object Pooling makes sense when object creation is very heavy like DB connection and Threads (use executor). 


#### Reducing context switch overhead (Re-read this para from book)
* In-line logging is expensive. Output stream (System.out) is synchronized → multiple threads contend for its lock
* Fix – Dedicated Logger Thread (producer-consumer): Use a queue to send the logs 
> Note: Read on internals of how logging frameworks work and their impact on applciation performance. 
