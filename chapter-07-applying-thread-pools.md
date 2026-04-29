# Chapter 7: Applying Thread Pools 
* Running dependent task on a `SingleThreadExecutor` can cause deadlocks. 
* Thread Starvation Deadlock: If a task running inside a thread pool submits subtasks to the same thread pool, and then waits for those subtasks to finish (Future.get()), it can cause a deadlock. In a SingleThreadExecutor, the parent task holds the only available thread hostage while waiting for the subtask, but the subtask is stuck in the queue waiting for that exact thread to become free.
```java
public class ThreadDeadlock {
    ExecutorService exec = Executors.newSingleThreadExecutor();
    public class RenderPageTask implements Callable<String> {
        public String call() throws Exception {
            Future<String> header, footer;
            header = exec.submit(new LoadFileTask("header.html"));
            footer = exec.submit(new LoadFileTask("footer.html"));
            String page = renderBody();
            // Will deadlock -- task waiting for result of subtask
            return header.get() + page + footer.get();
        }
    }
}
```
* **Solution**: Use multithreaded pools, or use CompletableFuture. 
* For **Long Running Task**, never let unbounded wait for results, always use timed-wait operations. So that your thread is not clogged with long-running task.

---

### Sizing Thread Pools 
* The Very first step is to make sure you're not using extremes like too big or too small threadpool sizes. If used too big, high memory usage, if used too small -- under utilization of resources.
* Brainstorm on things like: processor cores of system, how much memory?, task is CPU heavy or I/O heavy? , 
* If it's a CPU bound task use `(N + 1)` threads, where N is CPU core count. 
* If it's an I/O bound task or blocking operation, you need a large pool size and some thread will be waiting for I/O result and won't be consuming CPU cycles. So this will depend on the waiting time for threads for those blocking operations. 
* And size can be tuned by using different pool size and benchmarking the CPU utilization. 

$$N_{threads} = N_{cpu} * U_{cpu} * \left(1 + \frac{W}{C}\right)$$

* **$N_{cpu}$**: Total number of CPU cores available.
* **$U_{cpu}$**: Target CPU utilization (from 0.0 to 1.0). If you want to leave some CPU breathing room for other background OS tasks, you might target 0.8 (80%).
* **$W$**: Wait time (time spent waiting on DB, network, etc.).
* **$C$**: Compute time (time spent actively processing data).
* **$\frac{W}{C}$**: The Wait-to-Compute ratio. If a task spends 90ms waiting on the database and 10ms processing the result, the ratio is 9.

> In Production systems we don't use these formula and get a final values, instead the optimal value is decided via load testing using tools like JMeter, Gatling, etc.

---

### Configuring ThreadPoolExecutor

```java
public ThreadPoolExecutor(
        int corePoolSize, 
        int maximumPoolSize,
        long keepAliveTime,
        TimeUnit unit,
        BlockingQueue<Runnable> workQueue, 
        ThreadFactory threadFactory, 
        RejectedExecutionHandler handler) { ... }


corePoolSize: baseline thread count (always kept)
maximumPoolSize: upper limit of threads
keepAliveTime: how long idle extra threads wait before terminating
unit: unit for keepAliveTime
workQueue: stores tasks before execution
threadFactory: defines how threads are created
handler: action when task cannot be accepted
```

* Bounded ThreadPools limit the tasks that can be run concurrently 
* For Bounded ones, the task which can not be executed is kept waiting in queue. The only concern in this case is that what if client keeps on sending tasks faster than they can be processed? Even before running out of memory the `response time` for queued requested will be drastically impacted already.
* There are 3 main approaches of task queueing: 
* 1. Un-bounded queue 
* 2. Bounded queue
* 3. Synchronous Hand-off

> `newFixedThreadPool` & `newSingleThreadExecutor`, uses unbounded queues. 
> 
> 
> A more stable resource management strategy is to use bounded queue, but what happens to the task which comes when the queue is full? 
> 
> Another way is to use `SynchronousQueue` with unbounded pools. `SynchronousQueue` is not a actual queue, instead it's direct hand-off, task arrives - assign directly to thread, if no thread - create one, if you can not create thread - reject the task as per `saturation policy`. Direct hand-off can also be efficient as you don't need to place the task in queue and then retrieve when ready to be executed. 
> 
> Use `SynchronousQueue` only when the pool is unbounded or task rejection can be considered. 


---

**Saturation policy**: rule applied when thread pool is full (max threads reached + queue full).

**Common policies:**

* **AbortPolicy**: throws exception, rejects task
* **CallerRunsPolicy**: calling thread executes task
* **DiscardPolicy**: silently drops task
* **DiscardOldestPolicy**: drops oldest queued task, adds new one


---

#### Thread Factories 
```java
public interface ThreadFactory { 
    Thread newThread(Runnable r);
}
```
* It helps define what kind of thread you want from your thread pool to be created. You can give names to thread, change priority, set Daemon etc. _(last 2 are not recommended to be done)_
* You can add logs when a Thread gets created or terminated. 

> We also have Threadpool hooks (function calls) like beforeExecute (per task), afterExecute (per task), and terminate (when pool shuts down). 
> Can be used to find time taken per thread to execute tasks. 


### Parallelizing recursive algorithms
* Suppose you are traversing on a tree and on each node you're doing some heavy operations, and adding result for that node into a collection. We can do this by spawning thread for each node's computation.

#### Sequential Way 
```java
public<T> void sequentialRecursive(List<Node<T>> nodes, Collection<T> results) {
    for (Node<T> n : nodes) { 
        results.add(n.compute()); sequentialRecursive(n.getChildren(), results);
    }
}
```
* Here main thread waits for each node's computation to finish, before proceeding to others. 

#### Parallel Way 
```java
public <T> void parallelRecursive(final Executor exec, List<Node<T>> nodes, final Collection<T> results
) {
    for (final Node<T> n : nodes) {
        exec.execute(new Runnable() {
            public void run() {
                results.add(n.compute());
            }
        });
        parallelRecursive(exec, n.getChildren(), results);
    }
}
```
* Here we delegate compute part to the pool's worker thread and gave recursive calls to child nodes.




