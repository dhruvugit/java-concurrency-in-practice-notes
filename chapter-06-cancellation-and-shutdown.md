# Chapter 6: Cancellation and Shutdown
* We might want to cancel/stop a task before it's completion due to client cancellation or application needs to shut down.
* Java provides a cooperative mechanism `Interruption`. Where one thread asks another thread to stop whatever task it is doing. We don't want thread to immediately stop and abandon the task as it can lead data structures in corrupted stated if not cleaned properly. So developer has to handle this Interruption and do cleanup for efficient termination.
* **Reasons to cancel task**: User-requested cancellation, time bound operations, errors/shutdowns.
* **Cancelling task via flag**:
```java
public class Worker implements Runnable {
    private volatile boolean cancelled; //critical to make it volatile 

    public void run() {
        while (!cancelled) {
            System.out.println("Working...");
            //Thread.sleep(50000);
        }
    }
    public void cancel() {
        cancelled = true;
    }
}
```
* In the above code, suppose you did thread.sleep for 50 sec, so even if you cancel the task it will look for updated flag only after 50 sec.
* So better to use InterruptionException that Java provides for stopping the task.
* So above approach for blocking task like `BlockingQueue.take()` or `Thread.sleep` won't stop the thread immediately.
* **Interruption**:
    * Each thread have `isInterrupted` flag which blocking methods like `BlockingQueue.take()`, `Object.wait()`, `Thread.sleep` tracks, whenever they see the `isInterrupted` flag to be true (flip done by thread.interrupt), they try to stop the task they are doing by throwing InterruptionException. JVM do not guarantees how these blocking method will respond to this, but it's  reasonably quick.
    * When blocking methods throw `InterruptedException`, the JVM automatically clears the `isInterrupted` flag (sets it back to false) before entering the catch block. If you are writing a task that runs in a thread pool, you should do your cleanup in the catch block and then restore the interrupt status by calling `Thread.currentThread().interrupt()`. This ensures the ThreadPool (the owner) knows the interrupt occurred.
    * Calling `interrupt()` does not necessarily stop the target thread from doing what it is doing, it merely delivers message that interruption has been requested. Blocking methods are very reactive to this.
> What if there is no method which reacts to `interrupt` ? It will simply set the interrupt flag to true and if your code supports somem while loop which reacts to this flag it will stop the task, else it will keep on running.

> If ThreadA aquired monitor lock via synchronized, and threadB is waiting for lock. Calling  `interrupt()` on ThreadB won't stop it from waiting it will keep on waiting for the lock. Until unless you use locks of type `lockInterruptibly()`

> We as developers rarely do `Thread.interrupt()` it's the framework which internally extensively use this, like Graceful Server Shutdowns (Spring Boot / Tomcat), closing DB connections.


* Cancellation is a concept where application decides to stop the task which is no longer needed, whereas `Interruptoin` is mechanism in Java which helps to cancel task.
* With thread getting interrupted will thread be dead or ready to take new task? Actually it depends why thread was interrupted. If shutdownNow() is called, the ThreadPool changes its master state to STOP and interrupts all threads. The worker threads see this STOP state and gracefully die, refusing new tasks. If a task is cancelled via Future.cancel(), the pool is still RUNNING. The framework intercepts the cancellation, interrupts the specific worker thread, and then internally clears the thread's interrupt flag so the clean thread can be reused for the next task.
* Only code (ThreadPool here) that implements a thread’s interruption policy may swallow an interruption request. General-purpose task and library code should never swallow interruption requests. Swallow means catching the `InterruptionException` and doing nothing.
* If your code don't have blocking methods, they can still be responsive to Interruption via `isInterrupted` flag. But make sure to check on this flag before doing any critical task rather than just checking at the beginning.

#### Timed-run Example
```java
private static final ScheduledExecutorService cancelExec = ...;

public static void timedRun(Runnable r, long timeout, TimeUnit unit) {
    final Thread taskThread = Thread.currentThread();

    cancelExec.schedule(new Runnable() { //thread of pool 
        @Override
        public void run() {
            taskThread.interrupt();
        }
    }, timeout, unit);

    r.run();
}
```
* Above code is a classic example of one thread not respecting or knowing the interruption policy of thread.
* Suppose some other threadA call the method `timedRun`, internally it uses thread from ScheduledExecutorService. Suppose task gets finished in 2 sec and timeout for interruption is 10 sec. Now when this thread will go to pool after 2 sec. And on 10th sec if that very thread if executing some other task, the new task will interrupt. That's why interruption should be done by one who owns the thread. Threadpool was not aware of this interruption timeout. So one should handle this by creating own thread in timedRun and call interruption on that.
* **The best way to cancel a running task is via `Futute.cancel()`.**
* When the `Future` object throws `InterruptedException` or `TimeOutException` this simply means that result is no longer needed, cancel the task with `Future.cancel()`.
```java
public class TimedRunExample {
    private static final ExecutorService taskExec = Executors.newCachedThreadPool();

    public static void timedRun(Runnable r, long timeout, TimeUnit unit) throws InterruptedException {

        Future<?> task = taskExec.submit(r);

        try {
            task.get(timeout, unit);
        } catch (TimeoutException e) {
            // timeout occurred; task will be cancelled in finally
        } catch (ExecutionException e) {
            // exception thrown inside task
            throw launderThrowable(e.getCause());
        } finally {
            // safe even if already completed
            task.cancel(true); // interrupts if running
        }
    }
}
```

> **_Sockets_** in java are not responsive to Interruption (as they are OS level blocking operations). In order to stop or close the sockets you need to explicity call the `socket.close()` from other threards or even override the `ThreadPool` to make other methods convinent with Socket. In modern Java we don't use these instead we use Virtual Threads or NIO frameworks to work with blocking sockets which are 100% responsive to Interruption.

> The JVM does not have ultimate control over OS-level blocking operations. For eg: above socket we saw, as they work on OS level thread logic written in C and they don't care about JVM's isInterruptedFlag.




### Stopping a thread-based service (Stopping Producer-Consumer models)
* You should never interrupt or change the priority of thread until at least you don't own it. Suppose you created some threads via ThreadPool, here you should never `interrupt` or `change priority` of these threads as you don't own them and you're not aware of its policies.
* Thread pool provide methods like `shutDown` `shutdownNow` use them for interruption. They are directly provided by owner i.e. `ThreadPool`.

#### Logging Service

#### Approach 1: Infinite Logger
```java
public class LogWriter {
    private final BlockingQueue<String> queue = new LinkedBlockingQueue<>(100);
    private final LoggerThread logger = new LoggerThread();

    // Producers call this
    public void log(String msg) throws InterruptedException {
        queue.put(msg); // Blocks if the queue is full
    }

    private class LoggerThread extends Thread {
        public void run() {
            try {
                while (true) { // Infinite loop!
                    writer.println(queue.take()); // Blocks if queue is empty
                }
            } catch(InterruptedException ignored) { } 
        }
    }
}
```
* Here the consumer is having just one thread to write the logs, and the Producer keeps adding task to the blocking queue.
* There is no shutdown mechanism, thread will be blocked on `queue.take()` forever.

#### Approach 2: Flag based shutdown
```java
public class UnreliableLogWriter {
    private boolean shutdownRequested = false;

    public void log(String msg) throws InterruptedException {
        // FLawed: Check-then-act race condition
        if (!shutdownRequested) {
            queue.put(msg);
        } else {
            throw new IllegalStateException("logger is shut down");
        }
    }
    
    public void stop() {
        shutdownRequested = true;
        loggerThread.interrupt(); // Wake up the consumer to exit
    }
}
```
* Suppose queue size is 1 and the current capacity is full.
* ThreadA comes for `log` and sees `!shutdownRequested` giving true, before hitting `put` method, context switch happens.
* ThreadB comes with `stop` request `shutdownRequested` to be true and interrupt logger, so consumer is down now.
* ThreadA now calls `queue.take()` and will block forever, as queue is full and consumer is down. Hence producer will keep on waiting with `put` method.

#### Approach 3:
```java
public class ReliableLogService {
    private boolean isShutdown;
    private int reservations; // The "ticket" counter

    public void log(String msg) throws InterruptedException {
        synchronized (this) {
            if (isShutdown) throw new IllegalStateException("Shutdown");
            ++reservations; // 1. Reserve a spot atomically
        }
        queue.put(msg);     // 2. Put the message outside the lock
    }

    private class LoggerThread extends Thread {
        public void run() {
            while (true) {
                synchronized (ReliableLogService.this) {
                    // Consumer only exits if shut down AND all reserved tickets are processed
                    if (isShutdown && reservations == 0) {
                        break; 
                    }
                }
                String msg = queue.take();
                synchronized (ReliableLogService.this) {
                    --reservations; // 3. Fulfill the ticket
                }
                writer.println(msg);
            }
        }
    }
}
```
* Here we have a counter for the entries of logs and we won't let consumer die before the count goes to zero.
* This handles the race-condition, but it's extremely complex as we are handling locks manually and will also make system slow.


#### Approach 4: Delegation via ExecutorService
* This is kind of the best solution for cancellation in Producer-Consumer scenario
```java
public class ModernLogService {
    private final ExecutorService exec = Executors.newSingleThreadExecutor();

    public void log(String msg) {
        try {
            exec.execute(() -> writer.println(msg));
        } catch (RejectedExecutionException ignored) {
        }
    }

    public void stop() throws InterruptedException {
        try {
            exec.shutdown();
            exec.awaitTermination(5, TimeUnit.SECONDS);
        } finally {
            writer.close();
        }
    }
}
```
* Here producer is not blocked as we are using `shutdown()` which rejects taking new logs.
* With `awaitTermination` we are waiting for tasks to get completed. But here is a catch there can be some task which do not gets completed within 5 sec and will lead to termination. Another way to handle this is to use shutDownNow after a wait and handle those tasks accordingly.
* Here we did delegation to ExecutorService which gracefully handled shutdown for us.

> In 4th approach we have a hard architectural choice
> 1. Block forever and let all tasks complete first, we already blocked producer, so we are good with producer.
> 2. Aggressively kill the remaining tasks to ensure the JVM can shut down

* Poison Pill: Another way to stop a producer consumer is using `Poison Pill`. We add a entry to the queue from which no new tasks will be accepted. For poison pill approach, the number of consumers and producers should be known beforehand. Suppose you have 3 producer, 4 consumer. You'll shutdown your service when you get exactly 3 poison pills.

#### shutdownNow Limitations
* `shutdownNow` does two things:
* 1. Returns the list of tasks that never started (still in the queue)
* 2. Sends interrupt signals to currently running tasks

* It don't tell you while tasks were in Running state, and got canceled abruptly, due to force shutdown.
* There is one way to get the list of such `Runnable` task.
```java
public void execute(final Runnable runnable) {
    exec.execute(new Runnable() {
        public void run() {
            try {
                runnable.run();          // run the actual task
            } finally {
                // After task finishes (or gets interrupted), check:
                if (isShutdown() && Thread.currentThread().isInterrupted()) {
                    tasksCancelledAtShutdown.add(runnable);  // record it as cancelled
                }
            }
        }
    });
}
```
With each task completion just check in it's finally block that is executor shutdown ? & was current thread interrupted ? If yes, add that task in taskCancelledAtShutdown.
> In the above example, there is a subtle race condition. It's `check-and-act`. Suppose a task has just completed execution and is in its last line of execution, and before the executor knows about its completion, someone calls `shutdownNow`. Now for that task, the condition in the finally block will be true, and it will be added to cancelled tasks, whereas it was actually completed. So use this approach just for the operations which are idempotent in nature.


### Abnormal Thread Termination
* In single threaded env when there is some uncaught exception, the stack trace goes to main and the program terminates.
* In case of multithreaded env, when a uncaught exception is thrown by some Thread-5, it silently logs exception and that thread dies silently. The Program keeps on running for other user threads.
* Use `UncaughtExceptionHandler` if using `.execute` which is fire and forget and do not return anything. You need to wrap this within `ThreadFactory`.
* `.submit` completely skips `UncaughtExceptionHandler`, so with this use `Future.get()` to handle the exceptions. If you never call `Future.get` you'll never see that exception.
* In Modern java completable future and Virtual Thread provides clean APIs to handle the exceptions.

### JVM Shutdown

* Shutdown Hook is simply a thread thread that we register and JVM starts this thread when it's shutting down.
* Hooks don't run on abrupt shutdown, like doing `kill -9`.
* If multiple hooks are defined, they all run concurrenlty, so chances of race condition. Better to use just one final single hook for overall application.
* If a hook hangs JVM hangs, so never use infinite loop there.




```java
public class WithShutdownHook {

    public static void main(String[] args) throws InterruptedException {

        // Register the hook FIRST, before doing any work
        // This is just a Thread — but it's NOT started yet
        // JVM will start it automatically during shutdown
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("[HOOK] JVM shutting down — cleaning up...");
            System.out.println("[HOOK] Closing DB connections...");
            System.out.println("[HOOK] Deleting temp files...");
            System.out.println("[HOOK] Cleanup done.");
        }));

        System.out.println("App started, doing work...");
        
        // Simulate long-running work
        // Hit Ctrl+C anytime — the hook will STILL run
        for (int i = 0; i < 10; i++) {
            System.out.println("Working... " + i);
            Thread.sleep(1000);
        }

        System.out.println("App finished normally.");
    }
}
```
#### Daemon Threads

* Use them for background task, never do IO task with them.
* JVM don't respect these threads.
* It Can be used for background cache cleaning. Java's GC runs on daemon thread.
```text
Normal thread exits:     finally blocks run ✅
                         stack unwinds      ✅
                         resources closed   ✅ (if coded properly)

Daemon thread killed:    finally blocks     ❌ SKIPPED
                         stack unwinds      ❌ SKIPPED
                         resources closed   ❌ SKIPPED
```