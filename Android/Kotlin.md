# kotlin语法糖

运行在JVM上的kotlin可以简单理解为是Java的语法糖，如果不能理解kotlin代码，可以查看对应的Java代码`Tools -> Koltin -> Show Kotlin ByteCode`

## invoke

## inline noinline crossinline

## 扩展函数

## 操作符重载

## 伴生对象

## 函数对象

# 启动一个协程

启动协程的方式有3种

## launch

```kotlin
fun main() {
    GlobalScope.launch{
       val name = fetchData()
        println("name $name")
    }
    println("main end")
    Thread.sleep(2000)
}

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

```

对应的java代码

``` java
public static final void main() {
      BuildersKt.launch$default((CoroutineScope)GlobalScope.INSTANCE, (CoroutineContext)null, (CoroutineStart)null, (Function2)(new Function2((Continuation)null) {
         int label;

         @Nullable
         public final Object invokeSuspend(@NotNull Object $result) {
            Object var4 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
            Object var10000;
            switch (this.label) {
               case 0:
                  ResultKt.throwOnFailure($result);
                  this.label = 1;
                  var10000 = MyCoroutineKt.fetchData(this);
                  if (var10000 == var4) {
                     return var4;
                  }
                  break;
               case 1:
                  ResultKt.throwOnFailure($result);
                  var10000 = $result;
                  break;
               default:
                  throw new IllegalStateException("call to 'resume' before 'invoke' with coroutine");
            }

            String name = (String)var10000;
            String var3 = "name " + name;
            System.out.println(var3);
            return Unit.INSTANCE;
         }

         @NotNull
         public final Continuation create(@Nullable Object value, @NotNull Continuation completion) {
            Intrinsics.checkNotNullParameter(completion, "completion");
            Function2 var3 = new <anonymous constructor>(completion);
            return var3;
         }

         public final Object invoke(Object var1, Object var2) {
            return ((<undefinedtype>)this.create(var1, (Continuation)var2)).invokeSuspend(Unit.INSTANCE);
         }
      }), 3, (Object)null);
      String var0 = "main end";
      System.out.println(var0);
      Thread.sleep(2000L);
   }
```

1. launch是协作作用域（CoroutineScope）的扩展函数，函数中的block参数是CoroutineScope扩展函数同时也是挂起函数，这也是函数体内能调用其他挂起函数的原因；
2. block 同时也可以简化为`suspend (R) -> T` ，又因为挂起函数为会函数添加一个续体参数，所以block是同时实现Function2和挂起接口的函数对象 `final class com/example/myapplication/coroutine/MyCoroutineKt$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2 `；为了方便我们称`MyCoroutineKt$main$1`为`Block`

### startCoroutineCancellable
``` kotlin

//AbstractCoroutine
public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
        start(block, receiver, this)
    }

//CoroutineStart 
public operator fun <R, T> invoke(block: suspend R.() -> T, receiver: R, completion: Continuation<T>): Unit =
        when (this) {
            DEFAULT -> block.startCoroutineCancellable(receiver, completion)
            ATOMIC -> block.startCoroutine(receiver, completion)
            UNDISPATCHED -> block.startCoroutineUndispatched(receiver, completion)
            LAZY -> Unit // will start lazily
        }

internal fun <R, T> (suspend (R) -> T).startCoroutineCancellable(  
receiver: R, completion: Continuation<T>,  
) = runSafely(completion) {  
createCoroutineUnintercepted(receiver, completion).intercepted().resumeCancellableWith(Result.success(Unit))  
}

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
public fun intercepted(): Continuation<Any?> =  
intercepted  
?: (context[ContinuationInterceptor]?.interceptContinuation(this) ?: this)  
.also { intercepted = it }

//CoroutineDispatcher
public final override fun <T> interceptContinuation(continuation: Continuation<T>): Continuation<T> =  
DispatchedContinuation(this, continuation)
```
1. CoroutineStart 重载了invoke函数，
   2. invoke kotlin 语法糖，可以直接通过对象调用该方法;

3. start 默认为CoroutineStart.DEFAULT;
4. createCoroutineUnintercepted：block 为BaseContinuationImpl的子类对象，实现create方法，返回一个新的Continuation对象；

5. intercepted：创建DispatchedContinuation，同时持有Continuation和Dispatcher，作为分发业务执行线程的桥梁；

> block.create ->continuation:ContinuationImpl 
> continuation.completion == StandaloneCoroutine
> continuation.context == StandaloneCoroutine.context
> continuation == DispatchedContinuation.continuation

### resumeCancellableWith
``` kotlin
internal inline fun resumeCancellableWith(result: Result<T>) {
        val state = result.toState()
        if (dispatcher.isDispatchNeeded(context)) {
            _state = state
            resumeMode = MODE_CANCELLABLE
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
isDispatchNeeded 默认为true ,暂时忽略线程分发（dispatcher.dispatch）过程，直接进入run方法执行

### run
``` kotlin
// `context:[StandaloneCoroutine{Active}@e43cb54, Dispatchers.Default] job:StandaloneCoroutine{Active}@e43cb54`
final override fun run() {
        assert { resumeMode != MODE_UNINITIALIZED }
        var fatalException: Throwable? = null
        try {
            val delegate = delegate as DispatchedContinuation<T>
            val continuation = delegate.continuation
            withContinuationContext(continuation, delegate.countOrElement) {
                val context = continuation.context
                val state = takeState() 
                val exception = getExceptionalResult(state)
                val job = if (exception == null && resumeMode.isCancellableMode) context[Job] else null
                if (job != null && !job.isActive) {
                    val cause = job.getCancellationException()
                    cancelCompletedResult(state, cause)
                    continuation.resumeWithStackTrace(cause)
                } else {
                    if (exception != null) {
                        continuation.resumeWithException(exception)
                    } else {
                        continuation.resume(getSuccessfulResult(state))
                    }
                }
            }
        } catch (e: Throwable) {
            fatalException = e
        } finally {
            fatalException?.let { handleFatalException(it) }
        }
    }
```
正常情况会直接进入continuation.resume

### resumeWith

```kotlin
//BaseContinuationImpl
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

对于示例中的launch方法 ，执行invokeSuspend，调用fetchData后挂起，等待恢复；

到这里我们发现，我们说launch方法，启动了一个协程，实际上就是创建一个续体，作为所有挂起函数调用的起点；

## delay

#### delay

```kotlin
public suspend fun delay(timeMillis: Long) {  
    if (timeMillis <= 0) return
    return suspendCancellableCoroutine sc@ { cont: CancellableContinuation<Unit> ->  
        if (timeMillis < Long.MAX_VALUE) {  
            cont.context.delay.scheduleResumeAfterDelay(timeMillis, cont)  
        }  
    }  
}

public suspend inline fun <T> suspendCancellableCoroutine(  
    crossinline block: (CancellableContinuation<T>) -> Unit  
): T =  
    suspendCoroutineUninterceptedOrReturn { uCont ->  
        val cancellable = CancellableContinuationImpl(uCont.intercepted(), resumeMode = MODE_CANCELLABLE)  
        cancellable.initCancellability()  
        block(cancellable)  
        cancellable.getResult()  
    }
```

#### scheduleResumeAfterDelay

```kotlin
// EventLoopImplBase
override fun scheduleResumeAfterDelay(timeMillis: Long, continuation: CancellableContinuation<Unit>) {
        val timeNanos = delayToNanos(timeMillis)
        if (timeNanos < MAX_DELAY_NS) {
            val now = nanoTime()
            DelayedResumeTask(now + timeNanos, continuation).also { task ->
                schedule(now, task)
                continuation.disposeOnCancellation(task)
            }
        }
    }

// EventLoopImplBase
fun schedule(now: Long, delayedTask: DelayedTask) {
        when (scheduleImpl(now, delayedTask)) {
            SCHEDULE_OK -> if (shouldUnpark(delayedTask)) unpark()
            SCHEDULE_COMPLETED -> reschedule(now, delayedTask)
            SCHEDULE_DISPOSED -> {} 
            else -> error("unexpected result")
        }
    }

// EventLoopImplBase
private fun scheduleImpl(now: Long, delayedTask: DelayedTask): Int {
        if (isCompleted) return SCHEDULE_COMPLETED
        val delayedQueue = _delayed.value ?: run {
            _delayed.compareAndSet(null, DelayedTaskQueue(now))
            _delayed.value!!
        }
        return delayedTask.scheduleTask(now, delayedQueue, this)
    }

// DelayedTask
fun scheduleTask(now: Long, delayed: DelayedTaskQueue, eventLoop: EventLoopImplBase): Int = synchronized<Int>(this) {
            if (_heap === DISPOSED_TASK) return SCHEDULE_DISPOSED 
            delayed.addLastIf(this) { firstTask ->
                if (eventLoop.isCompleted) return SCHEDULE_COMPLETED 
                if (firstTask == null) {
                    delayed.timeNow = now
                } else {
                    val firstTime = firstTask.nanoTime
                    val minTime = if (firstTime - now >= 0) now else firstTime
                    if (minTime - delayed.timeNow > 0) delayed.timeNow = minTime
                }
                if (nanoTime - delayed.timeNow < 0) nanoTime = delayed.timeNow
                true
            }
            return SCHEDULE_OK
        }
//EventLoopImplBase
private fun shouldUnpark(task: DelayedTask): Boolean = _delayed.value?.peek() === task
//EventLoopImplPlatform:
protected actual fun unpark() {
        val thread = thread // atomic read
        if (Thread.currentThread() !== thread)
            unpark(thread)
    }
```

schedule:执行task

1. scheduleImpl:创建DelayedTaskQueue;
2. scheduleTask:将scheduleTask加入TaskQueue

### run

```kotlin
//DefaultExecutor
override fun run() {
        ThreadLocalEventLoop.setEventLoop(this)
        registerTimeLoopThread()
        try {
            var shutdownNanos = Long.MAX_VALUE
            if (!notifyStartup()) return
            while (true) {
                Thread.interrupted() 
                var parkNanos = processNextEvent()
                if (parkNanos == Long.MAX_VALUE) {
                
                    val now = nanoTime()
                    if (shutdownNanos == Long.MAX_VALUE) shutdownNanos = now + KEEP_ALIVE_NANOS
                    val tillShutdown = shutdownNanos - now
                    if (tillShutdown <= 0) return
                    parkNanos = parkNanos.coerceAtMost(tillShutdown)
                } else
                    shutdownNanos = Long.MAX_VALUE
                if (parkNanos > 0) {
             
                    if (isShutdownRequested) return
                    parkNanos(this, parkNanos)
                }
            }
        } finally {
            _thread = null
            acknowledgeShutdownIfNeeded()
            unregisterTimeLoopThread()
            if (!isEmpty) thread 
        }
    }

override fun processNextEvent(): Long {
        if (processUnconfinedEvent()) return 0
        enqueueDelayedTasks()
        val task = dequeue()
        if (task != null) {
            platformAutoreleasePool { task.run() }
            return 0
        }
        return nextTime
    }
```

## async

```kotlin
public fun <T> CoroutineScope.async(
    context: CoroutineContext = EmptyCoroutineContext,
    start: CoroutineStart = CoroutineStart.DEFAULT,
    block: suspend CoroutineScope.() -> T
): Deferred<T> {
    val newContext = newCoroutineContext(context)
    val coroutine = if (start.isLazy)
        LazyDeferredCoroutine(newContext, block) else
        DeferredCoroutine<T>(newContext, active = true)
    coroutine.start(start, coroutine, block)
    return coroutine
}


private open class DeferredCoroutine<T>(
    parentContext: CoroutineContext,
    active: Boolean
) : AbstractCoroutine<T>(parentContext, true, active = active), Deferred<T> {
    override fun getCompleted(): T = getCompletedInternal() as T
    override suspend fun await(): T = awaitInternal() as T
    override val onAwait: SelectClause1<T> get() = onAwaitInternal as SelectClause1<T>
}
```

1. 与launch方法一样，async也可以启动一个协程，但async返回Deferred；

#### await

```kotlin
protected suspend fun awaitInternal(): Any? {
        while (true) { // lock-free loop on state
            val state = this.state
            if (state !is Incomplete) {
                if (state is CompletedExceptionally) {
                    recoverAndThrow(state.cause)
                }
                return state.unboxState()
            }
            if (startInternal(state) >= 0) break
        }
        return awaitSuspend()
    }

private suspend fun awaitSuspend(): Any? = suspendCoroutineUninterceptedOrReturn { uCont ->
        val cont = AwaitContinuation(uCont.intercepted(), this)
        cont.initCancellability()
        cont.disposeOnCancellation(invokeOnCompletion(handler = ResumeAwaitOnCompletion(cont)))
        cont.getResult()
    }
```

# 挂起与恢复

线程A挂起，等待线程B的执行，线程B执行完成，通知线程A恢复执行；挂起分为阻塞时挂起与非阻塞式挂起。

## 阻塞式

`join`实现

```java
public void blockSuspend() {
        Thread t1 = new Thread(() -> {
            System.out.println("t1 run");
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        });
        t1.start();
        try {
            t1.join();
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        System.out.println("main run");
    }
```

## 非阻塞式

在`Android`中通过`Handler`消息机制实现

```kotlin
val handler = Handler(Looper.getMainLooper())
thread {    
    handler.post(...)
}.start()
```

## 协程式

在协程作用域内，用阻塞式的代码风格实现非阻塞式的挂起与恢复。

```kotlin
fun main() {
    GlobalScope.launch{
       val name = fetchData()
        println("name $name")
    }
    println("main end")
    Thread.sleep(2000)
}


suspend fun fetchData():String{
    delay(1000)
    return "Joe"
}

//main end
//name Joe
```

用delay模拟在子线程中获取数据，又恢复到主线程中执行；

## 挂起函数

用suspend修饰的函数，称为挂起函数。挂起函数只能在挂起函数内调用，第一个挂起函数就是协程体。

#### 实现原理

定义挂起函数

```kotlin
suspend fun fetchData():String{
    delay(1000)
    return "Joe"
}
```
对应的Java代码

```java
 @Nullable
   public static final Object fetchData(@NotNull Continuation var0) { //1
      Object $continuation;
      label20: {
         if (var0 instanceof <undefinedtype>) {
            $continuation = (<undefinedtype>)var0;
            if ((((<undefinedtype>)$continuation).label & Integer.MIN_VALUE) != 0) {
               ((<undefinedtype>)$continuation).label -= Integer.MIN_VALUE;
               break label20;
            }
         }

         $continuation = new ContinuationImpl(var0) { //2
            Object result;
            int label;

            @Nullable
            public final Object invokeSuspend(@NotNull Object $result) { //4
               this.result = $result;
               this.label |= Integer.MIN_VALUE;
               return MyCoroutineKt.fetchData(this);
            }
         };
      }

      Object $result = ((<undefinedtype>)$continuation).result;
      Object var3 = IntrinsicsKt.getCOROUTINE_SUSPENDED();
      switch (((<undefinedtype>)$continuation).label) {
         case 0: //3
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
1. 经过编译后，为挂起函数添加Continuation类型的参数，称为续体；
2. 创建匿名续体对象$continuation，将新增的续体参数保存在属性completion，从它的名称就可知道它的作用就是用来执行完挂起函数后恢复协程的执行。
3. 当函数被调用时label为0，将$continuation对象传递挂起函数delay，delay也是一个挂起函数，label+1，自身挂起；
4. 由delay函数调用$continuation的invokeSuspend方法恢复挂起函数执行，执行下一个lable业务；

##### CPS转换

为函数添加一个Continuation类型的参数，用于接受函数执行的结果，在挂起函数恢复后，由续体来决定接下来的操作，作用和handler中的callback类似。比如我们在挂起函数调用delay,实际上是将当前函数在delay之后的逻辑封装成续体，delay执行完，由续体来执行之后的业务。

##### 状态机

将挂起函数转换为一个状态机，每个挂起点对应一个状态，并将状态保存续体中，并通过 `Continuation` 控制恢复，恢复时执行下一个状态的挂起直到函数返回。

### 挂起点

一个真正的挂起函数，不是我们用suspend修饰了一个函数，然后在代码块中加点耗时操作就可以。而是只能调用特定的API或者业务中调用了真正的挂起函数。检查方法就是去查看对应的java代码，如果函数执行时先返回IntrinsicsKt.getCOROUTINE_SUSPENDED() 挂起等待恢复，才是真正的挂起函数。

# 结构化并发

通过层级化作用域和父子关系来管理协程生命周期的机制，确保协程的正常的正常的完成，取消和清理，确保资源安全、避免泄漏，并简化异步代码的编写与维护；
### 协程作用域（CoroutineScope)
定义协程运行的上下文环境，包含 `Job` 和 `CoroutineDispatcher`。
### 协程任务（Job）
表示一个协程任务，支持取消和层级管理。
## CoroutineScope
### 常见作用域
  1. GlobalScope：全局作用域（慎用，生命周期与应用一致，易泄漏）。
  2. viewModelScope：与 ViewModel 绑定，当 ViewModel 销毁时自动取消。
  3. lifecycleScope：与 Lifecycle 组件（如 Activity/Fragment）绑定。
  4. 自定义作用域：通过 `CoroutineScope(SupervisorJob() + Dispatchers.IO)` 创建。

## SuperVisorJob/supervisorJobScope

```kotlin
private class SupervisorJobImpl(parent: Job?) : JobImpl(parent) {
    override fun childCancelled(cause: Throwable): Boolean = false
}
```

重写childCancelled方法，不执行任何逻辑，隔离子协程的异常；

## Scope如何实现生命周期管理:以LifecycleScope为例

```kotlin

public val LifecycleOwner.lifecycleScope: LifecycleCoroutineScope
    get() = lifecycle.coroutineScope

public val Lifecycle.coroutineScope: LifecycleCoroutineScope
    get() {
        while (true) {
            val existing = internalScopeRef.get() as LifecycleCoroutineScopeImpl?
            if (existing != null) {
                return existing
            }
            val newScope = LifecycleCoroutineScopeImpl(
                this,
                SupervisorJob() + Dispatchers.Main.immediate
            )
            if (internalScopeRef.compareAndSet(null, newScope)) {
                newScope.register()
                return newScope
            }
        }
    }


internal class LifecycleCoroutineScopeImpl(
    override val lifecycle: Lifecycle,
    override val coroutineContext: CoroutineContext
) : LifecycleCoroutineScope(), LifecycleEventObserver {
    init {
        if (lifecycle.currentState == Lifecycle.State.DESTROYED) {
            coroutineContext.cancel()
        }
    }

    fun register() {
        launch(Dispatchers.Main.immediate) {
            if (lifecycle.currentState >= Lifecycle.State.INITIALIZED) {
                lifecycle.addObserver(this@LifecycleCoroutineScopeImpl)
            } else {
                coroutineContext.cancel()
            }
        }
    }

    override fun onStateChanged(source: LifecycleOwner, event: Lifecycle.Event) {
        if (lifecycle.currentState <= Lifecycle.State.DESTROYED) {
            lifecycle.removeObserver(this)
            coroutineContext.cancel()
        }
    }
}

public fun CoroutineContext.cancel(cause: CancellationException? = null) {
    this[Job]?.cancel(cause)
}
```

1. LifecycleCoroutineScopeImpl 同时实现CoroutineScope 和 LifecycleEventObserver接口，对象创建时就通过register实现与Lifecycle生命周期绑定；
2. 当Activity destroy时，执行 coroutineContext.cancel，也就是SupervisorJob的cancel方法回到Job的逻辑中。

# CoroutineContext

保存协程上下文的自定义集合，主要由以下4个`Element`组成：
- `Job`：协程的唯一标识，用来控制协程的生命周期(`new、active、completing、completed、cancelling、cancelled`)；
- `CoroutineDispatcher`：协程调度器，指定协程运行的线程(`IO、Default、Main、Unconfined`);
- `CoroutineName`: 指定协程的名称，默认为coroutine;
- `CoroutineExceptionHandler`: 指定协程的异常处理器，用来处理未捕获的异常.

## 数据结构

### Element

``` kotlin
public interface Key<E : Element>

public interface Element : CoroutineContext {  
    /**  
     * A key of this coroutine context element.     */    public val key: Key<*>  
  
    public override operator fun <E : Element> get(key: Key<E>): E? =  
        @Suppress("UNCHECKED_CAST")  
        if (this.key == key) this as E else null  
  
    public override fun <R> fold(initial: R, operation: (R, Element) -> R): R =  
        operation(initial, this)  
  
    public override fun minusKey(key: Key<*>): CoroutineContext =  
        if (this.key == key) EmptyCoroutineContext else this  
}
```
CoroutineContext中的元素都必须实现Element接口，每个元素都有唯一的Key, 原来检索元素。

### plus

``` kotlin
public operator fun plus(context: CoroutineContext): CoroutineContext =  
    if (context === EmptyCoroutineContext) this else // fast path -- avoid lambda creation  
        context.fold(this) { acc, element ->  
            val removed = acc.minusKey(element.key)  
            if (removed === EmptyCoroutineContext) element else {  
                // make sure interceptor is always last in the context (and thus is fast to get when present)  
                val interceptor = removed[ContinuationInterceptor]  
                if (interceptor == null) CombinedContext(removed, element) else {  
                    val left = removed.minusKey(ContinuationInterceptor)  
                    if (left === EmptyCoroutineContext) CombinedContext(element, interceptor) else  
                        CombinedContext(CombinedContext(left, element), interceptor)  
                }  
            }  
        }
```
1. `plus EmptyCoroutineContext` ：`Dispatchers.Main + EmptyCoroutineContext` 结果:`Dispatchers.Main`。
2. `plus` 相同类型的`Element`：`CoroutineName("c1") + CoroutineName("c2")`结果: `CoroutineName("c2")`。相同类型的直接替换掉。
3. `plus`方法的调用方没有`Dispatcher`相关的Element：`CoroutineName("c1") + Job()`结果:`CoroutineName("c1") <- Job`。头插法被plus的(`Job`)放在链表头部
4. `plus`方法的调用方只有`Dispatcher`相关的`Element` ：`Dispatchers.Main + Job()`结果:`Job <- Dispatchers.Main`。虽然是头插法，但是`ContinuationInterceptor`必须在链表头部。
5. `plus`方法的调用方是包含`Dispatcher`相关Element的链表： `Dispatchers.Main + Job() + CoroutineName("c5")`结果:`Job <- CoroutineName("c5") <- Dispatchers.Main`。Dispatchers.Main在链表头部，其它的采用头插法。

# Continuation

# Job

### 简介

```kotlin
public interface Job : CoroutineContext.Element {
    public companion object Key : CoroutineContext.Key<Job>
}
```

Job是CoroutineContext集合的重要组成元素，Key类型为Job，表示一个协程任务，支持取消和层级管理。

### Job的生命周期

![image.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/e7743d8628b64065b3d8248f26e01350~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

```kotlin
private val _state = atomic<Any?>(if (active) EMPTY_ACTIVE else EMPTY_NEW)
```

### initParentJob

``` kotlin
//AbstractCoroutine
init {
    if (initParentJob) initParentJob(parentContext[Job]) //1
}
//JobSupport
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
public final override fun start(): Boolean {
        loopOnState { state ->
            when (startInternal(state)) {
                FALSE -> return false
                TRUE -> return true
            }
        }
    }

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
            else -> return FALSE 
        }
    }
```

1. 通过lunch启动协程时,初始化AbstractCoroutine，initParentJob默认为true;
2. initParentJob，启动协程的CoroutineContext中是否包含Job Element，此Job为父Job
   1. start：父Job不为空，启动父Job， 将state设置为 EMPTY_ACTIVE 或 NodeList；
   2. attachChild：将父子Job进行关联；

### attachChild

``` kotlin

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
 private fun promoteEmptyToNodeList(state: Empty) {
        val list = NodeList()
        val update = if (state.isActive) list else InactiveNodeList(list)
        _state.compareAndSet(state, update)
    }

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
   3. 后续再加入子Job,父Job的state已经是Incomplete 
      1. state.list == null，创建NodeList，并将state设置为NodeList；
      2. state.list !=null，执行tryAdd逻辑，将子Job加入到list中；

### cancle

``` kotlin
public override fun cancel(cause: CancellationException?) {
        cancelInternal(cause ?: defaultCancellationException())
    }

public open fun cancelInternal(cause: Throwable) {
        cancelImpl(cause)
    }

internal fun cancelImpl(cause: Any?): Boolean {
        var finalState: Any? = COMPLETING_ALREADY
        if (onCancelComplete) { //1
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

onCancelComplete 默认为false，JobImpl  和 CompletableDeferredImpl重写为true

#### makeCancelling

``` kotlin
private fun makeCancelling(cause: Any?): Any? {
        var causeExceptionCache: Throwable? = null 
        loopOnState { state ->
            when (state) {
                is Finishing -> { //1
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
                        if (tryMakeCancelling(state, causeException)) return COMPLETING_ALREADY //2
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


```

#### tryMakeCancelling

````kotlin
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
````

1. tryMakeCancelling
   1. getOrPromoteCancellingList：返回state对应的list，state为Empty则新建；
   2. 将state 设置为Finishing

2. notifyCancelling
   1. onCancelling 默认为空实现；
   2. notifyHandlers：通知所有子Job，父Job正在取消；
      1. 子Job一般为ChildHandleNode， onCancelling为true,执行其的invoke方法;
      2. 执行其invoke方法，invoke -> JobSupport.parentCancelled -> JobSupport.cancelImpl;
   3. cancelParent：通知父Job，子Job正在取消；


#### makeCompletingOnce

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

10. CoroutineScope.launch的block函数参数为CoroutineScope的扩展函数，传入的scope为AbstractCoroutine，当协程体内业务处理完成，执行completion也就是AbstractCoroutine的resumeWith方法。

#### makeCompletingOnce

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

#### tryMakeCompletingSlowPath

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

## 简介

```kotlin
public interface Flow<out T> {
    public suspend fun collect(collector: FlowCollector<T>)
}

public fun interface FlowCollector<in T> {
    public suspend fun emit(value: T)
}
```

Flow是基于协程的响应式数据流，上游实现为Flow，负责数据的产生、变换、组合，下游为FlowCollector负责消费数据；

## 冷流

数据只有在有消费者（collect）时才开始生产

```kotlin
fun main() {
    GlobalScope.launch {
        flow {
            for(i in 0 ..3){
                emit(1)
            }
        }.collect{
            println(it)
        }
    }
    Thread.sleep(10000)
}

public fun <T> flow(block: suspend FlowCollector<T>.() -> Unit): Flow<T> = SafeFlow(block)

private class SafeFlow<T>(private val block: suspend FlowCollector<T>.() -> Unit) : AbstractFlow<T>() {
    override suspend fun collectSafely(collector: FlowCollector<T>) {
        collector.block()
    }
}

//AbstractFlow
public final override suspend fun collect(collector: FlowCollector<T>) {
        val safeCollector = SafeCollector(collector, coroutineContext)
        try {
            collectSafely(safeCollector)
        } finally {
            safeCollector.releaseIntercepted()
        }
    }
```

## map

map为Flow的扩展函数，扩展对象为flow1,返回一个新的flow对象 flow2;

```java
public inline fun <T, R> Flow<T>.map(crossinline transform: suspend (value: T) -> R): Flow<R> = transform { value ->
    return@transform emit(transform(value))//3
}

public inline fun <T, R> Flow<T>.transform(
    @BuilderInference crossinline transform: suspend FlowCollector<R>.(value: T) -> Unit
): Flow<R> = flow { //1
    collect { value -> //2
        return@collect transform(value)
    }
}

```

1. 创建一个新的Flow  flow2;
2. 为flow1新建一个FlowCollector,首先执行Flow1的collect，传入原始数据value;
3. 为flow2指定FlowCollector后，emit经过transform之后的value;

## 热流

数据的生成与收集者无关，即使没有订阅者也会执行（需手动启动）

### MutableStateFlow

### MutableSharedFlow

```kotlin
public fun <T> MutableSharedFlow(
    replay: Int = 0,
    extraBufferCapacity: Int = 0,
    onBufferOverflow: BufferOverflow = BufferOverflow.SUSPEND
): MutableSharedFlow<T> {
    require(replay >= 0) { "replay cannot be negative, but was $replay" }
    require(extraBufferCapacity >= 0) { "extraBufferCapacity cannot be negative, but was $extraBufferCapacity" }
    require(replay > 0 || extraBufferCapacity > 0 || onBufferOverflow == BufferOverflow.SUSPEND) {
        "replay or extraBufferCapacity must be positive with non-default onBufferOverflow strategy $onBufferOverflow"
    }
    val bufferCapacity0 = replay + extraBufferCapacity
    val bufferCapacity = if (bufferCapacity0 < 0) Int.MAX_VALUE else bufferCapacity0 
    return SharedFlowImpl(replay, bufferCapacity, onBufferOverflow) //1
}
```

1. SharedFlowImpl
   1. replay：新订阅者接收的历史数据量；
   2. bufferCapacity：额外缓冲区大小
   3. onBufferOverflow:默认为SUSPEND

#### emit

```kotlin
override suspend fun emit(value: T) {
        if (tryEmit(value)) return 
        emitSuspend(value)
    }

override fun tryEmit(value: T): Boolean {
        var resumes: Array<Continuation<Unit>?> = EMPTY_RESUMES
        val emitted = synchronized(this) {
            if (tryEmitLocked(value)) {
                resumes = findSlotsToResumeLocked(resumes)
                true
            } else {
                false
            }
        }
        for (cont in resumes) cont?.resume(Unit)
        return emitted
    }

private fun tryEmitLocked(value: T): Boolean {
        if (nCollectors == 0) return tryEmitNoCollectorsLocked(value)
        if (bufferSize >= bufferCapacity && minCollectorIndex <= replayIndex) {
            when (onBufferOverflow) {
                BufferOverflow.SUSPEND -> return false 
                BufferOverflow.DROP_LATEST -> return true 
                BufferOverflow.DROP_OLDEST -> {}
            }
        }
        enqueueLocked(value)
        bufferSize++
        if (bufferSize > bufferCapacity) dropOldestLocked()
        if (replaySize > replay) { 
            updateBufferLocked(replayIndex + 1, minCollectorIndex, bufferEndIndex, queueEndIndex)
        }
        return true
    }

private fun tryEmitNoCollectorsLocked(value: T): Boolean {
        assert { nCollectors == 0 } //1
        if (replay == 0) return true //2
        enqueueLocked(value) 
        bufferSize++ 
        if (bufferSize > replay) dropOldestLocked()
        minCollectorIndex = head + bufferSize 
        return true
    }
```

1. emit
   1. tryEmit:尝试发射数据，成功则返回
   2. emitSuspend
2. tryEmit
   1. tryEmitLocked
3. tryEmitLocked
   1. tryEmitNoCollectorsLocked，当前没有指定收集者
4. tryEmitNoCollectorsLocked
   1. 当前不存在订阅者；
   2. 无需保存历史数据；
5. enqueueLocked

## flowOn的实现

```kotlin
public fun <T> Flow<T>.flowOn(context: CoroutineContext): Flow<T> {
    checkFlowContext(context)
    return when {
        context == EmptyCoroutineContext -> this
        this is FusibleFlow -> fuse(context = context)
        else -> ChannelFlowOperatorImpl(this, context = context)
    }
}
```



# CancellableContinuationImpl

CancellableContinuationImpl 监听协程取消的Continuation

```kotlin
public override fun initCancellability() {
        val handle = installParentHandle()
            ?: return
        if (isCompleted) {
            handle.dispose()
            _parentHandle.value = NonDisposableHandle
        }
    }

private fun installParentHandle(): DisposableHandle? {
        val parent = context[Job] ?: return null 
        val handle = parent.invokeOnCompletion(handler = ChildContinuation(this))
        _parentHandle.compareAndSet(null, handle)
        return handle
    }

internal fun Job.invokeOnCompletion(
    invokeImmediately: Boolean = true,
    handler: JobNode,
): DisposableHandle = when (this) {
    is JobSupport -> invokeOnCompletionInternal(invokeImmediately, handler)
    else -> invokeOnCompletion(handler.onCancelling, invokeImmediately, handler::invoke)
}
//JobSupport
internal fun invokeOnCompletionInternal(
        invokeImmediately: Boolean,
        node: JobNode
    ): DisposableHandle {
        node.job = this
        val added = tryPutNodeIntoList(node) { state, list ->
            if (node.onCancelling) {
                val rootCause = (state as? Finishing)?.rootCause
                if (rootCause == null) {
                    list.addLast(node, LIST_CANCELLATION_PERMISSION or LIST_ON_COMPLETION_PERMISSION)
                } else {
                    if (invokeImmediately) node.invoke(rootCause)
                    return NonDisposableHandle
                }
            } else {
                list.addLast(node, LIST_ON_COMPLETION_PERMISSION)
            }
        }
        when {
            added -> return node
            invokeImmediately -> node.invoke((state as? CompletedExceptionally)?.cause)
        }
        return NonDisposableHandle
    }
```

1. 跟踪它的初始化流程initCancellability -> installParentHandle->invokeOnCompletion->invokeOnCompletionInternal
2. 熟悉的tryPutNodeIntoList，在Job的cancel流程中了解到，该方法会将子Job节点加入到父Job的状态管理列表中。

# suspendCoroutineUninterceptedOrReturn

# 参考链接

- [Kotlin协程createCoroutine和startCoroutine原理](https://www.cnblogs.com/xfhy/p/17152341.html)
  - [IntrinsicsJvm.kt](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/jvm/src/kotlin/coroutines/intrinsics/IntrinsicsJvm.kt)
- [使用PlantUML绘制类图](https://juejin.cn/post/6844903731293585421?searchId=20241002214831CBFB639525066B7006C1)
- [【kotlin】- delay函数实现原理](https://www.jianshu.com/p/2bcd6e21b496)