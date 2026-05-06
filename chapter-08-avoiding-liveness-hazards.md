# Chapter 8: Avoiding Liveness Hazards
* **Liveness Hazard** (liveness failure): A state where a application or thread is not able to make progress like getting stuck. 
> **Liveness**: Something good will eventually happen 
> 
> **Liveness failure**: application freezes, hangs, or spins endlessly. Eg: Deadlock, Livelock, Starvation


--- 

### Deadlock 
- When thread A holds lock L and tries to acquire lock M, but at the same time thread B holds M and tries to acquire L, both threads will wait forever.
- This can be visualized as cyclic graph. 
- Databases uses deadlock detection algorithms (mostly graph based), where the victim transaction gets aborted which is holding locks. 

```java
//Deadlock-prone
public class LeftRightDeadlock {
    private final Object left = new Object();
    private final Object right = new Object();

    public void leftRight() {
        synchronized (left) {
            synchronized (right) {
                doSomething();
            }
        }
    }

    public void rightLeft() {
        synchronized (right) {
            synchronized (left) {
                doSomethingElse();
            }
        }
    }
}
```
How to avoid ? Just make sure each thread acquires locks in the same order. 
> A program will be free of lock-ordering deadlocks if all threads acquire the locks they need in a fixed global order.

--- 

#### Dynamic lock order deadlock
```java
public void transferMoney(Account fromAccount, Account toAccount, DollarAmount amount) throws InsufficientFundsException {

    synchronized (fromAccount) {
        synchronized (toAccount) {

            if (fromAccount.getBalance().compareTo(amount) < 0) {
                throw new InsufficientFundsException();
            } else {
                fromAccount.debit(amount);
                toAccount.credit(amount);
            }
        }
    }
}
```
* Although the above code may seem like following lock ordering, but if two transactions start from both users, this will cause deadlock. 
* Solution: Take hash of both ids and acquire lock on a smaller one, in case of tie use a global tieLock then acquire any one. 


> Suppose your current thread owns a lock and in that code flow you're calling some alien method, whose implementation you don't know. So better to avoid calling alien methods while holding lock becasue you don't know what all locks that alien method can acquire to take. So better to do `Open Calls`, open call is nothing but calling alien method when your current Thread is not holding any lock. This will avoid the situation of deadlock. Open calls can be done using synchronized block and not keeping lock on the whole method. This way lock will be released while calling the alien method. 

> If you're acquiring multiple locks make sure, lock ordering is part of your design. Try to minimize multiple locking in single operation. 


* Time based lock `tryLock` can be used to avoid waiting for the lock which might cause deadlock. 
* If you just acquire one lock at a time, a lock-ordering deadlock is mathematically impossible.

--- 

### JVM Helping in Identifying Deadlocks 
* **Thread Dumps**: A thread dump is a snapshot of exactly what every single thread in the JVM is doing at that exact millisecond. It shows the stack trace, which locks the thread currently holds, and which locks it is waiting for.

--- 



### Other liveness hazards (_Till now we just saw lock ordering Deadlock_)
* **_Starvation_**: Causes of starvation are tweaking thread_priority (this priority thing is antipattern, never use this), infinite loop on the Thread which acquired lock; making other threads wait till that infinite loops ends.
* **_Livelock_**: A message that repeatedly fails processing due to a deterministic logic error and is requeued for retry can create a livelock, where the system makes no forward progress on that message. Such a message is referred to as a poison message.

> Solution to avoid livelock is adding jitter. In livelock CPU cycles are actively getting consumed whereas in deadlock, the threads are completely asleep/blocked and using zero CPU.

--- 
