# 协程上下文：CoroutineContext

## CoroutineContext

``` kotlin
public operator fun <E : Element> get(key: Key<E>): E?
public operator fun plus(context: CoroutineContext): CoroutineContext =
        if (context === EmptyCoroutineContext) this else //1
            context.fold(this) { acc, element ->
                val removed = acc.minusKey(element.key)
                if (removed === EmptyCoroutineContext) element else {
                    val interceptor = removed[ContinuationInterceptor]
                    if (interceptor == null) CombinedContext(removed, element) else {
                        val left = removed.minusKey(ContinuationInterceptor)
                        if (left === EmptyCoroutineContext) CombinedContext(element, interceptor) else
                            CombinedContext(CombinedContext(left, element), interceptor)
                    }
                }
            }
```
1. 重载get运算符，使用角标运算符读取元素会被转换为get运算符的方法调用；
2. 重载plus运算符，执行fold逻辑，假设A+B，acc为A，element的B
   1. B为EmptyCoroutineContext，返回A；
   2. A,B相同类型，即Key相等，acc.minusKey返回EmptyCoroutineContext,plus返回B;
   3. A,B为不同类型，A中不包含ContinuationInterceptor，返回CombinedContext，A为left结点；
   4. A,B为不同类型，A中仅包含ContinuationInterceptor，返回CombinedContext，B为left结点；
   5. A,B为不同类型，A中不仅包含ContinuationInterceptor，返回CombinedContext，取出A中ContinuationInterceptor放在右结点，剩下context和B按照3中规则组合

## Element

``` kotlin
public interface Element : CoroutineContext {
        /**
         * A key of this coroutine context element.
         */
        public val key: Key<*>

        public override operator fun <E : Element> get(key: Key<E>): E? =
            @Suppress("UNCHECKED_CAST")
            if (this.key == key) this as E else null

        public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
            operation(initial, this)

        public override fun minusKey(key: Key<*>): CoroutineContext =
            if (this.key == key) EmptyCoroutineContext else this
    }
```
### key

``` kotlin
public interface ContinuationInterceptor : CoroutineContext.Element {
    public companion object Key : CoroutineContext.Key<ContinuationInterceptor>
}

public data class CoroutineName(
    
    val name: String
) : AbstractCoroutineContextElement(CoroutineName) {
    public companion object Key : CoroutineContext.Key<CoroutineName>
}

public interface Job : CoroutineContext.Element {
    public companion object Key : CoroutineContext.Key<Job>
}

public interface CoroutineExceptionHandler : CoroutineContext.Element {
    public companion object Key : CoroutineContext.Key<CoroutineExceptionHandler>
}

```
1. 每个Element都有唯一Key,用来从CoroutineContext中检索Element;
2. 协程库中常见的4种实现CoroutineName\Job\ContinuationInterceptor\CoroutineExceptionHandler，都是通过将Key指定为对应的伴生对象，这属于Kotlin语法糖，可以直接通过类名来进行访问对应的Element;
   1. Job：协程的唯一标识，用来控制协程的生命周期(new、active、completing、completed、cancelling、cancelled)；
   2. CoroutineDispatcher：指定协程运行的线程(IO、Default、Main、Unconfined);
   3. CoroutineName: 指定协程的名称，默认为coroutine;
   4. CoroutineExceptionHandler: 指定协程的异常处理器，用来处理未捕获的异常.
## CombinedContext
```kotlin
 override fun <E : Element> get(key: Key<E>): E? {
        var cur = this
        while (true) {
            cur.element[key]?.let { return it }
            val next = cur.left
            if (next is CombinedContext) {
                cur = next
            } else {
                return next[key]
            }
        }
    }
	
public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =
        operation(left.fold(initial, operation), element)

public override fun minusKey(key: Key<*>): CoroutineContext {
        element[key]?.let { return left }
        val newLeft = left.minusKey(key)
        return when {
            newLeft === left -> this
            newLeft === EmptyCoroutineContext -> element
            else -> CombinedContext(newLeft, element)
        }
    }
```
1. CombinedContext使用链表结构保存context,left <- element;
2. get:遍历链表,首先访问right结点element,返回指定key元素；
3. fold:从left开始执行，参考CoroutineContext#plus中的fold逻辑；
4. minuskey:从element开始执行，去掉相同的key；

# 协程作用域：CoroutineScope

1. 什么是结构化并发？

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}

public fun CoroutineScope(context: CoroutineContext): CoroutineScope =
    ContextScope(if (context[Job] != null) context else context + Job())

```

1. 在通过CoroutineScope.launch创建协程时，返回了一个Job, 并将scope.coroutineContext与参数context进行组合后，赋值给coroutine；

# 协程分发器：Dispatcher

# lunch流程分析

``` kotlin
fun main() {
    GlobalScope.launch {
        val context = coroutineContext
        println("context $context")
        getName()
    }
    Thread.sleep(2000)
}

suspend fun getName(): String {
    delay(1000)
    return "Joe"
}
```

对象生成的Java代码

``` java
public static final void main() {
    // launch
      BuildersKt.launch$default((CoroutineScope)GlobalScope.INSTANCE, (CoroutineContext)null, (CoroutineStart)null, (Function2)(new Function2((Continuation)null) {
         // $FF: synthetic field
         private Object L$0;
         int label;

         @Nullable
         public final Object invokeSuspend(@NotNull Object $result) {
            Object var5 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
            switch (this.label) {
               case 0:
                  ResultKt.throwOnFailure($result);
                  CoroutineScope $this$launch = (CoroutineScope)this.L$0;
                  CoroutineContext context = $this$launch.getCoroutineContext();
                  String var4 = "context " + context;
                  System.out.println(var4);
                  this.label = 1;
                  if (MyCoroutineKt.getName(this) == var5) {
                     return var5;
                  }
                  break;
               case 1:
                  ResultKt.throwOnFailure($result);
                  break;
               default:
                  throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
            }

            return Unit.INSTANCE;
         }

         @NotNull
         public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
            Intrinsics.checkNotNullParameter(completion, "completion");
            Function2 var3 = new <anonymous constructor>(completion);
            var3.L$0 = value;
            return var3;
         }

         public final Object invoke(Object var1, Object var2) {
            return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
         }
      }), 3, (Object)null);
      Thread.sleep(2000L);
   }
// getName()
public static final Object getName(@NotNull Continuation var0) {
      Object $continuation;
      label20: {
         if (var0 instanceof <undefinedtype>) {
            $continuation = (<undefinedtype>)var0;
            if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
               ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
               break label20;
            }
         }

         $continuation = new ContinuationImpl(var0) {
            Object result;
            int label;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) {
               this.result = $result;
               this.label |= Integer.MIN_VALUE;
               return MyCoroutineKt.getName(this);
            }
         };
      }

      Object $result = ((<undefinedtype>)$continuation).result;
      Object var3 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch (((<undefinedtype>)$continuation).label) {
         case 0:
            ResultKt.throwOnFailure($result);
            ((<undefinedtype>)$continuation).label = 1;
            if (DelayKt.delay(1000L, (Continuation)$continuation) == var3) {
               return var3;
            }
            break;
         case 1:
            ResultKt.throwOnFailure($result);
            break;
         default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }

      return "Joe";
   }

```

这里涉及2个挂起函数：

1. launch中作为参数传入的CoroutineScope的扩展挂起函数，对应在Java中是SuspendLambda和Function2的实现类对象，将这个对象称为block；
2. getName 自定义的挂起函数，其中的匿名续体为ContinuationImpl子类；

具体的实现细节参考挂起函数的介绍

## GlobaScope#launch

```kotlin
public fun CoroutineScope.launch(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> Unit
): Job {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyStandaloneCoroutine(newContext, block) else
        StandaloneCoroutine(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}
//AbstractCoroutine
public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
        start(block, receiver, this) //1
    }
// CoroutineStart
public operator fun <R, T> invoke(block: suspend R.() -> T, receiver: R, completion: Continuation<T>): Unit =
        when (this) {
            DEFAULT -> block.startCoroutineCancellable(receiver, completion)
            ATOMIC -> block.startCoroutine(receiver, completion)
            UNDISPATCHED -> block.startCoroutineUndispatched(receiver, completion)
            LAZY -> Unit 
        }

internal fun <R, T> (suspend (R) -> T).startCoroutineCancellable(
    receiver: R, completion: Continuation<T>,
) = runSafely(completion) {
    createCoroutineUnintercepted(receiver, completion).intercepted().resumeCancellableWith(Result.success(Unit))
}
//kotlin/libraries/stdlib/jvm/src/kotlin/coroutines/intrinsics/IntrinsicsJvm.kt
public actual fun <R, T> (suspend R.() -> T).createCoroutineUnintercepted( 
    receiver: R,
    completion: Continuation<T>
): Continuation<Unit> {
    val probeCompletion = probeCoroutineCreated(completion)
    return if (this is BaseContinuationImpl)
        create(receiver, probeCompletion) 
    else {
        createCoroutineFromSuspendFunction(probeCompletion) {
            (this as Function2<R, Continuation<T>, Any?>).invoke(receiver, it)
        }
    }
}
//ContinuationImpl
public fun intercepted(): Continuation<Any?> =
        intercepted
            ?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)
                .also { intercepted = it }
// CoroutineDispatcher
public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =
        DispatchedContinuation(this, continuation)
```

1. CoroutineStart重载了操作符invoke，可以直接通过对象名调用该方法；
2. startCoroutineCancellable：
   1. createCoroutineUnintercepted： 调用block对象的create方法创建并返回一个新的Block类型的Continuation；
   2. intercepted：此时的context为completion的context,也就是StandaloneCoroutine的context,默认会包含Dispatchers.Default 其key为ContinuationInterceptor，调用interceptContinuation时返回DispatchedContinuation
3. resumeCancellableWith： 执行DispatchedContinuation.resumeCancellableWith

## DispatchedContinuation#resumeCancellableWith

```kotlin
internal inline fun resumeCancellableWith(result: Result<T>) {
        val state = result.toState() //1
        if (dispatcher.isDispatchNeeded(context)) { //2
            _state = state 
            resumeMode = MODE_CANCELLABLE //3
            dispatcher.dispatch(context, this)
        } else {
            executeUnconfined(state, MODE_CANCELLABLE) {
                if (!resumeCancelled(state)) {
                    resumeUndispatchedWith(result)
                }
            }
        }
    }
```

1. toState：result为Result.success(Unit)，其value为非Failure类型，直接返回value;
2. isDispatchNeeded 默认为true, 
3. 将resumeMode 设置为MODE_CANCELLABLE，

## SchedulerCoroutineDispatcher#dispatch

```kotlin
//SchedulerCoroutineDispatcher
override fun dispatch(context: CoroutineContext, block: Runnable): Unit = coroutineScheduler.dispatch(block)
//CoroutineScheduler
fun dispatch(block: Runnable, taskContext: TaskContext = NonBlockingContext, tailDispatch: Boolean = false) {
        trackTask()
        val task = createTask(block, taskContext) //1
        val isBlockingTask = task.isBlocking 
        val stateSnapshot = if (isBlockingTask) incrementBlockingTasks() else 0
        val currentWorker = currentWorker() //2
        val notAdded = currentWorker.submitToLocalQueue(task, tailDispatch)
        if (notAdded != null) { 
            if (!addToGlobalQueue(notAdded)) {
                throw RejectedExecutionException("$schedulerName was terminated")
            }
        }
        val skipUnpark = tailDispatch && currentWorker != null //4
        if (isBlockingTask) {
            signalBlockingWork(stateSnapshot, skipUnpark = skipUnpark)
        } else {
            if (skipUnpark) return
            signalCpuWork()
        }
    }

private fun addToGlobalQueue(task: Task): Boolean {
        return if (task.isBlocking) {
            globalBlockingQueue.addLast(task)
        } else {
            globalCpuQueue.addLast(task)
        }
    }

internal const val NonBlockingContext: TaskContext = false

internal const val BlockingContext: TaskContext = true
```

1. 将Runnable封装为Task，taskContext默认NonBlockingContex，task.isBlocking == false;
2. 首次执行currentWorker为null, notAdded为当前task，不为空;
   1. currentWorker：当前线程是否为Worker类型；
   2. addToGlobalQueue：将task加入到globalCpuQueue中;
3. skipUnpark == false

## CoroutineScheduler#signalCpuWork

```kotlin
fun signalCpuWork() {
        if (tryUnpark()) return
        if (tryCreateWorker()) return
        tryUnpark()
    }

private fun tryUnpark(): Boolean {
        while (true) {
            val worker = parkedWorkersStackPop() ?: return false
            if (worker.workerCtl.compareAndSet(PARKED, CLAIMED)) {
                LockSupport.unpark(worker)
                return true
            }
        }
    }
private fun parkedWorkersStackPop(): Worker? {
        parkedWorkersStack.loop { top -> //1
            val index = (top and PARKED_INDEX_MASK).toInt()
            val worker = workers[index] ?: return null //2
            val updVersion = (top + PARKED_VERSION_INC) and PARKED_VERSION_MASK
            val updIndex = parkedWorkersStackNextIndex(worker)
            if (updIndex < 0) return@loop 
            if (parkedWorkersStack.compareAndSet(top, updVersion or updIndex.toLong())) {
                worker.nextParkedWorker = NOT_IN_STACK
                return worker
            }
        }
    }

public inline fun <T> AtomicRef<T>.loop(action: (T) -> Unit): Nothing {
    while (true) {
        action(value)
    }
}

private fun parkedWorkersStackNextIndex(worker: Worker): Int {
        var next = worker.nextParkedWorker
        findNext@ while (true) {
            when {
                next === NOT_IN_STACK -> return -1 
                next === null -> return 0 
                else -> {
                    val nextWorker = next as Worker
                    val updIndex = nextWorker.indexInArray
                    if (updIndex != 0) return updIndex 

                    next = nextWorker.nextParkedWorker
                }
            }
        }
    }

private fun tryCreateWorker(state: Long = controlState.value): Boolean {
        val created = createdWorkers(state) //1
        val blocking = blockingTasks(state)
        val cpuWorkers = (created - blocking).coerceAtLeast(0)
        if (cpuWorkers < corePoolSize) { //2
            val newCpuWorkers = createNewWorker()
            if (newCpuWorkers == 1 && corePoolSize > 1) createNewWorker() //3
            if (newCpuWorkers > 0) return true
        }
        return false
    }
private fun createNewWorker(): Int {
        val worker: Worker
        return synchronized(workers) 
            if (isTerminated) return -1
            val state = controlState.value
            val created = createdWorkers(state)
            val blocking = blockingTasks(state)
            val cpuWorkers = (created - blocking).coerceAtLeast(0)
            if (cpuWorkers >= corePoolSize) return 0
            if (created >= maxPoolSize) return 0
            val newIndex = createdWorkers + 1
            require(newIndex > 0 && workers[newIndex] == null)
            worker = Worker(newIndex)
            workers.setSynchronized(newIndex, worker) //1
            require(newIndex == incrementCreatedWorkers())
            cpuWorkers + 1
        }.also { worker.start() } //2
    }
```

1. tryUnpark
   1. parkedWorkersStackPop:
      1. parkedWorkersStack 为原子变量，loop方法循环执行，直到action函数对象返回；
      2. index索引对应worker为null，直接返回；
      3. parkedWorkersStackNextIndex
2. tryCreateWorker
   1. 采用一个64位的Long类型来描述线程池中works的状态
      1. controlState: 高22位，描述CPU核心数，默认值为核心线程数左移42位；
      2. blockingTasks：中21位，描述正在工作的work数（state & 中21位1）；
      3. createdWorkers：低21位，描述已创建的work数 (state & 低21位1)；
   2. 首次启动created 和 blocking 都为0；
   3. 首次启动创建了2个线程？
3. createNewWorker
   1. 创建成功，将其加入的CoroutineScheduler的workers数组中；
   2. 启动线程；

## Worker#start

work为线程子类，start时，直接执行run方法；

```kotlin
override fun run() = runWorker()

private fun runWorker() {
            var rescanned = false
            while (!isTerminated && state != WorkerState.TERMINATED) {
                val task = findTask(mayHaveLocalTasks) //1
                if (task != null) {
                    rescanned = false
                    minDelayUntilStealableTaskNs = 0L
                    executeTask(task)
                    continue
                } else {
                    mayHaveLocalTasks = false
                }
                if (minDelayUntilStealableTaskNs != 0L) {
                    if (!rescanned) {
                        rescanned = true
                    } else {
                        rescanned = false
                        tryReleaseCpu(WorkerState.PARKING)
                        interrupted()
                        LockSupport.parkNanos(minDelayUntilStealableTaskNs)
                        minDelayUntilStealableTaskNs = 0L
                    }
                    continue
                }
                tryPark()
            }
            tryReleaseCpu(WorkerState.TERMINATED)
        }

fun findTask(mayHaveLocalTasks: Boolean): Task? {
            if (tryAcquireCpuPermit()) return findAnyTask(mayHaveLocalTasks)
            return findBlockingTask()
        }

private fun tryAcquireCpuPermit(): Boolean = when {
            state == WorkerState.CPU_ACQUIRED -> true //1
            this@CoroutineScheduler.tryAcquireCpuPermit() -> {
                state = WorkerState.CPU_ACQUIRED
                true
            }

            else -> false
        }
// CoroutineScheduler.tryAcquireCpuPermit
 private inline fun tryAcquireCpuPermit(): Boolean = controlState.loop { state ->
        val available = availableCpuPermits(state)
        if (available == 0) return false
        val update = state - (1L shl CPU_PERMITS_SHIFT)
        if (controlState.compareAndSet(state, update)) return true
    }

private fun findAnyTask(scanLocalQueue: Boolean): Task? {
            if (scanLocalQueue) {
                val globalFirst = nextInt(2 * corePoolSize) == 0
                if (globalFirst) pollGlobalQueues()?.let { return it }
                localQueue.poll()?.let { return it }
                if (!globalFirst) pollGlobalQueues()?.let { return it }
            } else {
                pollGlobalQueues()?.let { return it }
            }
            return trySteal(STEAL_ANY)
        }
```

1. findTask:查找可执行的task
   1. tryAcquireCpuPermit：获取CPU许可，允许创建的最大核心数，不为0，返回true，并自减1；
   2. findAnyTask：scanLocalQueue为false，优先从globalCpuQueue，再从globalCpuQueue中获取;
   3. trySteal?:
2. 

##  DispatchedTask#run

```kotlin
final override fun run() {
        assert { resumeMode != MODE_UNINITIALIZED } 
        var fatalException: Throwable? = null
        try {
            val delegate = delegate as DispatchedContinuation<T>
            val continuation = delegate.continuation //1
            withContinuationContext(continuation, delegate.countOrElement) {
                val context = continuation.context //2
                val state = takeState() 
                val exception = getExceptionalResult(state) //3
                val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null //4
                if (job != null && !job.isActive) {
                    val cause = job.getCancellationException()
                    cancelCompletedResult(state, cause)
                    continuation.resumeWithStackTrace(cause)
                } else {
                    if (exception != null) {
                        continuation.resumeWithException(exception)
                    } else {
                        continuation.resume(getSuccessfulResult(state)) //2
                    }
                }
            }
        } catch (e: Throwable) {
            // This instead of runCatching to have nicer stacktrace and debug experience
            fatalException = e
        } finally {
            fatalException?.let { handleFatalException(it) }
        }
    }
```

1. continuation 是lunch中block函数对象，也就是BaseContinuationImpl的子类；
2. context为 StandaloneCoroutine的context 类型为Dispatchers.Default + StandaloneCoroutine;
3. 正常场景state 为result的value,在resumeCancellableWith进行赋值，exception== null;
4. resumeMode也在resumeCancellableWith进行赋值，job为StandaloneCoroutine，job.isActive == true
5. 执行continuation.resume方法；

## BaseContinuationImpl#resumeWith

```kotlin
public inline fun <T> Continuation<T>.resume(value: T): Unit =
    resumeWith(Result.success(value))
    
public final override fun resumeWith(result: Result<Any?>) {
	    var current = this
        var param = result
        while (true) {
            probeCoroutineResumed(current)
            with(current) {
                val completion = completion!! 
                val outcome: Result<Any?> =
                    try {
                        val outcome = invokeSuspend(param)
                        if (outcome === COROUTINE_SUSPENDED) return
                        Result.success(outcome)
                    } catch (exception: Throwable) {
                        Result.failure(exception)
                    }
                releaseIntercepted() 
                if (completion is BaseContinuationImpl) {
                    current = completion
                    param = outcome
                } else {
                    completion.resumeWith(outcome)
                    return
                }
            }
        }
    }
    
```

1. 执行block的invokeSuspend方法

# 挂起函数

1. 挂起函数只能在挂起函数内部或者协程体内调用? 什么协程体呢？
2. callback方式如何实现挂起？
  1. 将需要在主线程的执行的业务封装成callback，子线程执行完成之后，切换线程调用这个callback，
  2. 常见的方式Thread.join(),handler.post，都可以实现在子线程业务完成之后恢复到指定线程执行。
     1. join会阻塞主线程；
     2. handler利用消息队列机制，逻辑复杂；
## 普通挂起函数
``` kotlin
suspend fun getName():String{
    delay(100)
    return "Joe"
}
```
一个普通的挂起函数，将其转换为对应的java代码，
``` java
 public static final Object getName(@NotNull Continuation var0) {
      Object $continuation;
      label20: {
         if (var0 instanceof <undefinedtype>) {
            $continuation = (<undefinedtype>)var0;
            if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
               ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
               break label20;
            }
         }

         $continuation = new ContinuationImpl(var0) {
            // $FF: synthetic field
            Object result;
            int label;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) {
               this.result = $result;
               this.label |= Integer.MIN_VALUE;
               return CoroutinesDemo1Kt.getName(this);
            }
         };
      }

      Object $result = ((<undefinedtype>)$continuation).result;
      Object var3 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch (((<undefinedtype>)$continuation).label) {
         case 0:
            ResultKt.throwOnFailure($result);
            ((<undefinedtype>)$continuation).label = 1;
            if (DelayKt.delay(100L, (Continuation)$continuation) == var3) {
               return var3;
            }
            break;
         case 1:
            ResultKt.throwOnFailure($result);
            break;
         default:
            throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
      }

      return "Joe";
   }
```
转换为易读的形式
```java
public static final Object getName(@NotNull Continuation completion) { //1
        Name name;
        label20: {
            if (completion instanceof Name) {
                name = (Name)completion;
                if ((name.label & Integer.MIN_VALUE) != 0) {
                    name.label -= Integer.MIN_VALUE;
                    break label20;
                }
            }
            name = new Name(completion); //2
        }

        Object $result = name.result;
        Object var3 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
        switch (name.label) { //3
            case 0:
                ResultKt.throwOnFailure($result);
                name.label = 1;
                if (DelayKt.delay(100L, name) == var3) { 
                    return var3;
                }
                break;
            case 1: //4
                ResultKt.throwOnFailure($result);
                break;
            default:
                throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
        }

        return "Joe";
    }
	
static class Name extends ContinuationImpl{

        Object result;
        int label;

        public Name(@Nullable Continuation<Object> completion) {
            super(completion);
        }

        @androidx.annotation.Nullable
        @Override
        protected Object invokeSuspend(@NonNull Object result) {
            this.result = result;
            this.label |= Integer.MIN_VALUE;
            return CoroutinesDemo1Kt.getName(this);
        }
    }
```
1. 经过编译后，为挂起函数添加Continuation类型的参数，称为续体；
2. 创建匿名续体对象Name，将新增的续体参数保存在属性completion，从它的名称就可知道它的作用就是用来执行完挂起函数后恢复协程的执行。
3. 当函数被调用时label为0，将name对象传递挂起函数delay，label+1，自身挂起；
4. 由delay函数内部恢复函数执行，调用name的invokeSuspend方法，执行label为1时的业务；

## 扩展函数的挂起函数

## 总结

### CPS转换

1. 挂起函数经过编译后，会添加一个Continuation的参数，称为续体，作为属性保存在挂起函数内存创建的匿名续体实现对象中，属性名为completion，从这个名字就可知，它的作用于callback类似，在挂起函数执行完成之后用来恢复协程的执行；

### 状态机

1. 挂起函数中每个挂起点都对应一个状态，用int值表示，如delay函数调用就是一个挂起点，状态数量为挂起点+1；
2. 执行delay前，将状态+1，delay执行完，由delay控制，进入下个状态的挂起点执行；

## 续体

# Job

## Job的生命周期

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7743d8628b64065b3d8248f26e01350~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

## AbstractCoroutine#init

```kotlin
//AbstractCoroutine  
init {
        if (initParentJob) initParentJob(parentContext[Job]) //1 
    }

```

1.  lunch启动协程时,初始化AbstractCoroutine，initParentJob默认为true;

## JobSupport#initParentJob

``` kotlin
// JobSupport 141
protected fun initParentJob(parent: Job?) {
        assert { parentHandle == null }
        if (parent == null) {
            parentHandle = NonDisposableHandle
            return
        }
        parent.start() //1
        val handle = parent.attachChild(this) //2
        parentHandle = handle //3
        if (isCompleted) {
            handle.dispose()
            parentHandle = NonDisposableHandle 
        }
    }
// JobSupport 376
public final override fun start(): Boolean {
        loopOnState { state ->
            when (startInternal(state)) {
                FALSE -> return false
                TRUE -> return true
            }
        }
    }

// JobSupport 389
private fun startInternal(state: Any?): Int {
        when (state) {
                if (state.isActive) return FALSE 
                if (!_state.compareAndSet(state, EMPTY_ACTIVE)) return RETRY
                onStart()
                return TRUE
            }
            is InactiveNodeList -> { 
                if (!_state.compareAndSet(state, state.list)) return RETRY
                onStart()
                return TRUE
            }
            else -> return FALSE // not a new state
        }
    }
```

1. start：父Job不为空，启动父Job， 将state设置为 EMPTY_ACTIVE 或 NodeList；
2. attachChild：将父子Job进行关联；

## JobSupport#attachChild

``` kotlin
// JobSupport 1011
public final override fun attachChild(child: ChildJob): ChildHandle {
        val node = ChildHandleNode(child).also { it.job = this } //1
        val added = tryPutNodeIntoList(node) { _, list -> //2
            val addedBeforeCancellation = list.addLast(
                node,
                LIST_ON_COMPLETION_PERMISSION or LIST_CHILD_PERMISSION or LIST_CANCELLATION_PERMISSION
            )
            if (addedBeforeCancellation) {
                true
            } else {
                val addedBeforeCompletion = list.addLast(
                    node,
                    LIST_CHILD_PERMISSION or LIST_ON_COMPLETION_PERMISSION
                )
                val rootCause = when (val latestState = this.state) {
                    is Finishing -> {
					  latestState.rootCause
                    }
                    else -> {
                        assert { latestState !is Incomplete }
                        (latestState as? CompletedExceptionally)?.cause
                    }
                }
                node.invoke(rootCause)
                if (addedBeforeCompletion) {
                    assert { rootCause != null }
                    true
                } else {
                    return NonDisposableHandle
                }
            }
        }
        if (added) return node
        node.invoke((state as? CompletedExceptionally)?.cause)
        return NonDisposableHandle
    }
// JobSupport 125
private val _state = atomic<Any?>(if (active) EMPTY_ACTIVE else EMPTY_NEW)

// JobSupport 532
    private inline fun tryPutNodeIntoList(
        node: JobNode,
        tryAdd: (Incomplete, NodeList) -> Boolean
    ): Boolean {
        loopOnState { state ->
            when (state) {
                is Empty -> { //1
                    if (state.isActive) { //2
                        if (_state.compareAndSet(state, node)) return true
                    } else
                        promoteEmptyToNodeList(state) 
                }
                is Incomplete -> when (val list = state.list) {
                    null -> promoteSingleToNodeList(state as JobNode) //3
                    else -> if (tryAdd(state, list)) return true
                }
                else -> return false
            }
        }
    }
// JobSupport 554
 private fun promoteEmptyToNodeList(state: Empty) {
        val list = NodeList()
        val update = if (state.isActive) list else InactiveNodeList(list)
        _state.compareAndSet(state, update)
    }
// JobSupport 561
private fun promoteSingleToNodeList(state: JobNode) {
	state.addOneIfEmpty(NodeList())
	val list = state.nextNode
	 _state.compareAndSet(state, list)
   }
```

1. 将childJob封装为对应ChildHandleNode，其job和parent 成员指向parent；
2. tryPutNodeIntoList
   1. 首次调用state 默认为Empty, 根据isActive不同，对应2种状态EMPTY_ACTIVE else EMPTY_NEW
   2. 如果isActive == true, 将parent状态设置为ChildHandleNode，否则调用promoteEmptyToNodeList将状态设置为InactiveNodeList；
   3. 再次加入子Job,state为Incomplete 
      1. 如state为ChildHandleNode（JobNode.list == null），创建NodeList，并将状态设置为NodeList；
3. tryAdd

## JobSupport#cancle

``` kotlin
public override fun cancel(cause: CancellationException?) {
        cancelInternal(cause ?: defaultCancellationException())
    }

public open fun cancelInternal(cause: Throwable) {
        cancelImpl(cause)
    }

// JobSupport 693
internal fun cancelImpl(cause: Any?): Boolean {
        var finalState: Any? = COMPLETING_ALREADY
        if (onCancelComplete) {
            finalState = cancelMakeCompleting(cause)
            if (finalState === COMPLETING_WAITING_CHILDREN) return true
        }
        if (finalState === COMPLETING_ALREADY) {
            finalState = makeCancelling(cause)
        }
        return when {
            finalState === COMPLETING_ALREADY -> true
            finalState === COMPLETING_WAITING_CHILDREN -> true
            finalState === TOO_LATE_TO_CANCEL -> false
            else -> {
                afterCompletion(finalState)
                true
            }
        }
    }

```

1. onCancelComplete 默认为false，JobImpl  和 CompletableDeferredImpl重写为false

### JobSupport#makeCancelling

``` kotlin
// JobSupport 761
private fun makeCancelling(cause: Any?): Any? {
        var causeExceptionCache: Throwable? = null 
        loopOnState { state ->
            when (state) {
                is Finishing -> {
                    val notifyRootCause = synchronized(state) {
                        if (state.isSealed) return TOO_LATE_TO_CANCEL
                        val wasCancelling = state.isCancelling 
                        if (cause != null || !wasCancelling) {
                            val causeException = causeExceptionCache ?: createCauseException(cause).also { causeExceptionCache = it }
                            state.addExceptionLocked(causeException)
                        }
                        state.rootCause.takeIf { !wasCancelling }
                    }
                    notifyRootCause?.let { notifyCancelling(state.list, it) }
                    return COMPLETING_ALREADY
                }
                is Incomplete -> {
                    val causeException = causeExceptionCache ?: createCauseException(cause).also { causeExceptionCache = it }
                    if (state.isActive) {
                        if (tryMakeCancelling(state, causeException)) return COMPLETING_ALREADY //1
                    } else {
                      val finalState = tryMakeCompleting(state, CompletedExceptionally(causeException))
                        when {
                            finalState === COMPLETING_ALREADY -> error("Cannot happen in $state")
                            finalState === COMPLETING_RETRY -> return@loopOnState
                            else -> return finalState
                        }
                    }
                }
                else -> return TOO_LATE_TO_CANCEL 
            }
        }
    }
// JobSupport 817
private fun tryMakeCancelling(state: Incomplete, rootCause: Throwable): Boolean {
        assert { state !is Finishing } 
        assert { state.isActive } 
        val list = getOrPromoteCancellingList(state) ?: return false //1
        val cancelling = Finishing(list, false, rootCause)
        if (!_state.compareAndSet(state, cancelling)) return false //2
        notifyCancelling(list, rootCause)
        return true
    }

private fun notifyCancelling(list: NodeList, cause: Throwable) {
        onCancelling(cause) //1
        list.close(LIST_CANCELLATION_PERMISSION)
        notifyHandlers(list, cause) { it.onCancelling } //2
        cancelParent(cause) //3
    }
// JobSupport 360
private inline fun notifyHandlers(list: NodeList, cause: Throwable?, predicate: (JobNode) -> Boolean) {
        var exception: Throwable? = null
        list.forEach { node ->
            if (node is JobNode && predicate(node)) {
                try {
                    node.invoke(cause) //1
                } catch (ex: Throwable) {
                    exception?.apply { addSuppressed(ex) } ?: run {
                        exception = CompletionHandlerException("Exception in completion handler $node for $this", ex)
                    }
                }
            }
        }
        exception?.let { handleOnCompletionException(it) }
    }
// JobSupport 336
private fun cancelParent(cause: Throwable): Boolean {
        if (isScopedCoroutine) return true
        val isCancellation = cause is CancellationException
        val parent = parentHandle
        if (parent === null || parent === NonDisposableHandle) {
            return isCancellation
        }
        return parent.childCancelled(cause) || isCancellation //1
    }

// ChildHandleNode 1575
override fun invoke(cause: Throwable?) = childJob.parentCancelled(job)
// ChildHandleNode 1581
override fun childCancelled(cause: Throwable): Boolean = job.childCancelled(cause)
```

1. tryMakeCancelling
   1.  getOrPromoteCancellingList：返回state对应的list，为空则新建；
   2. 将state 设置为Finishing
2. notifyCancelling
   1. onCancelling 默认为空实现；
   2. notifyHandlers：通知所有子Job，父Job正在取消，子Job为ChildHandleNode， onCancelling为true;
      1. 执行其invoke方法，invoke -> JobSupport.parentCancelled -> JobSupport.cancelImpl;
   3. cancelParent：通知父Job，子Job正在取消，子Job为ChildHandleNode，其job属性和parent属性一样指向父job
      1. ChildHandleNode.childCancelled -> JobSupport.childCancelled -> JobSupport.cancelImpl；

## JobSupport#makeCompletingOnce

``` kotlin
//BaseContinuationImpl
public final override fun resumeWith(result: Result<Any?>) {
		var current = this
        var param = result
        while (true) {
            //......
            with(current) {
                val completion = completion!! 
                val outcome: Result<Any?> =
                    try {
       					//......
                        Result.success(outcome)
                    } catch (exception: Throwable) {
                        Result.failure(exception)
                    }
       
                if (completion is BaseContinuationImpl) {
				//......
                } else {
                   	//1
                    completion.resumeWith(outcome)
                    return
                }
            }
        }
    }

// AbstractCoroutine
public final override fun resumeWith(result: Result<T>) {
        val state = makeCompletingOnce(result.toState())
        if (state === COMPLETING_WAITING_CHILDREN) return
        afterResume(state)
    }
```

1. CoroutineScope.launch的block函数参数为CoroutineScope的扩展函数，传入的scope为AbstractCoroutine，当协程体内业务处理完成，执行completion也就是AbstractCoroutine的resumeWith方法。

### JobSupport#makeCompletingOnce

``` kotlin
// JobSupport 857
internal fun makeCompletingOnce(proposedUpdate: Any?): Any? {
        loopOnState { state ->
            val finalState = tryMakeCompleting(state, proposedUpdate)
            when {
                finalState === COMPLETING_ALREADY ->
                    throw IllegalStateException(
                        "Job $this is already complete or completing, " +
                            "but is being completed with $proposedUpdate", proposedUpdate.exceptionOrNull
                    )
                finalState === COMPLETING_RETRY -> return@loopOnState
                else -> return finalState 
            }
        }
    }

 private fun tryMakeCompleting(state: Any?, proposedUpdate: Any?): Any? {
        if (state !is Incomplete)
            return COMPLETING_ALREADY
        if ((state is Empty || state is JobNode) && state !is ChildHandleNode && proposedUpdate !is CompletedExceptionally) {
            if (tryFinalizeSimpleState(state, proposedUpdate)) {
                return proposedUpdate
            }
            return COMPLETING_RETRY
        }
        return tryMakeCompletingSlowPath(state, proposedUpdate)
    }
```

### JobSupport#tryMakeCompletingSlowPath

``` kotlin
private fun tryMakeCompletingSlowPath(state: Incomplete, proposedUpdate: Any?): Any? {
    	//1
        val list = getOrPromoteCancellingList(state) ?: return COMPLETING_RETRY 
    	//2与cancel时创建的Finish不同，rootCause为空；
        val finishing = state as? Finishing ?: Finishing(list, false, null)
        val notifyRootCause: Throwable?
        synchronized(finishing) {
            if (finishing.isCompleting) return COMPLETING_ALREADY
		//3 将isCompleting设置为true；
            finishing.isCompleting = true 
            if (finishing !== state) {
                //4 cas操作修改状态，失败重试；
                if (!_state.compareAndSet(state, finishing)) return COMPLETING_RETRY
            }
            assert { !finishing.isSealed } 
            val wasCancelling = finishing.isCancelling 
            (proposedUpdate as? CompletedExceptionally)?.let { finishing.addExceptionLocked(it.cause) }
          notifyRootCause = finishing.rootCause.takeIf { !wasCancelling }
        }
        notifyRootCause?.let { notifyCancelling(list, it) }
        val child = list.nextChild()
        if (child != null && tryWaitForChild(finishing, child, proposedUpdate))
            return COMPLETING_WAITING_CHILDREN
        list.close(LIST_CHILD_PERMISSION)
        val anotherChild = list.nextChild()
        if (anotherChild != null && tryWaitForChild(finishing, anotherChild, proposedUpdate))
            return COMPLETING_WAITING_CHILDREN
        return finalizeFinishingState(finishing, proposedUpdate)
    }
```

# Flow

