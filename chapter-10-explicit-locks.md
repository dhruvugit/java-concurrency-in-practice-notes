# Chapter 10: Explicit Locks

```java
public interface Lock {

    void lock();
    void lockInterruptibly() throws InterruptedException;
    boolean tryLock();
    boolean tryLock(long timeout, TimeUnit unit) throws InterruptedException;
    void unlock();
    Condition newCondition();
}
```
* `ReentrantLock` are one of explicit locks, they have same memory semantics as intrinsic lock (synchronized), they just come with more advanced controls like not waiting forever for a lock, and interrupting a thread waiting for a lock. 
* Make sure you're doing `unlock()` in `finally` block. 
* Intrinsic lock acquisition is not responsive with `interrupt` operation. But explicit locks are, via `lockInterruptibly()`, also `timed tryLock()`
* Now we need two try blocks as they are responsive to interruption too (or throw InterruptedException) 
* For the same task, you won't see a huge difference in intrinsic & reentrantLock. 


### Fairness
* Fairness simply means, if a thread is waiting for a lock for a long time, it will get the priority for acquisition. 
* Non-Fairness means that even if a thread is already waiting for the lock, if a new thread comes at the exact instance when the lock is released, the new thread will get the lock irrespective of who else was waiting. Otherwise, this also sits in the queue. `Barging` is term which gets used to skip the queue.  
* Fairness might look a good choice at instance but it comes up with heavy OS overhead to suspend and resume threads. 
> Don't blindly make your lock fairness enabled. 
* Fairness locking can be used when the locks are held for long duration, this will reduce the chances of new thread coming frequently for new acquisition. 

> Use ReentrantLock when you need its advanced features like timed, polled or interrupt lock acquisition. Otherwise use synchronized as you don't need to care about lock and unlocks. 

#### Read-write locks 
* Multiple Readers allowed but only one writer at a time. 
* Excellent for read heavy data structures which may also need modifications at time. 
```java
public interface ReadWriteLock { 
    Lock readLock();
    Lock writeLock();
}
```
