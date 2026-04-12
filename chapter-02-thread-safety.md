# Chapter 2: Thread Safety 
* Thread Safe Code: Managing access to state, in particular to shared, mutable state. It's not just about having multiple threads and use locks. Hence, protecting data from uncontrolled concurrent access. 
* Access to the shared object (can be changed via multiple threads) should be synchronized. 
* word `synchronization`: synchronized keyword, Atomic variables, volatile variables, explicit locks. 
* Ways to preserve state: don't share across threads, immutability or using synchronization

### Thread Safety 
* A class can be called thread safe when it continues to behave correctly when accessed by multiple threads. 
* Thread safe classes provide in built synchronization so that client doesn't need to provide its own. 
* **Stateless objects (no instance (shared) mutable data) are always thread-safe.**

```java
@RestController
public class FactorController {

    @GetMapping("/factor")
    public List<Integer> factor(int n) {
        List<Integer> result = new ArrayList<>();

        for (int i = 2; i <= n; i++) {
            while (n % i == 0) {
                result.add(i);
                n /= i;
            }
        }
        return result;
    }
}
```
* In the above example, even if 200 client comes and uses 200 tomcat threads, as this controller class is stateless the result won't be wrong even without synchronization logic. Because there is no shared state for controller. 
* There will be just one `FactorController` and each thread will have own stack space & there is no way threads can enter to each other's stack space. 

### Atomicity 
* If in our factor example we want to know how much calls are there to controller, we may need to introduce a instance variable for controller. 
```java
@RestController
public class UnsafeCountingController {

    private long count = 0; // shared mutable state

    @GetMapping("/factor")
    public List<Integer> factor(int n) {

        List<Integer> result = new ArrayList<>();

        for (int i = 2; i <= n; i++) {
            while (n % i == 0) {
                result.add(i);
                n /= i;
            }
        }

        count++; // NOT thread-safe
        return result;
    }

    @GetMapping("/count")
    public long getCount() {
        return count;
    }
}
```
* Now the controller is not thread safe as operations like `++count` or `count++` are not thread safe as they rely on 3 operations which are `read-modify-write`. Hence, two threads can read same old value and write the same value, whereas on each method call we are expecting incremented value. 
> Race condition = uncontrolled access to shared mutable state under concurrency & the final result depends on the timing/interleaving of execution
* Most common race condition is `check-then-act`. Suppose you observe something to be true (file doesn't exist), so you ahead by creating one but later on observation becomes invalid as someone else created that file in the meantime (can cause file corruption, overwritten data etc.)
* The classic case for above point is `Singleton` class's `Lazy Initialization` implementation. 
```java
// Not thread safe
public class DelayedInitializer {

    private HeavyResource resource = null;

    public HeavyResource getResource() {
        if (resource == null) {
            resource = new HeavyResource(); //may return two different object to two callers 
        }
        return resource;
    }
}
//Classic lazy initialization race condition when two threads can read null at the same time and create two different objects. 
```

> **Compound actions**:  `check-then-act` & `read-modify-write`. They might look atomic operations but consist of multiple internal operations. Can be avoided by using "AtomicLong" or "synchronized" keyword

```java
@RestController
public class SafeCountingController {

    private final AtomicLong count = new AtomicLong(0);

    @GetMapping("/increment")
    public long increment() {
        return count.incrementAndGet();
    }

    @GetMapping("/count")
    public long getCount() {
        return count.get();
    }
}
```

#### Why count.incrementAndGet(); is thread safe? 
* Because it relies on CAS (Compare-And-Swap)
```java
//Pseudo Logic 
while (true) {
    long current = value;
    long next = current + 1;

    if (compareAndSwap(current, next)) { //(expectedValue, newValue)
        return next;
    }
}
```
> compareAndSwap ensures: "Update only if no one else has changed the value since I last saw it."


### Locking 
* Suppose you have two thread safe variables in your class. Will this make your class thread safe ? Not really for example: 
```java
@RestController
public class UnsafeCachingFactorizerController {
    private final AtomicReference<BigInteger> lastNumber = new AtomicReference<>();
    private final AtomicReference<List<BigInteger>> lastFactors = new AtomicReference<>();

    @GetMapping("/factor-cached")
    public ResponseEntity<List<BigInteger>> factorize(@RequestParam BigInteger number) {
        
        // 1. Check the cache
        if (number.equals(lastNumber.get())) {
            return ResponseEntity.ok(lastFactors.get());
        } else {
            List<BigInteger> factors = factor(number);
            lastNumber.set(number); // Line Y 
            //suppose now due to context switch, another thread comes with some different number 
            lastFactors.set(factors);
            return ResponseEntity.ok(factors);
        }
    }

    private List<BigInteger> factor(BigInteger i) {
        //mathematical factorization logic...
        return List.of(i); 
    }
}
```
```text
* lastNumber = 15, lastFactors = [3, 5]
* Now if ThreadA comes with number 10, goes to else condition and after Line Y context switch happens. 
* ThreadB comes with number 10, ThreadA set the lastNumber as 10, so ThreadB goes to `if` block and returns the wrong factors `[3, 5]`
* Hence we need synchronization between these two thread safe variables too. 
```

#### Intrinsic Locks  (Or Monitor Locks)
* Every single java object has built in lock associated with it handled internally by JVM. 
* We can acquire these locks by entering synchronized blocks or methods. 
* Intrinsic behaves like mutexes as only one thread is allowed to enter the synchronized section. 
* Locks are automatically released by object as soon as the thread exits the synchronized block
> Downsides of Intrinsic Locks: No manual interruption to threads (no timeouts), no fairness. 


```java
//Object Level Lock 
synchronized (this) {
    // critical section
}
```

```java
// Class level lock 
synchronized (MyClass.java) {
    // critical section
}
```

#### Reentrancy 
Reentrancy means locks are acquired per thread basis not per invocation basis. 
* It tracks: lockCount + OwningThread
* When the lockCount is zero, lock is considered unheld. 
* When a new thread comes, lockCount increases and JVM records the owner. 
* Count decreases when owning thread exists the synchronized block. 

```java
public class Widget {

    public synchronized void doSomething() {
        System.out.println(toString() + ": calling doSomething");
    }
}

public class LoggingWidget extends Widget {

    @Override
    public synchronized void doSomething() {
        System.out.println(toString() + ": calling doSomething");
        super.doSomething();
    }
}
```
* With above code one might think that there can be situation of deadlock when calling `super` method . 
* But as we saw that intrinsic locks are reentrant in nature. Hence, for same thread owner it can acquire the lock, the lockCount will become 2. 



### Guarding state with locks
* Make sure to guard the whole compound action with lock (not just write operations, as we saw wrong reads impacting results)
* There is no relationship between object's intrinsic lock and its state. Using synchronized(this) don't make a super shield around your object, JVM just make sures that no other thread should be able to acquire that very same lock. 
* Some other threads can freely change your instance variable and corrupt your data. It's developer's responsibility to handle that.
> Locks don't lock data. They lock code paths.
* Every variable involved in an invariant must be guarded by the EXACT SAME lock. We saw this in lastFactor, lastNumber example. 

#### Why can't we make everything synchronized ? 
```java
if (!vector.contains(element)) {
    vector.add(element);
}
```
```java
synchronized (vector) {
    if (!vector.contains(element)) {
        vector.add(element);
    }
}
```
* Here `contains` & `add` method is thread safe. But the distance or invocation time between them is not. 
* This is a Check-Then-Act race condition.
* ThreadA comes checks element don't exist (ready to add element5). Context switch 
* ThreadB comes checks element don't exist, adds element5. 
* ThreadA wakes up, and he also adds the same element5. Hence, vector has duplicate values now. 
* Also, synchronizing your each method may liveliness issues, where 99% time threads are just waiting for locks. 

### Liveness and performance
Avoid holding locks during lengthy computations or operations at risk of not completing quickly such as network or console I/O.