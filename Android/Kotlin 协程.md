# 挂起函数
## 挂起与恢复
线程A挂起，等待线程B的执行，线程B执行完成，通知线程A恢复执行；挂起分为阻塞时挂起与非阻塞式挂起。
### 阻塞式
join实现
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

### 非阻塞式
在Android中通过handler实现

### 协程式

在协程作用域内，用阻塞式的代码风格实现非阻塞式的挂起与恢复。

## 什么是挂起函数？
通过suspend修饰的函数，称为挂起函数。`挂起函数只能在挂起函数内调用`。

### 实现原理

定义挂起函数

```kotlin
suspend fun getName():String{
    delay(100)
    return "Joe"
}
```
对应的Java代码

> Tools -> Koltin -> Show Kotlin ByteCode

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
4. 由delay函数自动恢复getName执行，调用Name的invokeSuspend方法，执行下一个lable业务；

#### CPS转换

为函数添加一个Continuation类型的参数，用于接受函数执行的结果，在挂起函数恢复后，由续体来决定接下来的操作，作用和handler中的callback类似。比如我们在挂起函数调用delay,实际上当前函数在delay之后的逻辑封装成续体，delay执行完，由续体来执行之后的业务。

#### 状态机

将挂起函数转换为一个状态机，每个挂起点对应一个状态，并将状态保存续体中，并通过 `Continuation` 控制恢复，恢复时执行下一个状态的挂起直到函数返回。

### 挂起点

一个真正的挂起函数，不是我们用suspend修饰了一个函数，然后在代码块中加点耗时操作就可以。而是只能调用特定的API或者业务中调用了真正的挂起函数。检查方法就是去查看对应的java代码，如果函数执行时先返回IntrinsicsKt.getCOROUTINE_SUSPENDED() 挂起等待恢复，才是真正的挂起函数。

## lunch流程分析：挂起后如何自动恢复

### lunch

```kotlin
fun main() {
    GlobalScope.launch {
 println("receiver $this context:${this.coroutineContext} job:${this.coroutineContext[Job]}")
    }
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
//AbstractCoroutine
public fun <R> start(start: CoroutineStart, receiver: R, block: suspend R.() -> T) {
        start(block, receiver, this)
    }

//CoroutineStart 1
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
```

1. CoroutineStart 重载了invoke函数，一个kotlin 语法糖，可以直接通过对象调用该方法。
2. start 默认为CoroutineStart.DEFAULT，block 为挂起函数对象
### startCoroutineCancellable
``` kotlin
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
        create(receiver, probeCompletion) //2
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
1. `suspend (R) -> T` 为挂起扩展函数的函数对象，挂起函数为会函数添加一个参数，所以block是同时实现Function2和挂起方法的函数对象 `class com/example/myapplication/coroutine/MyCoroutineKt$main$1 extends kotlin/coroutines/jvm/internal/SuspendLambda implements kotlin/jvm/functions/Function2`；为了方便我们称`MyCoroutineKt$main$1`为`Block`
3. createCoroutineUnintercepted：block 为BaseContinuationImpl的子类对象，实现create方法，返回一个新的Continuation对象；
4. intercepted：创建DispatchedContinuation，同时持有Continuation和Dispatcher，作为分发业务执行线程的桥梁；

block.create ->continuation:ContinuationImpl 
continuation.completion == StandaloneCoroutine
continuation.context == StandaloneCoroutine.context
continuation == DispatchedContinuation.continuation

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
1. isDispatchNeeded 默认为true ,暂时忽略线程分发（dispatcher.dispatch）过程，直接进入run方法执行
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

对于示例中的lunch方法 ，直接执行completion.resumeWith，进入StandaloneCoroutine执行；

# CoroutineScope

结构化并发
结构化并发 来解决协程不可控的问题：

1. 可以取消协程任务；
2. 协程任务正在执行中，可以追踪任务的状态；
3. 协程任务正在执行中，如果出现异常，可以发出信息；
4. 管理协程的生命的周期；
#### 常用的子类
GlobalScope: 进程级别，跟随App进程；
MainScope: 在Activity中使用，可以在onDestroy中使用
ViewModelScope:绑定ViewModel生命周期；
LifecycleScope: 跟随Lifecycle生命周期，绑定Activity/Fragment的生命周期
Scope如何实现生命周期管理？

### CoroutineContext
#### 简介

保存协程上下文的自定义集合，主要由以下4个`Element`组成：
- `Job`：协程的唯一标识，用来控制协程的生命周期(`new、active、completing、completed、cancelling、cancelled`)；
- `CoroutineDispatcher`：协程调度器，指定协程运行的线程(`IO、Default、Main、Unconfined`);
- `CoroutineName`: 指定协程的名称，默认为coroutine;
- `CoroutineExceptionHandler`: 指定协程的异常处理器，用来处理未捕获的异常.
#### 数据结构
##### Element
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
##### `plus`
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
### delay 
``` kotlin
public suspend fun delay(timeMillis: Long) {  
    if (timeMillis <= 0) return // don't delay  
    return suspendCancellableCoroutine sc@ { cont: CancellableContinuation<Unit> ->  
        // if timeMillis == Long.MAX_VALUE then just wait forever like awaitCancellation, don't schedule.  
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
        /*  
         * For non-atomic cancellation we setup parent-child relationship immediately         * in case when `block` blocks the current thread (e.g. Rx2 with trampoline scheduler), but         * properly supports cancellation.         */        cancellable.initCancellability()  
        block(cancellable)  
        cancellable.getResult()  
    }
```
### 协程的取消
1. yield  /isActive /ensureActive
2. 取消之后 资源无法释放：
	- try catch  cancel异常的处理；
	- use函数释放
3. 子协程取消（内部会抛出JobCancllationException），不会影响父协程的工作
### 协程的超时任务
### 协程的异常处理
异常会传递给父协程
### SuperVisorJob/supervisorJobScope
### 草稿
4. join与await 10秒与19秒？
5. supervisorScope 与coroutineScope
7. 大写的函数 ：简单工厂设计模式
8. async 立即开始调度 返回值和异常 等待await
9. 只有顶级协程才能处理异常？ExceptionHandler
10. 全局异常处理？自定义服务
11. flow与Rxjava
12. flow 冷流？
### flow
1. Flow上下文保存机制？，上下文保持一致？




# 参考链接

- [Kotlin协程createCoroutine和startCoroutine原理](https://www.cnblogs.com/xfhy/p/17152341.html)
  - [IntrinsicsJvm.kt](https://github.com/JetBrains/kotlin/blob/master/libraries/stdlib/jvm/src/kotlin/coroutines/intrinsics/IntrinsicsJvm.kt)
- [使用PlantUML绘制类图](https://juejin.cn/post/6844903731293585421?searchId=20241002214831CBFB639525066B7006C1)