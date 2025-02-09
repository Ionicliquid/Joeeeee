# JVM
## 四大引用
1. 强引用
2. 软引用：内存不足gc时 才会被回收的应用；
3. 弱引用：gc时就会被回收的引用；
4. 虚引用
## gc roots


# 线程池
## 任务的执行流程

#### execute
```java
public void execute(Runnable command) {  
    if (command == null)  
        throw new NullPointerException();  
	 int c = ctl.get();  
    if (workerCountOf(c) < corePoolSize) {  //1
        if (addWorker(command, true))  
            return;  
        c = ctl.get();  
    }  
    if (isRunning(c) && workQueue.offer(command)) {  
        int recheck = ctl.get();  
        if (! isRunning(recheck) && remove(command))  
            reject(command);  
        else if (workerCountOf(recheck) == 0)  
            addWorker(null, false);  
    }  
    else if (!addWorker(command, false))  
        reject(command);  
}
```
源码中也有对流程的注释，执行过程分为三种情况。
1. 运行线程数小于核心线程池，直接新建线程执行；
2. 尝试加入到阻塞队列，如果成功，测
3. 直接加入到任务队列，
#### addWorker：检查线程池状态 创建并启动线程；
``` java
private boolean addWorker(Runnable firstTask, boolean core) {  
    retry:  
    for (int c = ctl.get();;) {  
       // 线程状态state != RUNNING, 
        if (runStateAtLeast(c, SHUTDOWN)  
            && (runStateAtLeast(c, STOP)  // state>=STOP
                || firstTask != null  
                || workQueue.isEmpty()))  
            return false;  
  
        for (;;) {  
            if (workerCountOf(c)  
                >= ((core ? corePoolSize : maximumPoolSize) & COUNT_MASK))  //如果为核心线程，不大于核心线程数，非核心线程不大于最大线程数
                return false;  
            if (compareAndIncrementWorkerCount(c))  // CAS 操作增加有效线程数
                break retry;  
            c = ctl.get();  // Re-read ctl  
            if (runStateAtLeast(c, SHUTDOWN))  
                continue retry;  
            // else CAS failed due to workerCount change; retry inner loop  
        }  
    }  
  
    boolean workerStarted = false;  
    boolean workerAdded = false;  
    Worker w = null;  
    try {  
        w = new Worker(firstTask);  // 创建线程
        final Thread t = w.thread;  
        if (t != null) {  
            final ReentrantLock mainLock = this.mainLock;  
            mainLock.lock();  
            try {  
                int c = ctl.get();  
  
                if (isRunning(c) ||  
                    (runStateLessThan(c, STOP) && firstTask == null)) {  
                    if (t.getState() != Thread.State.NEW)  
                        throw new IllegalThreadStateException();  
                    workers.add(w);  
                    workerAdded = true;  
                    int s = workers.size();  
                    if (s > largestPoolSize)  
                        largestPoolSize = s;  
                }  
            } finally {  
                mainLock.unlock();  
            }  
            if (workerAdded) {  
                container.start(t);  //启动线程
                workerStarted = true;  
            }  
        }  
    } finally {  
        if (! workerStarted)  
            addWorkerFailed(w);  
    }  
    return workerStarted;  
}
```
runWorker:执行任务
``` java
final void runWorker(Worker w) {  
    Thread wt = Thread.currentThread();  
    Runnable task = w.firstTask;  
    w.firstTask = null;  
    w.unlock(); // allow interrupts  
    boolean completedAbruptly = true;  
    try {  
        while (task != null || (task = getTask()) != null) {  
            w.lock();  
	         if ((runStateAtLeast(ctl.get(), STOP) ||  
                 (Thread.interrupted() &&  
                  runStateAtLeast(ctl.get(), STOP))) &&  
                !wt.isInterrupted())  
                wt.interrupt();  
            try {  
                beforeExecute(wt, task);  
                try {  
                    task.run();  
                    afterExecute(task, null);  
                } catch (Throwable ex) {  
                    afterExecute(task, ex);  
                    throw ex;  
                }  
            } finally {  
                task = null;  
                w.completedTasks++;  
                w.unlock();  
            }  
        }  
        completedAbruptly = false;  
    } finally {  
        processWorkerExit(w, completedAbruptly);  
    }  
}

private Runnable getTask() {  
    boolean timedOut = false; // Did the last poll() time out?  
  
    for (;;) {  
        int c = ctl.get();  
  
        // Check if queue empty only if necessary.  
        if (runStateAtLeast(c, SHUTDOWN)  
            && (runStateAtLeast(c, STOP) || workQueue.isEmpty())) {  
            decrementWorkerCount();  
            return null;        }  
  
        int wc = workerCountOf(c);  
  
        // Are workers subject to culling?  
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;  // 允许核心线程超时 或者 当前线程数大于核心线程
  
        if ((wc > maximumPoolSize || (timed && timedOut))  
            && (wc > 1 || workQueue.isEmpty())) {  
            if (compareAndDecrementWorkerCount(c))  
                return null;  
            continue;        }  
  
        try {  
            Runnable r = timed ?  
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :  
                workQueue.take();  
            if (r != null)  
                return r;  
            timedOut = true;  
        } catch (InterruptedException retry) {  
            timedOut = false;  
        }  
    }  
}
```
