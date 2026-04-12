# Chapter 1: Introduction 

* **Resource utilization**: While IO operations let thread do some computation task instead of waiting on IO. (IO -> Input Output , file read or DB operations)
* **Fairness**: No thread should be starved and must get reasonable CPU share (context switching). 
* Threads are called lightweight processes, they share file handles and memory & has it's own program counter, stack, local variables. 

### Benefits of Threads 
* Converting sequential task to async. 
* Increase the throughput (Amount of work completed per time) of the system by increasing the processor and thread counts. 
* Use thread per client model (which most of the servers already do). Another way is to use single threaded server with Non-blocking IO code which includes callbacks and a hell lot of debugging when something fails.  
> Another good topic of discussion is how redis being single threaded handles so much load. In short, it's mainly due to in-memory operations, multiplexing (epoll/kqueue, Event loop), pipelining (client-side request batching)

### Risks of threads
Just because threads gets your work done faster dosen't mean you can spawn as many threads as you want, it comes up with a cost of unpredictable output if synchronization points are not properly handled. 
```java
// Not Thread Safe
public class UnsafeSequence {
    private int value;
/** Intended to return a unique value. */
    public int getNext() { 
        return value++; 
    } 
}
```
* Spawning multiple threads in this case will give incorrect results due to no method synchronization   
* Two threads ask for `value` can give same results whereas they should be different. 
* `getNext()` consist of 3 operations: read `value`, increment, write to `value`. Hence not `atomic` and prone to wrong results. 

```java
//Thread Safe
public synchronized int getNext() { 
    return nextValue++;
}
```
* Doing so will allow only one thread to make changes in variable. Hence, unique value on each invocation. 

**Liveness hazards**: Deadlock, Starvation, Resource contention(thread blocked), Priority inversion, Livelock

* **LiveLock** 
```text
Thread A and Thread B both need a database connection and a file handle to process a request.

Thread A grabs the database connection.

Thread B grabs the file handle.

Thread A tries to grab the file handle, fails, and decides to be "polite." It releases the database connection and immediately starts over.

Thread B tries to grab the database connection, fails, decides to be "polite," releases the file handle, and immediately starts over.

Thread A grabs the database connection again. Thread B grabs the file handle again.
```
* LiveLock is different from Deadlock, here in LiveLock the threads are active, consuming CPU cycles but never doing productive progress. Whereas in deadlock, two threads literally block each and other & wait for each other to release the lock.
* LiveLock can be avoided by using jitter between calls of eg: 20 ms, or using exponential backoff. 

* Threads do improve performance even on single cores due to context switching. But using huge thread count can backfire in context switching, as now CPU will send more time on restoring the context & scheduling threads. Hence, it should be managed properly.


### Threads are everywhere
* Even if we don't create threads on our own, the framework creates them. For eg: tomcat server spawns around 200 threads (by default) for client. 
* **Remote Method Invocation (RMI)**: RMI allows to make remote calls to different JVM servers which includes marshaling (serialization) * Unmarshaling (deserialization). RMI also spawns many worker threads (as it can have request from many remote servers) which developers don't manage. But if your remote object have internal state (counter, cache, DB Connection) it's developer's responsibility to protect that state via synchronization, locks, or thread-safe data structures. 
