# Chapter 5: Task Execution
```java
class ThreadPerTaskWebServer {
    public static void main(String[] args) throws IOException {
        ServerSocket socket = new ServerSocket(80);
        while (true) {
            final Socket connection = socket.accept();
            Runnable task = new Runnable() {
                public void run() {
                    handleRequest(connection);
                }
            };
            new Thread(task).start();
        }
    }
}
```
* Above code is a web server which handles multiple client request by creating thread for each request. It's much better than having single threaded web server. 
* But there are many cons to it: 
* 1. Thread lifecycle overhead. Creation, Terminating thread 
* 2. Resource Consumption. If we have more Runnable threads than the cores count, many threads sits idle. 
* 3. There can be limit to framework for the threads it can create. You can't create 1M threads for 1M incoming request. 

Even though with above code, we might get good responsiveness & throughput, but it's clearly poor resource management. 
> That's where the Executor framework comes

### Executor Framework
```java
public interface Executor{
    void execute(Runnable command); 
}
```
* We saw bounded queues in the previous chapter, which prevents overloaded application from running out of memory. It's one of compoenent here in Executor framework. 
* This works on `Producer-Consumer` model, where producer keeps on adding task & threads that execute task are consumers. 
* Execution policies are like in what thread task will be executed, task priority, how many concurrent task, rejection task policies, action before and after task execution. These act as a resource management tool which depends on quality-of-service and requirements. 

#### ThreadPool
* Homogeneous pool of worker threads, which take task from the queue, execute it, waits for the next task. 
* We don't need to create the threads each time we need them. We have them in pool, so thread creation time got saved. 
* We might need to decide the optimal pool size as per requirement. Such that enought thread to keep CPU busy and not as many which may cause unnecessary memory consumption. 
* **FixedThreadPool**: Creates pool with fixed thread count. Internally used `unbounded` LinkedQueue, so better to avoid if task count is very high. Go with custom thread pool in that case. 
* **CachedThreadPool**: No queues here, direct handoff of task from threads to execution. Thread idle for 60 sec gets terminated. Good for burst events like sending APNs/FCM push notifications which takes milliseconds. Avoid when requests arrive faster than they are getting processed, this usually results in a catastrophic java.lang.OutOfMemoryError: unable to create new native thread. It uses `SynchronousQueue` to be precise which have size zero, for handoff. 
* **SingleThreadPool**: Exactly one thread, backed by an unbounded queue. Guarantees serial execution. SingleThreadExecutor exists for control and guarantees, not parallelism.
* **ScheduledThreadPool**: Designed to execute tasks after a given delay or execute them periodically. It uses a DelayedWorkQueue. Usecase: Polling & Heartbeats.
> **Non-Daemon Threads** (User Threads): Primary threads of application. Manual thread creation creates a Non-Daemon thread until unless spawned from daemon thread context. JVM stay alive as long as there is one non-daemon thread running. 
> 
> **Daemon Thread**: Daemon threads are "background" service providers, used to server user threads. JVM just don't care about Daemon thread. JVM just stays alive till last user thread. 
> 
> In the case of SpringBoot application, it's **server-threads** (non-daemon) which keeps JVM alive even your main thread ends. 


#### Executor vs ExecutorService
* _Executor_: Simple interface with just one execute method. 
* _ExecutorService_: Extends Executor, Adds task management + lifecycle control
```java
public interface ExecutorService extends Executor { void shutdown();
    List<Runnable> shutdownNow(); //abrupt shutdown. Attempt to cancel running task and does not strat any new task 
    boolean isShutdown();
    boolean isTerminated();
    void shutdown(); // do not take new task, allow to complete running task 
    boolean awaitTermination(long timeout, TimeUnit unit) //checks if there is any still task running, if true. Do shutdownNow(), by developer if he wants to. 
    throws InterruptedException;
    // ... additional convenience methods for task submission
}

```

#### Delayed & Periodic Task 
* Java 5.0 used to use `Timer` (object which takes TimerTask for execution, single threaded) & `TimerTask` (which runs via Timer object). For a interval and fixed timed task. Deprecated now. Being singlethreaded blocks the prev sequential task and don't catch exception, lead to thread termination. 
* ScheduledThreadPoolExecutor is now extensively used and can run via multiple threads. Uses `DelayedWorkQueue` (idea from DelayQueue) .
> Understanding `DelayedWorkQueue` & `DelayQueue` in itself are broad and interesting topics. Definitely worth investing time. 


#### Exploitable Parallelism 
When a single client request can be decomposed into multiple independed subtask where the result of one subtask is not dependent on others. 
* Example: Client request came to render a webpage, we're passing a thread for this task. But this task can be decomposed into subtask like rendering text and image as separate subtasks. Here we did exploited parallelism. 

### Callable and Future 
* Executor uses Runnable as it's basic impl. Runnable -> (can't return values, can't throw checked exceptions)
```java
public interface Callable<V> {
    V call() throws Exception;
}
```
```java
public interface ExecutorService extends Executor {
    Future<?> submit(Runnable task);
    <T> Future<T> submit(Callable<T> task);
}
```
* When we submit a task to ExecutorService it is wrapped as FutureTask and all control to that is done by Future (represents lifecycle of task). 
* Why to cancel? To avoid long-running task. 
* Lifecycle of task: Submitted, Running, Completed/Canceled. 
```java
public interface Future<V> {
    //can try to cancel running task if they are responsive to Interruption. 
    boolean cancel(boolean mayInterruptIfRunning); //cancels task which are submitted but not started
    
    boolean isCancelled();

    boolean isDone();

    V get() throws InterruptedException, ExecutionException, CancellationException;

    V get(long timeout, TimeUnit unit)
            throws InterruptedException, ExecutionException,
            CancellationException, TimeoutException;
}
```


> FutureTask can also be submitted to ExecutorService as it implements Runnable. 


* Just because you can decompose a single task in multiple sub-tasks, you should not be doing this blindly. Suppose subtask-1 takes 10sec and subtask-2 takes 1 sec. With exploting parallelsim you are increasing performance by just 10%, which comes with the overhead of task co-ordination and resource location. Hence, parallelism should be exploited when we have a large number of tasks and doing so results in greater performance improvement. 

> Just gave a quick read to CompletionService, as it's very old and not used now. CompletableFuture is now used instead of CompletionService. 


* This chapter explained why you should consider Executor in cases when you need multiple threads, executor helps in thread lifecycle & supports a rich variety of execution policies