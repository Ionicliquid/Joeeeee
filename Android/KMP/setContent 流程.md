# setContent
```kotlin
public fun ComponentActivity.setContent(  
    parent: CompositionContext? = null,  
    content: @Composable () -> Unit  
) {  
    val existingComposeView =  
        window.decorView.findViewById<ViewGroup>(android.R.id.content).getChildAt(0) as? ComposeView  
  
    if (existingComposeView != null)  
        with(existingComposeView) {  
            setParentCompositionContext(parent)  
            setContent(content)  
        }  
    else  
        ComposeView(this).apply {  
	        setParentCompositionContext(parent)    
		    setContent(content)  
	        setOwners()  
            setContentView(this, DefaultActivityContentLayoutParams)  
        }  
}
// AbstractComposeView
override fun onAttachedToWindow() {  
    super.onAttachedToWindow()  
  
    previousAttachedWindowToken = windowToken  
  
    if (shouldCreateCompositionOnAttachedToWindow) {  
        ensureCompositionCreated()  
    }  
}
//ComposeView
fun setContent(content: @Composable () -> Unit) {  
    shouldCreateCompositionOnAttachedToWindow = true  
    this.content.value = content  
    if (isAttachedToWindow) {  
        createComposition()  
    }  
}
```
ComposeView#setContent 中将shouldCreateCompositionOnAttachedToWindow = true  ，当onAttachedToWindow 回调时，执行ensureCompositionCreated；
# ensureCompositionCreated
```kotlin
private fun ensureCompositionCreated() {  
    if (composition == null) {  
        try {  
            creatingComposition = true  
            composition = setContent(resolveParentCompositionContext()) { Content() }  
        } finally {  
            creatingComposition = false  
        }  
    }  
}

private fun resolveParentCompositionContext() =  
    parentContext  
        ?: findViewTreeCompositionContext()?.cacheIfAlive()  
        ?: cachedViewTreeCompositionContext?.get()?.takeIf { it.isAlive }  
        ?: windowRecomposer.cacheIfAlive()

internal val View.windowRecomposer: Recomposer  
    get() {  
        checkPrecondition(isAttachedToWindow) {  
            "Cannot locate windowRecomposer; View $this is not attached to a window"        }  
        val rootView = contentChild  
        return when (val rootParentRef = rootView.compositionContext) {  
            null -> WindowRecomposerPolicy.createAndInstallWindowRecomposer(rootView)  
            is Recomposer -> rootParentRef  
            else -> error("root viewTreeParentCompositionContext is not a Recomposer")  
        }  
    }

internal fun AbstractComposeView.setContent(  
    parent: CompositionContext,  
    content: @Composable () -> Unit  
): Composition {  
    GlobalSnapshotManager.ensureStarted()  
    val composeView =  
        if (childCount > 0) {  
            getChildAt(0) as? AndroidComposeView  
        } else {  
            removeAllViews()  
            null  
        }  
            ?: AndroidComposeView(context, parent.effectCoroutineContext).also {  
                addView(it.view, DefaultLayoutParams)  
            }  
    return doSetContent(composeView, parent, content)  
}

internal object GlobalSnapshotManager {  
    private val started = AtomicBoolean(false)  
    private val sent = AtomicBoolean(false)  
  
    fun ensureStarted() {  
        if (started.compareAndSet(false, true)) {  
            val channel = Channel<Unit>(1)  
            CoroutineScope(AndroidUiDispatcher.Main).launch {  
                channel.consumeEach {  
                    sent.set(false)  
                    Snapshot.sendApplyNotifications()  
                }  
            }            Snapshot.registerGlobalWriteObserver {  
                if (sent.compareAndSet(false, true)) {  
                    channel.trySend(Unit)  
                }  
            }  
        }  
    }  
}

```
1. resolveParentCompositionContext 创建CompositionContext也就是Recompser
2. doSetContent
3. GlobalSnapshotManager.ensureStarted 注册GlobalSnapshot的写监听，当状态发生变化时，触发Snapshot.sendApplyNotifications
#### resolveParentCompositionContext
```kotlin
private fun resolveParentCompositionContext() =  
    parentContext  
        ?: findViewTreeCompositionContext()?.cacheIfAlive()  
        ?: cachedViewTreeCompositionContext?.get()?.takeIf { it.isAlive }  
        ?: windowRecomposer.cacheIfAlive()

internal val View.windowRecomposer: Recomposer  
    get() {  
        checkPrecondition(isAttachedToWindow) {  
            "Cannot locate windowRecomposer; View $this is not attached to a window"        }  
        val rootView = contentChild  
        return when (val rootParentRef = rootView.compositionContext) {  
            null -> WindowRecomposerPolicy.createAndInstallWindowRecomposer(rootView)  
            is Recomposer -> rootParentRef  
            else -> error("root viewTreeParentCompositionContext is not a Recomposer")  
        }  
    }

val LifecycleAware: WindowRecomposerFactory = WindowRecomposerFactory { rootView ->  
    rootView.createLifecycleAwareWindowRecomposer()  
}

fun View.createLifecycleAwareWindowRecomposer(  
    coroutineContext: CoroutineContext = EmptyCoroutineContext,  
    lifecycle: Lifecycle? = null  
): Recomposer {  
    val baseContext =  
        if (  
            coroutineContext[ContinuationInterceptor] == null ||  
                coroutineContext[MonotonicFrameClock] == null  
        ) {  
            AndroidUiDispatcher.CurrentThread + coroutineContext  
        } else coroutineContext  
    val pausableClock =  
        baseContext[MonotonicFrameClock]?.let { PausableMonotonicFrameClock(it).apply { pause() } }  
  
    var systemDurationScaleSettingConsumer: MotionDurationScaleImpl? = null  
    val motionDurationScale =  
        baseContext[MotionDurationScale]  
            ?: MotionDurationScaleImpl().also { systemDurationScaleSettingConsumer = it }  
  
    val contextWithClockAndMotionScale =  
        baseContext + (pausableClock ?: EmptyCoroutineContext) + motionDurationScale  
    val recomposer =  
        Recomposer(contextWithClockAndMotionScale).also { it.pauseCompositionFrameClock() }  
    val runRecomposeScope = CoroutineScope(contextWithClockAndMotionScale)  
	//.....
    return recomposer  
}


```
![[recomposer的协程上下文.png]]
1. 组合 3 种 CoroutineContext 来构造Recomposer和 协程作用域 runRecomposeScope，如上图；

#### runRecomposeAndApplyChanges
```kotlin
suspend fun runRecomposeAndApplyChanges() = recompositionRunner { parentFrameClock ->  
    val toRecompose = mutableListOf<ControlledComposition>()  
    val toInsert = mutableListOf<MovableContentStateReference>()  
    val toApply = mutableListOf<ControlledComposition>()  
    val toLateApply = mutableScatterSetOf<ControlledComposition>()  
    val toComplete = mutableScatterSetOf<ControlledComposition>()  
    val modifiedValues = MutableScatterSet<Any>()  
    val modifiedValuesSet = modifiedValues.wrapIntoSet()  
    val alreadyComposed = mutableScatterSetOf<ControlledComposition>()  
  
    fun clearRecompositionState() {  
        synchronized(stateLock) {  
            toRecompose.clear()  
            toInsert.clear()  
  
            toApply.fastForEach {  
                it.abandonChanges()  
                recordFailedCompositionLocked(it)  
            }  
            toApply.clear()  
  
            toLateApply.forEach {  
                it.abandonChanges()  
                recordFailedCompositionLocked(it)  
            }  
            toLateApply.clear()  
  
            toComplete.forEach { it.changesApplied() }  
            toComplete.clear()  
  
            modifiedValues.clear()  
  
            alreadyComposed.forEach {  
                it.abandonChanges()  
                recordFailedCompositionLocked(it)  
            }  
            alreadyComposed.clear()  
        }  
    }  
  
    fun fillToInsert() {  
        toInsert.clear()  
        synchronized(stateLock) {  
            movableContentAwaitingInsert.fastForEach { toInsert += it }  
            movableContentAwaitingInsert.clear()  
        }  
    }  
  
    while (shouldKeepRecomposing) {  
        awaitWorkAvailable()  
        if (!recordComposerModifications()) continue  
        parentFrameClock.withFrameNanos { frameTime ->  
            if (hasBroadcastFrameClockAwaiters) {  
                trace("Recomposer:animation") {  
                    broadcastFrameClock.sendFrame(frameTime)  
                    Snapshot.sendApplyNotifications()  
                }  
            }  
  
            trace("Recomposer:recompose") {  
                recordComposerModifications()  
                synchronized(stateLock) {  
                    compositionInvalidations.forEach { toRecompose += it }  
                    compositionInvalidations.clear()  
                }  
                modifiedValues.clear()  
                alreadyComposed.clear()  
                while (toRecompose.isNotEmpty() || toInsert.isNotEmpty()) {  
                    try {  
                        toRecompose.fastForEach { composition ->  
                            performRecompose(composition, modifiedValues)?.let { toApply += it }  
                            alreadyComposed.add(composition)  
                        }  
                    } catch (e: Throwable) {  
                        processCompositionError(e, recoverable = true)  
                        clearRecompositionState()  
                        return@withFrameNanos  
                    } finally {  
                        toRecompose.clear()  
                    }  
                    if (modifiedValues.isNotEmpty() || compositionInvalidations.isNotEmpty()) {  
                        synchronized(stateLock) {  
                            knownCompositionsLocked().fastForEach { value ->  
                                if (  
                                    value !in alreadyComposed &&  
                                    value.observesAnyOf(modifiedValuesSet)  
                                ) {  
                                    toRecompose += value  
                                }  
                            }  
                            compositionInvalidations.removeIf { value ->  
                                if (value !in alreadyComposed && value !in toRecompose) {  
                                    toRecompose += value  
                                    true  
                                } else {  
                                    false  
                                }  
                            }  
                        }                    }  
  
                    if (toRecompose.isEmpty()) {  
                        try {  
                            fillToInsert()  
                            while (toInsert.isNotEmpty()) {  
                                toLateApply += performInsertValues(toInsert, modifiedValues)  
                                fillToInsert()  
                            }  
                        } catch (e: Throwable) {  
                            processCompositionError(e, recoverable = true)  
                            clearRecompositionState()  
                            return@withFrameNanos  
                        }  
                    }  
                }  
  
                if (toApply.isNotEmpty()) {  
                    changeCount++  
                    try {  
                        toApply.fastForEach { composition -> toComplete.add(composition) }  
                        toApply.fastForEach { composition -> composition.applyChanges() }  
                    } catch (e: Throwable) {  
                        processCompositionError(e)  
                        clearRecompositionState()  
                        return@withFrameNanos  
                    } finally {  
                        toApply.clear()  
                    }  
                }  
  
                if (toLateApply.isNotEmpty()) {  
                    try {  
                        toComplete += toLateApply  
                        toLateApply.forEach { composition -> composition.applyLateChanges() }  
                    } catch (e: Throwable) {  
                        processCompositionError(e)  
                        clearRecompositionState()  
                        return@withFrameNanos  
                    } finally {  
                        toLateApply.clear()  
                    }  
                }  
  
                if (toComplete.isNotEmpty()) {  
                    try {  
                        toComplete.forEach { composition -> composition.changesApplied() }  
                    } catch (e: Throwable) {  
                        processCompositionError(e)  
                        clearRecompositionState()  
                        return@withFrameNanos  
                    } finally {  
                        toComplete.clear()  
                    }  
                }  
                synchronized(stateLock) { deriveStateLocked() }  
                Snapshot.notifyObjectsInitialized()  
                alreadyComposed.clear()  
                modifiedValues.clear()  
                compositionsRemoved = null  
            }  
        }        discardUnusedMovableContentState()  
    }  
}
```
1. awaitWorkAvailable 挂起等待可用重组信息
2. 
####  recompositionRunner
``` kotlin
private suspend fun recompositionRunner(  
    block: suspend CoroutineScope.(parentFrameClock: MonotonicFrameClock) -> Unit  
) {  
    val parentFrameClock = coroutineContext.monotonicFrameClock  
    withContext(broadcastFrameClock) {  
        val callingJob = coroutineContext.job  
        registerRunnerJob(callingJob)  
        val unregisterApplyObserver =  
            Snapshot.registerApplyObserver { changed, _ ->  
                synchronized(stateLock) {  
                    if (_state.value >= State.Idle) {  
                        val snapshotInvalidations = snapshotInvalidations  
                        changed.fastForEach {  
                            if (  
                                it is StateObjectImpl &&  
                                !it.isReadIn(ReaderKind.Composition)  
                            ) {  
                                return@fastForEach  
                            }  
                            snapshotInvalidations.add(it)  
                        }  
                        deriveStateLocked()  
                    } else null  
                }  
                    ?.resume(Unit)  
            }  
        addRunning(recomposerInfo)  
        try {  
            knownCompositions().fastForEach { it.invalidateAll() }  
            coroutineScope { block(parentFrameClock) }  
        } finally {  
            unregisterApplyObserver.dispose()  
            synchronized(stateLock) {  
                if (runnerJob === callingJob) {  
                    runnerJob = null  
                }  
                deriveStateLocked()  
            }  
            removeRunning(recomposerInfo)  
        }  
    }  
}
```
1. registerApplyObserver 注册applyObservers，当状态发生变化时 GlobalSnapshot 会通知此观察者；
# composing
``` kotlin
// Recomposer
private inline fun <T> composing(  
    composition: ControlledComposition,  
    modifiedValues: MutableScatterSet<Any>?,  
    block: () -> T  
): T {  
    val snapshot =  
        Snapshot.takeMutableSnapshot(  
            readObserverOf(composition),  
            writeObserverOf(composition, modifiedValues)  
        )  
    try {  
        return snapshot.enter(block)  
    } finally {  
        applyAndCheck(snapshot)  
    }  
}
```
1. 创建快照并在当前快照内进行组合，对当前快照发生的状态读写进行监听；
## recordReadOf
```kotlin
// Recomposer
private fun readObserverOf(composition: ControlledComposition): (Any) -> Unit {  
    return { value -> composition.recordReadOf(value) }  
}
// CompositionImpl
override fun recordReadOf(value: Any) {  
    if (!areChildrenComposing) {  
        composer.currentRecomposeScope?.let {  
            it.used = true  
            val alreadyRead = it.recordRead(value)  
            if (!alreadyRead) {  
                if (value is StateObjectImpl) {  
                    value.recordReadIn(ReaderKind.Composition)  
                }  
  
                observations.add(value, it)   
                if (value is DerivedState<*>) {  
                    val record = value.currentRecord  
                    derivedStates.removeScope(value)  
                    record.dependencies.forEachKey { dependency ->  
                        if (dependency is StateObjectImpl) {  
                            dependency.recordReadIn(ReaderKind.Composition)  
                        }  
                        derivedStates.add(dependency, value)  
                    }  
                    it.recordDerivedStateValue(value, record.currentValue)  
                }  
            }  
        }  
    }  
}
// Snapshot.kt
fun <T : StateRecord> T.readable(state: StateObject): T {  
    val snapshot = Snapshot.current  
    snapshot.readObserver?.invoke(state)  
    return readable(this, snapshot.snapshotId, snapshot.invalid)  
        ?: sync {  
            val syncSnapshot = Snapshot.current  
            readable(state.firstStateRecord as T, syncSnapshot.snapshotId, syncSnapshot.invalid)  
                ?: readError()  
        }  
}


```
1. value.recordReadIn 标记当前状态参与了组合，重组时用于判断是否需要修改；
# Composable
1. @Composable 修饰的函数经过编译后，会添加Composer和int类型的2个参数
## startRestartGroup
``` kotlin
override fun startRestartGroup(key: Int): Composer {  
    startReplaceGroup(key)  
    addRecomposeScope()  
    return this  
}
```
1. startReplaceGroup: 插入Group到Group数组
2. addRecomposeScope：插入RecomposeScopeImpl到Slot数组
## updateScope
``` kotlin
override fun updateScope(block: (Composer, Int) -> Unit) { this.block = block }
```
1. 更新重组范围函数block，重组时会自动根据是否修改来决定调用此block函数