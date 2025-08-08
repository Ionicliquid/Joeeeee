# KMP源码
## 背景

KMP 中以 Compose 为基础来构建跨平台复用的 UI，Compose 是响应式 UI 体系，Compose 是如何封装状态管理驱动 UI 刷新渲染的呢？本文简要介绍 Compose 源码的核心思想和架构，以便对 KMP 跨平台的设计思路和演进路线提供一些知识储备。
起源 (Android Compose): Compose 最初是 Google 发起的一个项目，旨在彻底革新 Android 的 UI 开发模式，对标 Flutter 和 SwiftUI 的声明式范式。在这个阶段，大家普遍称之为 “Android Compose”，因为它完全是为了 Android 平台而生。
官方品牌 (Jetpack Compose): 随着项目成熟并准备正式发布，Google 将其纳入了 Android Jetpack 库套件中。Jetpack 是一系列旨在帮助开发者遵循最佳实践、减少样板代码并编写在不同 Android 版本和设备上表现一致的代码的库。因此，官方名称正式定为  “Jetpack Compose” ，强调了它作为 Android 官方推荐的现代 UI 工具包的地位。
走向跨平台 (Compose Multiplatform): Compose 的核心模块（如 runtime, compiler, ui）从设计之初就具有良好的平台无关性。JetBrains（Kotlin 语言的创造者）看到了这一点，并基于这个核心，扩展开发了支持其他平台的版本，这就是  “Compose Multiplatform” (CMP) 。它复用了 Jetpack Compose 的核心，但为其添加了桌面 (Desktop)、iOS 和 Web 的渲染后端。

  

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vPzQMNwryr9YC2XwibVK2zEbpibFkqOBFicCCRG3QT7DDeHcBUB1iaibHBow/640?from=appmsg&randomid=yyfddaln&tp=webp&wxfrom=5&wx_lazy=1)

具体看两者差异：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vAnicYsg9x7XCztp63H3uMHpXd4Tib5S6qLRCgpR62fXKRYpEia4r9rCIA/640?wx_fmt=png&from=appmsg&randomid=mkc90tc2&tp=webp&wxfrom=5&wx_lazy=1)

KMP 如何解决代码复用问题

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vpaj9nwVicGmh0yk3WGpicoJJRgItGYXWDtwgEhmbicGvOyxxaON6JLqgA/640?from=appmsg&randomid=kcksdvz8&tp=webp&wxfrom=5&wx_lazy=1)
KMP 生态在不断完善

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vzlbgibSTZWfb0qJtep14VmXGfd2D3JlaxFrda5e5YeFZDVGrap6X6ZA/640?from=appmsg&randomid=m2yvriq9&tp=webp&wxfrom=5&wx_lazy=1)


## 概览

### 1 流程介绍

整体上 Compose 主要包含了几个重要的过程：组合和重组、状态管理、UI 构建、绘制

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vbb3BWicby1mpLt7aO0uHbgibz81ib2LKGymdvkPYeatkjWUuToOT85ftw/640?wx_fmt=png&from=appmsg&randomid=ws5jfy9u&tp=webp&wxfrom=5&wx_lazy=1)

具体核心过程涉及到的源码位置：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vQRuSG9WE58U9cquKPsNr6OU69nPDsgm3UlNjNibn0e5xD3dVDyOYDoQ/640?wx_fmt=png&from=appmsg&randomid=ba1jyb82&tp=webp&wxfrom=5&wx_lazy=1)

  

UI 布局代码以一个计数器例子来举例说明每一帧绘制流程，从点击 +，触发 State value++ 开始

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vInJCaoCia8NE6e9UHU3Gib6LhGSkVPGicic4icqyAa3pgPiaarJWhDqKx7CA/640?from=appmsg&randomid=ijazgqkm&tp=webp&wxfrom=5&wx_lazy=1)

  

绘制的核心流程：重组-> 布局 -> 绘制

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35v5aJGRYymmZ42JkTU0xgoDXIEFnV9ynP4BJmcib3njIzERTboh3Gxxew/640?from=appmsg&randomid=jwjqvfil&tp=webp&wxfrom=5&wx_lazy=1)

  

### 2 代码架构

整体的源码类图结构：
- 状态管理
- 重组
- 布局和绘制
    

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35v5Xia1GFPyS6sTqhyOxneKLuHZZM4jia8M6tiazspIpwh0KLPC1vFPebrQ/640?from=appmsg&randomid=q5oceyvt&tp=webp&wxfrom=5&wx_lazy=1)

  

其中状态管理相关的类结构：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vMbZdg3kvhic6EnAsty8s1ccMzg1kLPjUD80qbFRe9vicvyABeOPLpibFQ/640?from=appmsg&randomid=fuo0tdt6&tp=webp&wxfrom=5&wx_lazy=1)

  

重组触发流程的类结构：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vUMOqy2LDB96TVg21reWNJouaZcrYfKIyh1OibGAvN5hdQfuagavIOjA/640?from=appmsg&randomid=cmerjf2z&tp=webp&wxfrom=5&wx_lazy=1)

  

布局和绘制的类结构：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vohjSpuKKoicqHqp09yJzhb8MPTWWCLxRGd0mkgygAIH1xC7cPmjf8xQ/640?from=appmsg&randomid=96ff2sx6&tp=webp&wxfrom=5&wx_lazy=1)

  

## 编译

编译阶段目标是方便状态管理，将 UI 组合函数中植入一些串联管理组合函数的角色，方便进行 UI 构建和状态管理。
开发阶段代码：
``` kotlin
@Composable  
fun ConditionalUI(showProfile: Boolean) {  
    Column {  
        Text("Header")  
        if (showProfile) {  
            ProfileView() // 分支 A  
        } else {  
            LoginPrompt() // 分支 B  
        }  
        Text("Footer")  
    }  
}
```


编译之后伪代码：

```kotlin
fun ConditionalUI(showProfile: Boolean, $composer: Composer, $changed: Int) {  
    // 1. 开始一个可重组的作用域 (Restart Group)  
    //    这使得 ConditionalUI 自身可以被单独重组  
    $composer.startRestartGroup(12345) // 12345 是一个编译器生成的唯一键  
  
    // 检查参数是否有变化，这是智能重组和跳过的关键  
    // (这是一个简化的逻辑)  
    val dirty = $changed  
    if (dirty and0b1011 == 0b0010 && $composer.skipping) {  
        $composer.skipToGroupEnd() // 如果参数未变且父级在跳过，则直接跳过整个函数  
    } else {  
        // 2. 调用 Column Composable  
        //    注意，$composer 和 $changed 参数会被传递下去  
        Column(modifier = ..., $composer, $changed, ...) {  
  
            // 3. 调用 Text("Header")  
            //    由于 "Header" 是一个常量，编译器会标记它为静态，进一步优化  
            Text("Header", $composer, 0b0110)  
  
            // 4. 【核心】处理 if/else 控制流  
            //    编译器为 if 语句创建一个匿名组  
            $composer.startGroup(67890) // 67890 是 if 语句的唯一键  
            if (showProfile) {  
                // 5. 为 if 的 true 分支创建一个组  
                $composer.startGroup(22222)  
                ProfileView($composer, 0)  
                $composer.endGroup()  
            } else {  
                // 6. 为 if 的 false 分支创建一个组  
                $composer.startGroup(33333)  
                LoginPrompt($composer, 0)  
                $composer.endGroup()  
            }  
            $composer.endGroup() // 结束 if 语句的组  
  
            // 7. 调用 Text("Footer")  
            Text("Footer", $composer, 0b0110)  
        }  
    }  
  
    // 8. 结束可重组作用域，并提供一个用于重组的 lambda  
    val scope = $composer.endRestartGroup()  
    scope?.updateScope { nextComposer, force ->  
        // 当需要重组时，这个 lambda 会被调用  
        ConditionalUI(showProfile, nextComposer, $changed or0b1)  
    }  
}

```

  

主要工作如下：

- Composer 编译器会在 Compose 函数中增加 Composer 和 changed 参数，方便 Composer 来管理所有的 UI 层级和状态，Composer 会将这些信息写入 SlotTable
- Key 的生成与作用：composer.startRestartGroup(key) 中的 key 至关重要。编译器会根据 Composable 在源码中的 位置 （行号、列号）生成一个在当前父函数中唯一的、稳定的整数 Key。当 if/else 或 when 导致某个 Composable 在下一次重组中“消失”时，Composer 正是依靠这个 Key 来识别出“哪个组不见了”，从而高效地从 SlotTable 中移除对应的节点，而不需要进行复杂的树比对。
- $stable  标记 ：编译器会进行深入的类型分析。如果一个类的所有公共属性都是 val 且类型也是稳定的（基本类型、String、或也标记为 @Stable/@Immutable 的类），编译器会自动将其标记为“稳定”。在计算 changed 位图时，如果一个参数是稳定的且其实例引用没有变（===），Compose 就可以假定其内容也没有变，从而获得一个巨大的性能优化—— 直接跳过对该参数的深入比对 。
- Lambda 的“记住”与重写：对于 Composable 中的 Lambda 表达式，尤其是事件回调（如 onClick），编译器会特殊处理。它会隐式地将 Lambda 包裹在一个 remember 调用中，并捕获其依赖的变量。例如：
``` kotlin
    var name by remember { mutableStateOf("World") }Button(onClick = { println("Hello, $name") }) { /*...*/ }
```
    onClick 这个 Lambda 依赖于 name。编译器会生成类似这样的代码：
``` kotlin 
// 伪代码
val onClickLambda = remember(name) { { println("Hello, $name") } }  
Button(onClick = onClickLambda, ...)
```
 这意味着，只有当 name 发生变化时，才会重新创建一个新的 Lambda 实例。如果 name 不变，Button 在重组时会收到一个完全相同的 Lambda 实例。由于 Function 类型在 Compose 中被认为是稳定的，这使得 Button 可以跳过重组，因为它所有的参数都没有变化。

## 状态管理 

### 1 UI 组织方式

Compose 中 UI 结构构建十分高效将 UI 树压平到一个数组结构 SlotTable 中，其中通过一个数组 slots 完成所有 UI 相关信息的存储，通过一个 groups 数组完成所有的 UI 区块的位置范围。
- groups  数组是导航地图 ：Composer 依靠 groups 数组来理解代码的结构，特别是像 if-else 这样的条件分支。它知道上次走的是哪条路。
- slots  数组是数据仓库 ：所有真实的数据都在这里。当分支切换时，旧分支的数据被移除，新分支的数据被添加。
这样随着 compose 函数的执行就可以通过 groups 的索引机制进行 UI 的构建或者重组，或者执行 UI 的刷新。

  

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vAV6EPADTnBd7jA7J48AiaFiaKicOApF68ia9W9Pmn1CdFEuUYn1cUCczRQ/640?from=appmsg&randomid=avjoegrz&tp=webp&wxfrom=5&wx_lazy=1)

  

布局代码

```
@Composablefun AuthScreen() {    var showLogin by remember { mutableStateOf(true) }    Column {        if (showLogin) {            Button(onClick = { showLogin = false }) {                Text("Login")            }        } else {            Button(onClick = { showLogin = true }) {                Text("Logout")            }        }        Text("Welcome to the App!")    }}
```

  

SlotTable

```
// groups 数组记录了结构groups: [AuthScreen, Column, IF_GROUP, Button, Text, Text]                                  ^-- 标记这里是 if// slots 数组存储了实际数据slots: [state(true), LNode(Col), onClick, LNode(Btn), "Login", LNode(Txt), "Welcome...", LNode(Txt)]
```

  

UI 树

```
LayoutNode(Column)  ├── LayoutNode(Button)  │   └── LayoutNode(Text, text="Login")  └── LayoutNode(Text, text="Welcome to the App!")
```

  

A SlotTable

为了实现高效的插入和删除，SlotTable 的底层数组实际上使用了一种源自文本编辑器的优化技术—— Gap Buffer 。

  

- 工作原理：想象一个巨大的数组，Composer 的当前插入点就像文本光标。当需要在光标处插入新内容时，它不是移动光标之后的所有数据，而是在光标位置创建一个“间隙”（Gap）。插入操作就在这个间隙中进行。当光标需要移动时，只需要移动间隙两端的数据来“填充”旧的间隙，并在新位置创建间隙。对于连续的 Composable 调用，光标（即 Composer 的内部指针）顺序移动，插入效率极高。
    
- 优势：这种机制使得在 SlotTable 中间插入或删除一组节点（例如，if 条件从 false 变为 true）的成本，与这组节点的数量成正比，而与 SlotTable 的总大小无关。这是 Compose 能够高效处理动态 UI（如长列表）的基石之一。
    

  

Gap 的设置使得 UI 变化区域存在一段空间，可以使得插入或者删除 UI 节点变得非常高效，无需移动整个数组即可完成，比如在 A B GAP GAP C D 中插入 X，只需要将 GAP 起始点位置插入 X 即可。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vOeRgdMhmxeFhGsINdxWu1SVEukrGQXycTPP3jjA1jZKfAnqX5LnO8w/640?from=appmsg&randomid=e1509hj0&tp=webp&wxfrom=5&wx_lazy=1)

  

### 2 状态管理

整个 UI 状态管理机制由如下重要角色构成
- Recomposer：项目总调度/发动机
- Composition  (  CompositionImpl )：整个建筑工程/UI 树实例
- Composer  (  ComposerImpl )：具体的建筑工人/UI 描述翻译器
- Composable 函数：建筑指令/UI 蓝图片段
State  改变 → 重组 Composable → 更新   SlotTable   → 精确修改   LayoutNode  树 → 触发重新布局和绘制

  

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vgoDQ0ntEPNWicg0NmmoicZ4I44tI5wxCWhhZ6iaHk079DDw2LRibW717ibQ/640?from=appmsg&randomid=tfnfbkgd&tp=webp&wxfrom=5&wx_lazy=1)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vMpWic7Bs4ia3FcBZatjE3lfXhmAfOw28U92jCaVroH7kgKKwsII6nwmg/640?wx_fmt=png&from=appmsg&randomid=n0mbmpqe&tp=webp&wxfrom=5&wx_lazy=1)

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35valzIZvvMg03JbIIUWVKRQ3rQQy86qZSWFmaruONjna0ib2bE95nV7ow/640?wx_fmt=png&from=appmsg&randomid=cw029ysq&tp=webp&wxfrom=5&wx_lazy=1)
整个 Compose 的核心在于状态变量驱动 UI 重组，具体从代码分析流程如下
#### A 状态修改
GlobalSnapshotManager 注册全局 UI 刷新事件的监听

``` kotlin
internal object GlobalSnapshotManager {  
    private val started = AtomicBoolean(false)  
  
    fun ensureStarted() {  
        if (started.compareAndSet(false, true)) {  
            val channel = Channel<Unit>(Channel.CONFLATED)  
            CoroutineScope(AndroidUiDispatcher.Main).launch {  
            // 状态变量修改后驱动重组和应用  
                channel.consumeEach {  
                    Snapshot.sendApplyNotifications()  
                }  
            }  
            // 状态变量的修改  
            Snapshot.registerGlobalWriteObserver {  
                channel.trySend(Unit)  
            }  
        }  
    }  
}
```

  

状态管理代理状态变量的 Set 和 Get 实现对状态变量读写的监听，SnapshotMutableStateImpl

```kotlin
internal open class SnapshotMutableStateImpl<T>(  
    value: T,  
    override val policy: SnapshotMutationPolicy<T>  
) : StateObject, SnapshotMutableState<T> {  
    @Suppress("UNCHECKED_CAST")  
    override var value: T  
        get() = next.readable(this).value  
        set(value) = next.withCurrent {  
            if (!policy.equivalent(it.value, value)) {  
                next.overwritable(this, it) { this.value = value }  
            }  
        }
```

```kotlin
internal inline fun <T : StateRecord, R> T.overwritable(  
    state: StateObject,  
    candidate: T,  
    block: T.() -> R  
): R {  
    var snapshot: Snapshot = snapshotInitializer  
    return sync {  
        snapshot = Snapshot.current  
        this.overwritableRecord(state, snapshot, candidate).block()  
    }.also {  
        notifyWrite(snapshot, state)  
    }  
}
```

记录状态变量修改，后续通知全局的状态 SnapShot 管理，将这些作为 Changed 传入

``` kotlin
override fun recordModified(state: StateObject) {  
    (modified ?: IdentityArraySet<StateObject>().also { modified = it }).add(state)  
}
```

Snapshot 通知状态变量修改触发全局的响应状态变量更新的流程

``` kotlin
internal fun notifyWrite(snapshot: Snapshot, state: StateObject) {  
    snapshot.writeCount += 1  
    snapshot.writeObserver?.invoke(state)  
}
```

GlobalSnapshotManager

``` kotlin
Snapshot.registerGlobalWriteObserver {  
    channel.trySend(Unit)  
}
```

``` kotlin
CoroutineScope(AndroidUiDispatcher.Main).launch {  
    channel.consumeEach {  
        Snapshot.sendApplyNotifications()  
    }  
}
```

监测到状态变量修改，Snapshot 更新启动

``` kotlin
fun sendApplyNotifications() {  
    val changes = sync {  
        currentGlobalSnapshot.get().modified?.isNotEmpty() == true  
    }  
    if (changes)  
        advanceGlobalSnapshot()  
}
```

SnapshotKt 通知状态观测者 ReComposer 准备开始调度响应状态变量的修改

``` kotlin
private fun <T> advanceGlobalSnapshot(block: (invalid: SnapshotIdSet) -> T): T {  
    var previousGlobalSnapshot = snapshotInitializer as GlobalSnapshot  
  
    var modified: IdentityArraySet<StateObject>? = null// Effectively val; can be with contracts  
    val result = sync {  
        previousGlobalSnapshot = currentGlobalSnapshot.get()  
        modified = previousGlobalSnapshot.modified  
        if (modified != null) {  
            pendingApplyObserverCount.add(1)  
        }  
        takeNewGlobalSnapshot(previousGlobalSnapshot, block)  
    }  
  
    // If the previous global snapshot had any modified states then notify the registered apply  
    // observers.  
    modified?.let {  
        try {  
            val observers: List<(Set<Any>, Snapshot) -> Unit> =  
                sync { applyObservers.toMutableList() }  
            observers.fastForEach { observer ->  
                observer(it, previousGlobalSnapshot)  
            }  
        } finally {  
            pendingApplyObserverCount.add(-1)  
        }  
    }  
  
    sync {  
        checkAndOverwriteUnusedRecordsLocked()  
        modified?.fastForEach { processForUnusedRecordsLocked(it) }  
    }  
  
    return result  
}

```

Recomposer 开始调度全局工作，状态变量的修改被作为 Changed 传入 于 recordModified 记录修改的状态变量

``` kotlin
@OptIn(ExperimentalComposeApi::class)  
private suspend fun recompositionRunner(  
    block: suspend CoroutineScope.(parentFrameClock: MonotonicFrameClock) -> Unit  
) {  
    val parentFrameClock = coroutineContext.monotonicFrameClock  
    withContext(broadcastFrameClock) {  
        // Enforce mutual exclusion of callers; register self as current runner  
        val callingJob = coroutineContext.job  
        registerRunnerJob(callingJob)  
  
        // Observe snapshot changes and propagate them to known composers only from  
        // this caller's dispatcher, never working with the same composer in parallel.  
        // unregisterApplyObserver is called as part of the big finally below  
        val unregisterApplyObserver = Snapshot.registerApplyObserver { changed, _ ->  
            synchronized(stateLock) {  
                if (_state.value >= State.Idle) {  
                    snapshotInvalidations.addAll(changed)  
                    deriveStateLocked()  
                } elsenull  
            }?.resume(Unit)  
        }  
  
        addRunning(recomposerInfo)  
  
        try {  
            // Invalidate all registered composers when we start since we weren't observing  
            // snapshot changes on their behalf. Assume anything could have changed.  
            synchronized(stateLock) {  
                knownCompositions.toMutableList()  
            }.fastForEach { it.invalidateAll() }  
  
            coroutineScope {  
                block(parentFrameClock)  
            }  
        } finally {  
            unregisterApplyObserver.dispose()  
            synchronized(stateLock) {  
                if (runnerJob === callingJob) {  
                    runnerJob = null  
                }  
                deriveStateLocked()  
            }  
            removeRunning(recomposerInfo)  
        }  
    }  
}
```
#### B 重组

状态变量修改后，已经记录 snapshotInvalidations 后续即可在 Vsync 周期内将 snapshotInvalidations 批量执行状态变量关联的 UI 的更新，这个过程就是重组。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vYT0hyFkB8LiaGkmA2MQ3hVauYPhxHmJJslhNF71OHIR3zsOVGAmiaDXg/640?from=appmsg&randomid=hjk6h1yj&tp=webp&wxfrom=5&wx_lazy=1)

  

- 重组的本质：当状态（State）改变时，Recomposer 会调度受该状态影响的 Composable 函数重新执行。这个过程就是“重组”。
    
- SlotTable  的角色 ：重组的主要产物是 更新   SlotTable 。Composer 会将 Composable 函数的新执行结果与 SlotTable 中记录的旧结果进行比较。
    
- SlotTable  的变化 ：比较的结果是 SlotTable 中对应的数据被更新、标记为删除，或者插入了新的数据。
    

  

重要的是，到这一步为止，真实的 UI 节点树（由 LayoutNode  构成）还没有发生任何变化。  SlotTable  只是一个更新后的“蓝图”或“指令集”。

  

Vsync 响应驱动重组 runRecomposeAndApplyChanges 在 window 创建时，增加到 Vsync 的 callback 中

```
suspend fun runRecomposeAndApplyChanges() = recompositionRunner { parentFrameClock ->    while (shouldKeepRecomposing) {        awaitWorkAvailable()        // Don't await a new frame if we don't have frame-scoped work        if (!recordComposerModifications()) continue}
```

```
private fun recordComposerModifications(): Boolean {    val changes = synchronized(stateLock) {        if (snapshotInvalidations.isEmpty()) return hasFrameWorkLocked        snapshotInvalidations.also { snapshotInvalidations = IdentityArraySet() }    }    val compositions = synchronized(stateLock) {        knownCompositions.toMutableList()    }    var complete = false    try {        run {            compositions.fastForEach { composition ->                composition.recordModificationsOf(changes)}
```

来到具体的  compositions 保存之前保存的修改的状态变量

```
override fun recordModificationsOf(values: Set<Any>) {    while (true) {        val old = pendingModifications.get()        val new: Any = when (old) {            null, PendingApplyNoModifications -> values            isSet<*> -> arrayOf(old, values)            isArray<*> -> (oldasArray<Set<Any>>) + values            else -> error("corrupt pendingModifications: $pendingModifications")        }        if (pendingModifications.compareAndSet(old, new)) {            if (old == null) {                synchronized(lock) {                    drainPendingModificationsLocked()                }            }            break        }    }}
```

Recomposer 将该 Composition 记录到 compositionInvalidations 中，标脏操作，便于后续执行

```
internal override fun invalidate(composition: ControlledComposition) {    synchronized(stateLock) {        if (composition !in compositionInvalidations) {            compositionInvalidations += composition            deriveStateLocked()        } else null    }?.resume(Unit)}
```

Vsync callback 中执行重组流程：

```
trace("Recomposer:recompose") {    // Drain any composer invalidations from snapshot changesand record    // composers to work on    recordComposerModifications()    synchronized(stateLock) {        compositionInvalidations.fastForEach {            toRecompose += it        }        compositionInvalidations.clear()    }    // Perform recomposition for any invalidated composers    val modifiedValues = IdentityArraySet<Any>()    val alreadyComposed = IdentityArraySet<ControlledComposition>()    while (toRecompose.isNotEmpty() || toInsert.isNotEmpty()) {        try {            toRecompose.fastForEach { composition ->                alreadyComposed.add(composition)                performRecompose(composition, modifiedValues)?.let {                    toApply += it                }            }        } catch (e: Exception) {            processCompositionError(e, recoverable = true)            clearRecompositionState()            return@withFrameNanos        } finally {            toRecompose.clear()        }
```

Composer doCompose 具体执行重组逻辑如果不设计新 UI 节点插入则 skipCurrentGroup

```
private fun doCompose(    invalidationsRequested: IdentityArrayMap<RecomposeScopeImpl, IdentityArraySet<Any>?>,    content: (@Composable () -> Unit)?) {    runtimeCheck(!isComposing) { "Reentrant composition is not supported" }    trace("Compose:recompose") {        compositionToken = currentSnapshot().id        providerUpdates.clear()        invalidationsRequested.forEach { scope, set ->            val location = scope.anchor?.location ?: return            invalidations.add(Invalidation(scope, location, set))        }        invalidations.sortBy { it.location }        nodeIndex = 0        var complete = false        isComposing = true        try {            startRoot()            // vv Experimental for forced            @Suppress("UNCHECKED_CAST")            val savedContent = nextSlot()            if (savedContent !== content && content != null) {                updateValue(content as Any?)            }            // ^^ Experimental for forced            // Ignore reads of derivedStateOf recalculations            observeDerivedStateRecalculations(derivedStateObserver) {                if (content != null) {                    startGroup(invocationKey, invocation)                    invokeComposable(this, content)                    endGroup()                } elseif (                    (forciblyRecompose || providersInvalid) &&                    savedContent != null &&                    savedContent != Composer.Empty                ) {                    startGroup(invocationKey, invocation)                    @Suppress("UNCHECKED_CAST")                    invokeComposable(this, savedContent as @Composable () -> Unit)                    endGroup()                } else {                    skipCurrentGroup()                }            }            endRoot()            complete = true        } finally {            isComposing = false            invalidations.clear()            if (!complete) abortRoot()        }    }}
```

skipCurrentGroup 会执行 recomposeToGroupEnd，其中会调用 compose 具体执行状态变量关联的 composble 函数

```
private fun recomposeToGroupEnd() {    var firstInRange = invalidations.firstInRange(reader.currentGroup, end)    while (firstInRange != null) {        val location = firstInRange.location        invalidations.removeLocation(location)        if (firstInRange.isInvalid()) {            recomposed = true            reader.reposition(location)            val newGroup = reader.currentGroup            // Record the changes to the applier location            recordUpsAndDowns(oldGroup, newGroup, parent)            oldGroup = newGroup            // Calculate the node index (the distance index in the node this groups nodes are            // located in the parent node).            nodeIndex = nodeIndexOf(                location,                newGroup,                parent,                recomposeIndex            )            // Calculate the compound hash code (a semi-unique code for every group in the            // composition used to restore saved state).            compoundKeyHash = compoundKeyOf(                reader.parent(newGroup),                parent,                recomposeCompoundKey            )            // We have moved so the cached lookup of the provider is invalid            providerCache = null            // Invoke the scope's composition function            firstInRange.scope.compose(this)            // We could have moved out of a provider so the provider cache is invalid.            providerCache = null            // Restore the parent of the reader to the previous parent            reader.restoreParent(parent)        } else {            // If the invalidation is not used restore the reads that were removed when the            // the invalidation was recorded. This happens, for example, when on of a derived            // state's dependencies changed but the derived state itself was not changed.            invalidateStack.push(firstInRange.scope)            firstInRange.scope.rereadTrackedInstances()            invalidateStack.pop()        }        // Using slots.current here ensures composition always walks forward even if a component        // before the current composition is invalidated when performing this composition. Any        // such components will be considered invalid for the next composition. Skipping them        // prevents potential infinite recomposes at the cost of potentially missing a compose        // as well as simplifies the apply as it always modifies the slot table in a forward        // direction.        firstInRange = invalidations.firstInRange(reader.currentGroup, end)    }}
```

ComposableLambdaImpl 重组执行方法，根据结果来判断 compose 是否有需要更新

```
override operator fun invoke(p1: Any?, c: Composer, changed: Int): Any? {    val c = c.startRestartGroup(key)    trackRead(c)    val dirty = changed orif (c.changed(this)) differentBits(1) else sameBits(1)    val result = (        _block as (            p1: Any?,            c: Composer,            changed: Int        ) -> Any?        )(        p1,        c,        dirty    )    c.endRestartGroup()?.updateScope { nc, _ ->        this(p1, nc, updateChangedFlags(changed) or0b1)    }    return result}
```

Composer updateValue 有修改则记录到 observer 中后续 apply change 回调这里

```
internal fun updateValue(value: Any?) {    if (inserting) {        writer.update(value)        if (valueis RememberObserver) {            record { _, _, rememberManager -> rememberManager.remembering(value) }            abandonSet.add(value)        }    } else {        val groupSlotIndex = reader.groupSlotIndex - 1        if (valueis RememberObserver) {            abandonSet.add(value)        }        // Composition apply changes        recordSlotTableOperation(forParent = true) { _, slots, rememberManager ->            if (valueis RememberObserver) {                rememberManager.remembering(value)            }            when (val previous = slots.set(groupSlotIndex, value)) {                is RememberObserver ->                    rememberManager.forgetting(previous)                is RecomposeScopeImpl -> previous.release()            }        }    }}
```

  

#### C 状态更新

以一个具体例子来演示状态的写入流程：

```
@Composablefun ConditionalUI(showProfile: Boolean) {    Column {        Text("Header")        if (showProfile) {            ProfileView() // 分支 A        } else {            LoginPrompt() // 分支 B        }        Text("Footer")    }}
```

用序列图来可视化 ConditionalUI 在 showProfile 从 true 变为 false 时的重组过程。

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vcSV3YVXIUOZs0WrnkeGO8TdmdNYzWE4ZUtlgasKUUcFgwwqaA8MWAw/640?from=appmsg&randomid=e3wdv3f5&tp=webp&wxfrom=5&wx_lazy=1)

  

以插入一个 UI 节点为例子：

```
internal fun updateValue(value: Any?) {    if (inserting) {        writer.update(value)        if (valueis RememberObserver) {            record { _, _, rememberManager -> rememberManager.remembering(value) }            abandonSet.add(value)        }    } else {        val groupSlotIndex = reader.groupSlotIndex - 1        if (valueis RememberObserver) {            abandonSet.add(value)        }        recordSlotTableOperation(forParent = true) { _, slots, rememberManager ->            if (valueis RememberObserver) {                rememberManager.remembering(value)            }            when (val previous = slots.set(groupSlotIndex, value)) {                is RememberObserver ->                    rememberManager.forgetting(previous)                is RecomposeScopeImpl -> previous.release()            }        }    }}
```

SlotWriter update 方法

```
fun update(value: Any?): Any? {    val result = skip()    set(value)    return result}
```

```
fun skip(): Any? {    if (insertCount > 0) {        insertSlots(1, parent)    }    return slots[dataIndexToDataAddress(currentSlot++)]}
```

具体 insert 过程

```
private fun insertSlots(size: Int, group: Int) {    if (size > 0) {        moveSlotGapTo(currentSlot, group)        val gapStart = slotsGapStart        var gapLen = slotsGapLen        if (gapLen < size) {            val slots = slots            // Create a bigger gap            val oldCapacity = slots.size            val oldSize = oldCapacity - gapLen            // Double the size of the array, but at least MinGrowthSize and >= size            val newCapacity = max(                max(oldCapacity * 2, oldSize + size),                MinSlotsGrowthSize            )            val newData = Array<Any?>(newCapacity) { null }            val newGapLen = newCapacity - oldSize            val oldGapEndAddress = gapStart + gapLen            val newGapEndAddress = gapStart + newGapLen            // Copy the old arrays into the new arrays            slots.copyInto(                destination = newData,                destinationOffset = 0,                startIndex = 0,                endIndex = gapStart            )            slots.copyInto(                destination = newData,                destinationOffset = newGapEndAddress,                startIndex = oldGapEndAddress,                endIndex = oldCapacity            )            // Update the gap and slots            this.slots = newData            gapLen = newGapLen        }        val currentDataEnd = currentSlotEnd        if (currentDataEnd >= gapStart) this.currentSlotEnd = currentDataEnd + size        this.slotsGapStart = gapStart + size        this.slotsGapLen = gapLen - size    }}
```

Composition compose 执行结束会 apply 到 snapshot 

```
private inline fun <T> composing(    composition: ControlledComposition,    modifiedValues: IdentityArraySet<Any>?,    block: () -> T): T {    val snapshot = Snapshot.takeMutableSnapshot(        readObserverOf(composition), writeObserverOf(composition, modifiedValues)    )    try {        return snapshot.enter(block)    } finally {        applyAndCheck(snapshot)    }}
```

触发 snaptshot 更新

```
private fun applyAndCheck(snapshot: MutableSnapshot) {    try {        val applyResult = snapshot.apply()        if (applyResult is SnapshotApplyResult.Failure) {            error(                "Unsupported concurrent change during composition. A state object was " +                    "modified by composition as well as being modified outside composition."            )            // TODO(chuckj): Consider lifting this restriction by forcing a recompose        }    } finally {        snapshot.dispose()    }}
```

  

#### D UI 更新

在组合阶段的末尾，Composer 会根据 SlotTable 的变化，对 LayoutNode 树执行精确的“外科手术式”操作：

- 更新 (Update)：如果一个 Composable 只是参数变了（例如 Text 的文本内容），Composer 会从 SlotTable 中找到对应的、 已经存在 的 LayoutNode，并调用其 update 方法来更新属性。节点本身被复用，不会重新创建。
- 插入 (Insert)：如果出现了一个新的 Composable（例如 if 条件从 false 变为 true），Composer 会执行其 emit 指令，创建一个 新的  LayoutNode 实例，并将其插入到 LayoutNode 树的正确位置。
- 删除 (Remove)：如果一个 Composable 不再被调用，Composer 会找到它对应的 LayoutNode，并将其从树中移除。
- 移动 (Move)：如果使用了 key，并且列表项的顺序发生了变化，Composer 能够识别出这种情况，并直接在 LayoutNode 树中 移动 现有的节点到新的位置，而不是销毁再重建它们。
Vsync callback 中在执行重组之后，会将重组后的结果使用 Applier 更新到 LayoutNode

``` kotlin
if (toApply.isNotEmpty()) {  
    changeCount++  
  
    // Perform apply changes  
    try {  
        // We could do toComplete += toApply but doing it like below  
        // avoids unncessary allocations since toApply is a mutable list  
        // toComplete += toApply  
        toApply.fastForEach { composition->  
            toComplete.add(composition)  
        }  
        toApply.fastForEach { composition->  
            composition.applyChanges()  
        }  
    } catch (e: Exception) {  
        processCompositionError(e)  
        clearRecompositionState()  
        return@withFrameNanos  
    } finally {  
        toApply.clear()  
    }  
}
```

Apply changes，根据 SlotWriter 在重组阶段记录的需要变更的 slots

```kotlin
private fun applyChangesInLocked(changes: MutableList<Change>) {  
    val manager = RememberEventDispatcher(abandonSet)  
    try {  
        if (changes.isEmpty()) return  
        trace("Compose:applyChanges") {  
            applier.onBeginChanges()  
  
            // Apply allchanges  
            slotTable.write { slots ->  
                val applier = applier  
                changes.fastForEach { change ->  
                    change(applier, slots, manager)  
                }  
                changes.clear()  
            }  
            applier.onEndChanges()  
        }  
  
        // Side effects run after lifecycle observers so that any remembered objects  
        // that implement RememberObserver receive onRemembered before a side effect  
        // that captured it and operates on it can run.  
        manager.dispatchRememberObservers()  
        manager.dispatchSideEffects()  
  
        if (pendingInvalidScopes) {  
            trace("Compose:unobserve") {  
                pendingInvalidScopes = false  
                observations.removeValueIf { scope -> !scope.valid }  
                cleanUpDerivedStateObservations()  
            }  
        }  
    } finally {  
        // Only dispatch abandons if we do not have any late changes. The instances in the  
        // abandon set can be remembered in the late changes.  
        if (this.lateChanges.isEmpty())  
            manager.dispatchAbandons()  
    }  
}
```

这里的 applyChange 会执行状态变量关联的操作，比如 Modifier 的赋值

```kotlin
override var modifier: Modifier = Modifier  
    set(value) {  
        require(!isVirtual || modifier === Modifier) {  
            "Modifiers are not supported on virtual LayoutNodes"  
        }  
        field = value  
        nodes.updateFrom(value)  
        layoutDelegate.updateParentData()  
        if (nodes.has(Nodes.IntermediateMeasure)) {  
            if (lookaheadRoot == null) {  
                lookaheadRoot = this  
            }  
        }  
    }
```

Composer 中调度 Applier 完成 LayoutNode 插入的例子：

```kotlin
// Insert nodes if necessary  
if (nodesToInsert.isNotEmpty()) {  
    record { applier, _, _ ->  
        val base = effectiveNodeIndex  
        @Suppress("UNCHECKED_CAST")  
        nodesToInsert.fastForEachIndexed { i, node ->  
            applier as Applier<Any?>  
            applier.insertBottomUp(base + i, node)  
            applier.insertTopDown(base + i, node)  
        }  
    }  
    if (to.slotTable == slotTable) {  
        // Inserting the contentinto the current slot tablethen we need to  
        // update the virtual node counts. Otherwise, we are inserting into  
        // a new slot table which is being created, notupdated, so the virtual  
        // node counts donot need to be updated.  
        val group = slotTable.anchorIndex(anchor)  
        updateNodeCount(  
            group,  
            updatedNodeCount(group) + nodesToInsert.size  
        )  
    }  
}
```

## 布局

布局部分和具体的 UI 节点相关，不同平台实现大同小异不再冗余阐述。

- 触发布局：一旦 LayoutNode 树被修改（无论是属性更新、插入、删除还是移动），这些被影响的节点及其父节点会被标记为“需要重新布局”。
- 执行布局：布局阶段开始，Compose 会从被标记的最高层节点开始，重新测量和定位受影响的 LayoutNode。
- 执行绘制：布局完成后，任何在屏幕上位置或外观发生变化的 LayoutNode 会被重新绘制到屏幕上。  
## 绘制

绘制阶段的核心思想是 录制 和 回放，将绘制指令录制到 RenderNode displayList 中，在 RenderThread 中完成回放，RenderThread 调用 Skia 转译指令绘制到 Vulkan

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vJr4RRLJicricRQRYg4OmPE7BHzbMTWNAWzQTfqAkr1Xib0OXSd5T0l4VQ/640?from=appmsg&randomid=73sibx84&tp=webp&wxfrom=5&wx_lazy=1)

  

A RecordingCanvas

LayoutNode 实际绘制过程中通过DrawScope 中调用 drawRect, drawLine 等方法时，并不是在操作一个传统的、立即生效的 Canvas。操作的是一个 RecordingCanvas（在底层是 RenderNode.beginRecording() 返回的 Canvas）。

- 指令记录：这个 Canvas 对象所做的，仅仅是将的调用（drawRect）及其参数（颜色、尺寸、位置）序列化成一个绘图指令，并追加到 RenderNode 内部的 显示列表（Display List） 中。这个过程几乎没有 CPU 开销。
    
- 绘制阶段的产出：所以，整个绘制阶段的最终产出，不是屏幕上的像素，而是一棵 完成了绘图指令记录的   RenderNode   树 。
    

  

B AndroidComposeView 

  

AndroidComposeView 是 Compose 世界与传统 Android View 世界的边界。它继承自 ViewGroup。它的核心职责之一是在合适的时机，将 Compose 产出的 RenderNode 树“嫁接”到 View 系统的 RenderNode 树上，并触发一次绘制。

- attach  和   detach ：当 AndroidComposeView 被附加到窗口时 (onAttachedToWindow)，它会创建 Recomposer 并启动整个 Compose Runtime。当它分离时，会关闭 Recomposer 并释放资源。
    
- dispatchDraw：当 Android 的 View 绘制体系要求 AndroidComposeView 进行绘制时（调用其 dispatchDraw 方法），它并不执行传统的 onDraw。取而代之的是，它会调用一个内部方法，该方法将 Compose 的根 RenderNode 同步到 View 的渲染线程，即上一过程中录制好的 RenderNode 中更新好的 DisplayList。
    

  

C 绘制缓存

LayoutNode 非常智能，它会跟踪自身的绘制内容是否发生变化。如果一个 LayoutNode 只是移动了位置，而其内部内容（如背景色）没有变，它的 RenderNode 缓存就是有效的，不需要重新记录（beginRecording）。渲染线程只需要用新的变换矩阵（translationX/Y）重用这个 RenderNode 即可。

  

此外，Compose 会尽可能地 合并   RenderNode 。如果连续的几个 LayoutNode 都没有使用 graphicsLayer，它们的绘制指令会被记录到同一个父 RenderNode 中。这减少了 RenderNode 树的深度，降低了渲染引擎的管理开销，从而提升了性能。

  

通过对这些底层机制的深入理解，可以看到 Compose 并非简单的“状态变了就重画”。它是一个经过极致优化的、从编译器到渲染引擎端到端设计的精密系统，其核心哲学在于：通过在编译期和运行时收集尽可能多的信息，将更新的粒度降到最低。

  

  

源码位置：

![图片](https://mmbiz.qpic.cn/sz_mmbiz_png/RibZ8KwAStDNVbP5mkH9TKLec54rCM35vGKnpVGZhRX3VUoEjicFM7gR06Cj6vMaxrHC2WVVwe7JqdYGwVGJhcyQ/640?from=appmsg&randomid=6xweyttj&tp=webp&wxfrom=5&wx_lazy=1)

# Gap Buffer算法
1. 一种文本编辑算法；
2. 在光标移动处，预留buffer用于编辑，当光标移动到新的位置时，只需要更新新光标与旧间隙之间的文本位置，移动光标的时间复杂度是 **O(|新位置 - 旧位置|)**，是Gap 主要开销
3. 优点：局部编辑快（在光标附近连续的插入）；文本通过数组存储在内存中连续，访问更快；
4. 缺点：预留内存空间用于buffer 开销 ；光标移动距离过大时，复杂度为O(n)

# Jetpack Compose
## 基本概念
### 可组合函数与附带效应
#### 可组合函数
1. `可组合函数`是Compose中描述UI的函数
2. 函数带有 `@Composable` 注释。所有可组合函数都必须带有此注释；此注释可告知 Compose 编译器：此函数旨在将数据转换为界面。
3. 此函数接受数据。可组合函数可以接受一些参数，这些参数可让应用逻辑描述界面；
4. 此函数可以在界面中显示文本。为此，它会调用 `Text()` 可组合函数，该函数实际上会创建文本界面元素。可组合函数通过调用其他可组合函数来发出界面层次结构。
5. -此函数不会返回任何内容。发出界面的 Compose 函数不需要返回任何内容，因为它们描述所需的屏幕状态，而不是构造界面 widget。
6. - **可组合函数快速、幂等且没有附带效应**。
    - 使用同一参数多次调用此函数时，它的行为方式相同，并且它不使用其他值，如全局变量或对 `random()` 的调用。
    - 此函数描述界面而没有任何副作用，如修改属性或全局变量。
#### 附带效应
`附带效应`是指发生在**可组合函数作用域**之外的应用状态的变化。由于可组合项的生命周期和属性（例如不可预测的重组、以不同顺序执行可组合项的重组或可以舍弃的重组），可组合项在理想情况下应该是无附带效应的。
``` kotlin
@Composable
fun MyScreen(
    title:String
){

    Log.d("UI日志","MyScreen")
    Column{
        Text(title)
    }

}

```
### 重组与智能重组
在命令式界面模型中，如需更改某个 widget，您可以在该 widget 上调用 setter 以更改其内部状态。在 Compose 中，您可以使用新数据再次调用可组合函数。

这样做会导致函数进行重组，系统会根据需要使用新数据重新绘制函数发出的 widget。Compose 框架可以智能地仅重组已更改的组件。
例如，假设有以下可组合函数，它用于显示一个按钮：
``` kotlin
@Composable
fun ClickCounter(clicks: Int, onClick: () -> Unit) {
    Button(onClick = onClick) {
        Text("I've been clicked $clicks times")
    }
}

```
每次点击该按钮时，调用方都会更新 `clicks` 的值。Compose 会再次调用 lambda 与 `Text` 函数以显示新值；此过程称为“重组”。不依赖于该值的其他函数不会进行重组。

如前文所述，重组整个界面树在计算上成本高昂，因为会消耗计算能力并缩短电池续航时间。Compose 使用`智能重组`来解决此问题。

所谓的`智能重组`就是：Compose根据可组合函数的参数来决定是否进行重组。

也就是说，每一次可组合函数被调用的时候，他会检查所有传入的参数，如果本次传入的参数和上一次传入的参数都是相同的话（这里指的相同是指结构性相等，在kotlin中指的是==，在java中指的是调用对象的equals()方法) ，那么Compose就会略过调用这个可组合函数，以达到最快的重组效率。

让我们回到这个可组合函数，如果他的父级可组合函数由于某种原因触发了重组，那么Compose就会尝试调用MyScreen()来完成刷新，如果title参数没有发生变化的话，Compose实际上就会略过MyScreen的刷新。
``` kotlin
@Composable
fun MyScreen(
    title:String
){
    Column{
        Text(title)
    }
}

```
### Compose的生命周期
![[compose的生命周期.png]]
组合中可组合项的生命周期。 进入组合，执行 0 次或多次重组，然后退出组合。
每一次composable（重组）就是调用一次可组合函数
## 基础的Compose UI组件
### Text组件
`Text` 组件用于显示文本内容，可以通过 `fontSize`、`color`、`fontWeight` 等属性进行样式定制。你还可以使用 `maxLines` 来控制最大行数，`overflow` 来定义文本溢出的行为。

```kotlin
Text( text = "Hello, Jetpack Compose!", fontSize = 20.sp, color = Color.Blue, fontWeight = FontWeight.Bold, maxLines = 1, overflow = TextOverflow.Ellipsis )

```
- `fontSize`：设置字体大小。
- `color`：设置文本颜色。
- `fontWeight`：设置文本粗细。
- `maxLines`：限制文本最大显示行数。
- `overflow`：控制文本超出部分的显示方式，如 `TextOverflow.Ellipsis` 表示溢出部分显示省略号。
### Button组件
`Button` 组件用于触发用户交互事件。你可以通过 `onClick` 设置按钮的点击事件，并通过 `Modifier` 调整按钮的样式（如背景色、形状等）。
```kotlin
Button( onClick = { /* 执行操作 */ }, shape = RoundedCornerShape(8.dp), colors = ButtonDefaults.buttonColors(backgroundColor = Color.Blue) ) { Text("Click Me", color = Color.White) }
```

- `onClick`：按钮点击时触发的事件。
- `shape`：设置按钮的形状，这里使用了圆角形状。
- `colors`：设置按钮的背景色和其他颜色属性。
- `Text`：按钮内的文本。
### Image组件
`Image` 组件用于显示图片，可以通过 `painterResource` 加载资源图片，或者使用 `rememberImagePainter` 来加载网络图片。`contentDescription` 便于辅助功能的使用。
```kotlin
Image( painter = painterResource(id = R.drawable.ic_launcher_foreground), contentDescription = "App Icon", modifier = Modifier.size(80.dp).clip(CircleShape) )
```
- `painter`：加载图片的来源。
- `contentDescription`：辅助功能描述，帮助屏幕阅读器识别图片。
- `modifier`：用于控制图片的大小、形状等样式。
### Column & Row 组件（布局容器）
```kotlin
Column(
    verticalArrangement = Arrangement.spacedBy(10.dp),
    horizontalAlignment = Alignment.CenterHorizontally
) {
    Text("Item 1")
    Text("Item 2")
    Text("Item 3")
}

Row(
    horizontalArrangement = Arrangement.spacedBy(15.dp),
    verticalAlignment = Alignment.CenterVertically
) {
    Text("Left")
    Text("Right")
}
```
- `verticalArrangement`：设置竖直方向的排列方式。
- `horizontalArrangement`：设置水平方向的排列方式。
- `verticalAlignment` 和 `horizontalAlignment`：控制子组件的对齐方式。
### Box组件
`Box` 组件用于重叠视图，通常用于显示叠加元素。通过 `Modifier.align()` 可以精确控制子组件的位置。
``` kotlin
Box(
    modifier = Modifier.size(100.dp).background(Color.Gray)
) {
    Text(
        text = "Overlay Text",
        modifier = Modifier.align(Alignment.Center),
        color = Color.White
    )
}


```
- `modifier`：设置 `Box` 组件的样式，例如大小、背景色。
- `align`：设置子组件的对齐方式，在此示例中为居中对齐。

### Modifier 使用指南
`Modifier` 是 Compose 中非常核心的概念，它允许你对 UI 组件进行样式、布局、行为等方面的修改。通过 `Modifier`，你可以调整组件的大小、间距、背景、对齐、状态等特性。
#### 基本用法
`Modifier` 最常见的用法是设置组件的外观和布局属性。常用的属性包括 `padding`、`background`、`fillMaxSize`、`size` 等。
``` kotlin
Text( "Hello, Compose!", modifier = Modifier .padding(16.dp) .background(Color.LightGray) .fillMaxWidth() )
```
- **`padding`**：为组件设置内边距。你可以指定四个方向的内边距，或者统一设置。
- **`background`**：设置组件的背景色，可以使用 `Color` 或者 `Brush` 来定义渐变色背景。
- **`fillMaxWidth`**：让组件的宽度充满父容器。类似的属性还有 `fillMaxHeight`、`fillMaxSize` 等。
#### 布局相关
`Modifier` 可以与布局容器（如 `Column`、`Row`、`Box` 等）结合使用，以控制组件的对齐和排列方式。

- **`fillMaxWidth` 和 `fillMaxHeight`**：使组件填充父容器的宽度或高度。
- **`wrapContentSize`**：使组件的大小包裹其内容。
- **`align`**：通过 `Modifier.align()` 可以控制组件在父容器中的对齐方式

```kotlin
Box( modifier = Modifier .size(100.dp) .background(Color.Gray) ) { Text( text = "Center", modifier = Modifier.align(Alignment.Center), color = Color.White ) }
```
- **`align`**：用于设置子组件在 `Box` 内的对齐方式，这里是居中对齐。
#### 状态与交互
`Modifier` 还可以用来处理交互事件，例如响应点击、拖动、滑动等。

- **`clickable`**：使组件变得可点击，并触发点击事件。
- **`indication`**：用于显示点击、焦点等状态的视觉效果。
```kotlin
Box( modifier = Modifier .size(100.dp) .background(Color.Red) .clickable { /* 点击事件 */ } )
```
- **`clickable`**：将组件变为可点击，点击时触发相应事件。你可以传递 `onClick` Lambda 作为点击事件的响应。
- **`indication`**：与点击效果结合，提供可视化反馈，例如点击时的阴影效果。
#### 变换与动画
`Modifier` 还支持常见的变换操作，如旋转、缩放、平移等，常用于动画和动态效果。

- **`graphicsLayer`**：控制旋转、缩放、透明度等变换。
- **`rotate`**：旋转组件。
- **`scale`**：缩放组件。
- **`offset`**：偏移组件的位置。
``` kotlin
Box( modifier = Modifier .size(100.dp) .background(Color.Blue) .graphicsLayer(rotationZ = 45f) // 旋转组件 )
```

- **`graphicsLayer`**：用于对组件应用变换，如旋转、缩放、透明度等。
- **`rotationZ`**：旋转角度。
- **`scaleX`、`scaleY`**：设置水平和垂直缩放比例。
#### 圆形、圆角和裁剪
`Modifier` 允许你轻松将组件裁剪成圆形或圆角矩形，这对于实现现代 UI 效果非常有用。

- **`clip`**：裁剪组件的形状，例如圆形、圆角矩形等。
- **`RoundedCornerShape`**：用于定义圆角的大小。
- **`CircleShape`**：将组件裁剪为圆形。
``` kotlin
Box( modifier = Modifier .size(100.dp) .background(Color.Green) .clip(CircleShape) // 裁剪为圆形 .border(2.dp, Color.Black) // 添加边框 )

Box( modifier = Modifier .size(100.dp) .background(Color.Red) .clip(RoundedCornerShape(16.dp)) // 圆角裁剪 .border(2.dp, Color.Black) // 添加边框 )


```
- **`clip(CircleShape)`** ：将组件裁剪为圆形。
- **`clip(RoundedCornerShape(16.dp))`** ：将组件裁剪为圆角矩形，圆角半径为 16.dp。
- **`border`**：为组件添加边框，可以与 `clip` 配合使用，确保圆角或圆形边框效果。
#### 阴影效果
`Modifier` 也可以为组件添加阴影效果，这通常与 `clip` 和 `background` 配合使用，以增加层次感和视觉效果。

- **`shadow`**：为组件添加阴影，可以设置阴影的半径、颜色等。
```kotlin
Box( modifier = Modifier .size(100.dp) .background(Color.Yellow) .clip(RoundedCornerShape(16.dp)) // 圆角裁剪 
.shadow(8.dp, RoundedCornerShape(16.dp)) // 添加阴影 )

```
- **`shadow`**：设置阴影的半径，通常与 `clip` 配合使用来确保阴影与组件的形状一致。
### 使用LazyColumn构建高效列表
## 状态管理
在 Jetpack Compose 中，状态（State）是驱动 UI 更新的核心概念。理解 Compose 中的状态管理机制，有助于构建响应式界面，并提升应用的稳定性与可维护性。
### 什么是状态
在 Android 开发中，**状态**通常指的是界面中随时间变化、影响 UI 展示的数据。例如：

- 表单输入框的文本
- 按钮的点击次数
- 加载数据的结果

传统 View 系统通过 `findViewById` 获取控件，再手动更新视图。而在 Compose 中，UI 是由数据驱动的，数据变化会触发 UI 重新绘制（即 **重组**）。因此，管理和保存这些变化的数据成为 Compose 状态管理的核心。
### 为什么需要 `mutableStateOf` 和 `remember`？

#### 引入 `mutableStateOf`

在 Compose 中，`mutableStateOf` 是用来创建和管理可变状态的工具。它创建的状态对象可以在 UI 中观察，状态变化时会自动触发 UI 更新。例如，下面的代码使用 `mutableStateOf` 来存储按钮的点击次数
``` kotlin
@Composable fun Counter() { // 使用 mutableStateOf 创建可变的状态 var count = mutableStateOf(0) Column { Text(text = "点击次数: ${count.value}") Button(onClick = { count.value++ }) { Text("点击我") } } }

```
在这个例子中，`mutableStateOf(0)` 创建了一个可观察的状态对象，`count` 变量持有这个状态的值。每当按钮点击时，`count.value++` 会更新这个值，并触发 UI 更新。

然而，在这个代码中存在一个问题：每次 UI 更新（即重组）都会重新执行 `Counter()` 函数，这意味着 `count` 每次都会被重置为 `0`。这就导致每次点击按钮时，`count` 始终不变。

### 引入 `remember`
为了避免每次重组时状态丢失，Compose 提供了 `remember` 函数。`remember` 会在同一次重组中保存状态，使得状态数据能够在重组过程中保持不变。我们可以结合 `remember` 和 `mutableStateOf` 来解决这个问题：
``` kotlin
@Composable 
fun Counter() { // 使用 remember 来保留状态
var count by remember { mutableStateOf(0) }
Column { 
	Text(text = "点击次数: $count") 
	Button(onClick = { count++ }) 
		{ Text("点击我") }
	}
 }
```
在这个代码中，`remember { mutableStateOf(0) }` 确保 `count` 在同一次重组过程中保持状态。当点击按钮时，`count` 会正确增加，而 UI 也会随着 `count` 的变化自动更新。

#### `remember` 和 `mutableStateOf` 的底层原理

- `mutableStateOf` 是一个 `State<T>` 对象，内部使用了**观察者模式**，当状态变化时，Compose 会通知相关的 Composable 重新执行并更新 UI。
- `remember` 本质是一个缓存机制，能够在当前组合范围（Composition）内保持数据，防止 UI 重组时丢失状态。

### Compose 重组机制（Recomposition）
#### 重组是如何工作的？
在 Compose 中，**重组**（Recomposition）是指当状态发生变化时，Compose 会重新执行受影响的 Composable 函数，并重新绘制 UI。重组是 Compose 的核心特性，它使得 UI 动态响应数据的变化。

当我们修改一个 `State` 对象的值时（例如，通过 `mutableStateOf`），Compose 会检测到这个变化，并标记需要更新的 Composable。随着 Composable 被重新执行，UI 会根据新的数据重新呈现。
##### 重组与UI更新的关系
在传统的 Android 开发中，UI 更新是手动触发的，比如调用 `invalidate()` 或 `setText()` 方法。而在 Compose 中，UI 更新由**数据驱动**，当状态发生变化时，UI 会自动更新。
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Log.d("Compose", "Counter 重组")

    Column {
        Text("点击次数: $count")
        Button(onClick = { count++ }) {
            Text("点击我")
        }
    }
}

```

在这个例子中，每次按钮被点击时，`count` 会更新，Compose 会触发重组。通过 `Log` 输出，我们可以看到每次点击按钮时，`Counter` Composable 会重新执行，并在日志中输出 "Counter 重组"。

#### 重组的精细化控制
Compose 的一个关键优势是**高效的重组机制**，即使状态变化，也不会导致整个 UI 被重新绘制。Compose 会根据需要更新**最小范围**的 UI。

- **局部更新**：Compose 会仅重组受状态变化影响的部分 Composables。例如，如果按钮的点击次数变化，只会更新显示次数的 `Text` 组件，而不会重新创建整个 `Counter` 组件。
- **避免不必要的重组**：Compose 通过智能比较来确定哪些 Composables 需要更新，避免了重复的计算和 UI 渲染，优化了性能。
#### 重组的执行过程
1.  **触发重组**：当 `mutableStateOf` 的值发生变化时，Compose 会标记这个 Composable 需要重新执行。
2. **计算新的 UI**：Compose 会重新执行该 Composable，计算新的 UI 树（UI 结构）。
3. **更新 UI**：Compose 会将新的 UI 树与当前的 UI 树进行对比，只更新发生变化的部分，从而高效地呈现更新后的界面。
#### 为什么要关注重组？

理解 Compose 的重组机制对开发者非常重要，因为它能够帮助你：

- **避免性能问题**：确保不必要的 UI 更新不会发生，优化性能。
- **提高响应性**：确保 UI 始终与状态保持同步，用户体验流畅。
### `remember` vs `rememberSaveable`
- `remember` 只能在 **内存** 中保存状态，适用于**短生命周期**的数据。
- `rememberSaveable` 支持**持久化**，即使在 **进程被杀死或配置更改**（如旋转屏幕）时，也能恢复状态。
#### `rememberSaveable` 与 `remember` 的对比

`remember` 和 `rememberSaveable` 都用于在 `Compose` 中保存和恢复状态，但它们的区别在于如何处理配置变化（如屏幕旋转）和进程销毁。
##### `remember`
`remember` 用于保存状态，只在组件重组时保留状态。配置变化（如屏幕旋转）或进程销毁时，状态会丢失。
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Column {
        Text("点击次数: $count")
        Button(onClick = { count++ }) {
            Text("点击我")
        }
    }
}

```
#####  `rememberSaveable`
`rememberSaveable` 类似 `remember`，但它会将状态保存在 `Bundle` 中，在配置变化时恢复状态。适用于需要保持状态的场景，如表单输入。
```kotlin
@Composable
fun Counter() {
    var count by rememberSaveable { mutableStateOf(0) }

    Column {
        Text("点击次数: $count")
        Button(onClick = { count++ }) {
            Text("点击我")
        }
    }
}

```
- **区别**：`rememberSaveable` 可以在配置变化时恢复状态，而 `remember` 只在组件重组时保存状态。
###### `rememberSaveable` 的原理

`rememberSaveable` 使用 `Bundle` 来保存状态，使得状态能在配置变化时恢复。当屏幕旋转或进程销毁后，状态会自动恢复。

### 状态提升（State Hoisting）
状态提升是将状态从子组件提取到父组件，使 UI 与状态管理解耦。这种做法提升了组件的复用性、可测试性，并且允许多个组件共享相同的状态。
#### 状态提升的实际应用
为了实现计数器功能且保证状态在重组时不丢失，我们将状态提升到父组件中进行管理。如下所示
``` kotlin
@Composable
fun ParentComponent() {
    var count by remember { mutableStateOf(0) } // 状态提升到父组件

    Counter(count, onIncrement = { count++ })
}

@Composable
fun Counter(count: Int, onIncrement: () -> Unit) {
    Column {
        Text("点击次数: $count")
        Button(onClick = onIncrement) {
            Text("点击我")
        }
    }
}

```

在这个例子中：

- `ParentComponent` 组件管理 `count` 状态，并通过 `count` 和 `onIncrement` 回调传递给 `Counter` 组件。
- `Counter` 组件仅负责展示文本框和响应用户输入，实际的状态由父组件控制。

这种方式可以确保 `Counter` 组件的复用性：无论多少个 `Counter` 组件，它们都可以通过父组件共享和管理同一个计数器状态。

优势：

- **复用性**：`Counter` 组件变得独立且无状态，能在多个地方复用。
- **解耦性**：UI 展示和状态管理分离，提升了可维护性和测试性。
#### 什么时候不需要状态提升？
并不是所有情况下都需要进行状态提升。在一些简单的、状态完全局部的组件中，直接在组件内部管理状态更加简洁。例如，如果我们有一个组件用于显示计时器，它的状态只在组件内部有效，不需要与外部共享，那么就没有必要提升状态：
``` kotlin
@Composable
fun Timer() {
    var time by remember { mutableStateOf(0) }
    
    LaunchedEffect(true) {
        while (true) {
            delay(1000)
            time++
        }
    }

    Text("计时器: $time")
}

```
在这个例子中，`Timer` 组件内部管理 `time` 状态，它不需要和父组件交互，因此不需要进行状态提升。状态直接管理在 `Timer` 内部就足够了
### Compose 与 ViewModel 状态结合
通常我们通常会使用 `ViewModel` 来持有和管理状态，确保数据在组件生命周期内得以保存。结合 `Compose` 和 `ViewModel`，可以实现更加灵活和稳定的状态管理。
#### **ViewModel + StateFlow / LiveData**

`ViewModel` 用于管理和存储 UI 相关的数据，而 `StateFlow` 和 `LiveData` 是在 `Compose` 中常用的两种可观察的数据类型。通过 `collectAsState`（对于 `Flow`）或 `observeAsState`（对于 `LiveData`），`Compose` 会自动观察数据的变更并更新 UI。

##### 示例：使用 `StateFlow`
``` kotlin
class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count

    fun increment() {
        _count.value++
    }
}

@Composable
fun CounterScreen(viewModel: CounterViewModel = viewModel()) {
    // collectAsState 会自动观察 StateFlow 数据，并更新 UI
    val count by viewModel.count.collectAsState()

    Column {
        Text("点击次数: $count")
        Button(onClick = { viewModel.increment() }) {
            Text("点击我")
        }
    }
}

```
在这个例子中，`StateFlow` 被用来管理计数器的状态。`collectAsState` 会自动监听 `StateFlow` 的变化并更新 UI。
##### 示例：使用 `LiveData`
``` kotlin
class CounterViewModel : ViewModel() {
    private val _count = MutableLiveData(0)
    val count: LiveData<Int> = _count

    fun increment() {
        _count.value = (_count.value ?: 0) + 1
    }
}

@Composable
fun CounterScreen(viewModel: CounterViewModel = viewModel()) {
    // observeAsState 会自动观察 LiveData 数据，并更新 UI
    val count by viewModel.count.observeAsState(0)

    Column {
        Text("点击次数: $count")
        Button(onClick = { viewModel.increment() }) {
            Text("点击我")
        }
    }
}

```
在这个例子中，`LiveData` 用于管理计数器的状态。`observeAsState` 会自动监听 `LiveData` 的变化，并在数据变更时更新 UI。

- `collectAsState`（适用于 `StateFlow`）和 `observeAsState`（适用于 `LiveData`）能够自动监听数据的变化，并将变化及时反映到 UI 上。
- `StateFlow` 和 `LiveData` 都是响应式的，当数据变化时，它们会自动通知 `Compose` 来触发 UI 更新。
### 总结
- **状态** 是 Compose 的核心，驱动 UI 更新。
- 使用 `mutableStateOf` 创建可变状态，结合 `remember` 来保留状态，避免重组时数据丢失。
- `rememberSaveable` 适用于需要持久化的状态，如配置更改时需要保留的数据。
- 采用**状态提升**模式，解耦 UI 与数据，提升组件复用性和可测试性。
- 与 **ViewModel** 配合使用，可以在复杂应用中保持数据的长期存活和稳定性。

通过理解 Compose 状态管理机制，可以更高效、优雅地实现响应式 UI，提升应用性能与用户体验。
## 附带效应、协程与异步
## 动画
## 高级布局与绘制技巧
###  使用 Canvas 自定义绘制

`Canvas` 是 Compose 中提供的低层次绘图 API，类似于传统的 `onDraw()` 方法。通过 `drawRect()`、`drawCircle()`、`drawPath()` 等方法，你可以绘制各种图形，满足自定义需求。
#### 示例：绘制一个自定义圆形进度条

```kotlin
@Composable
fun CoolProgressBar(
    progress: Float, // 进度 0f - 1f
    modifier: Modifier = Modifier,
    strokeWidth: Float = 20f, // 进度条宽度
    startColor: Color = Color(0xFFFF5722), // 渐变起始色
    endColor: Color = Color(0xFF2196F3) // 渐变结束色
) {
    Canvas(modifier = modifier) {
        val size = size.minDimension
        val radius = size / 2f
        val center = Offset(size / 2f, size / 2f)

        // 绘制背景圆环
        drawCircle(
            color = Color.LightGray.copy(alpha = 0.3f),
            radius = radius - strokeWidth / 2,
            style = Stroke(width = strokeWidth, cap = StrokeCap.Round)
        )

        // 计算渐变颜色
        val brush = Brush.linearGradient(
            colors = listOf(startColor, endColor),
            start = Offset(0f, 0f),
            end = Offset(size, size)
        )

        // 计算进度角度
        val sweepAngle = progress * 360f

        // 绘制进度条
        drawArc(
            brush = brush,
            startAngle = -90f,
            sweepAngle = sweepAngle,
            useCenter = false,
            style = Stroke(width = strokeWidth, cap = StrokeCap.Round)
        )
    }
}

@Composable
fun CoolProgressBarDemo() {
    var progress by remember { mutableFloatStateOf(0.3f) }

    // 进度平滑动画
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(durationMillis = 1000, easing = FastOutSlowInEasing), label = ""
    )

    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        CoolProgressBar(
            progress = animatedProgress,
            modifier = Modifier
                .size(150.dp)
                .clip(CircleShape)
        )

        Spacer(modifier = Modifier.height(20.dp))

        Button(onClick = { progress = Random.nextFloat().coerceIn(0.1f, 1f)}) {
            Text("随机进度")
      

```
通过 `Canvas`，我们可以绘制一个带有背景圆环和动态进度的圆形进度条。`drawArc()` 让进度条根据传入的 `progress` 绘制。
### 使用 Layout 自定义布局

Compose 中，`Row`、`Column` 和 `Box` 可能无法满足所有布局需求。此时，可以使用 `Layout` API 创建复杂的自定义布局。`Layout` 像是传统View中`onMeasure()`+`onLayout()`的结合。

#### 示例：实现一个流式布局

```koltin
@Composable
fun FlowLayout(
    modifier: Modifier = Modifier,
    maxWidth: Dp = 300.dp, // 限制整体宽度
    content: @Composable () -> Unit
) {
    Layout(
        content = content,
        modifier = modifier
    ) { measurables, constraints ->

        // 将 maxWidth 转换为 Px，并与父布局宽度取最小值，确保不超出
        val layoutMaxWidth = minOf(maxWidth.roundToPx(), constraints.maxWidth)

        // 测量所有子项
        val placeables = measurables.map { it.measure(constraints.copy(maxWidth = layoutMaxWidth)) }

        var currentX = 0
        var currentY = 0
        var maxHeightInRow = 0

        layout(layoutMaxWidth, constraints.maxHeight) {
            placeables.forEach { placeable ->
                // 超出 maxWidth 时换行
                if (currentX + placeable.width > layoutMaxWidth) {
                    currentX = 0
                    currentY += maxHeightInRow
                    maxHeightInRow = 0
                }

                placeable.placeRelative(currentX, currentY)
                currentX += placeable.width
                maxHeightInRow = maxOf(maxHeightInRow, placeable.height)
            }
        }
    }
}

@Composable
fun TestFlowLayout() {
    FlowLayout(maxWidth = 320.dp) {
        FlowText("MyOwnColumn")
        FlowText("places items")
        FlowText("vertically.")
        FlowText("We've done it by hand!")
        FlowText("final")
    }
}

@Composable
fun FlowText(text: String) {
    Text(
        text = text,
        modifier = Modifier
            .padding(8.dp)
            .border(1.dp, Color.Gray)
            .padding(8.dp)
    )
}

```
通过 `Layout` API，可以控制每个元素的尺寸和位置。这里的布局会将每个标签垂直排列，形成一个简单的标签列表。
### 使用 Modifier.drawWithContent 定制绘制
如果你需要在现有组件上添加额外的绘制效果（如边框或渐变效果），可以使用 `Modifier.drawWithContent`。
#### 示例：为文本添加下划线

```kotlin
@Composable
fun UnderlinedText(text: String) {
    Text(
        text = text,
        modifier = Modifier.drawWithContent {
            drawContent() // 先绘制内容
            drawLine(
                color = Color.Red,
                start = Offset(0f, size.height),
                end = Offset(size.width, size.height),
                strokeWidth = 2f
            )
        }
    )
}

@Preview
@Composable
fun UnderlinedTextPreview() {
    UnderlinedText("带下划线的文本")
}

```
### 使用 AndroidView 复用传统 View
通过 `AndroidView`，你可以将传统的 `View` 嵌入 Compose 布局中，复用 XML 布局中已有的自定义视图。
#### 示例：在 Compose 中嵌入 传统View
```kotlin
@Composable
fun WebViewComponent(url: String) {
    AndroidView(
        modifier = Modifier.fillMaxSize(),
        factory = { context ->
            //WebView、SurfaceView、其它自定义View
        }
    )
}

```
### 五、总结

|自定义方式|适用场景|核心 API|
|---|---|---|
|**Canvas**|自定义图形、控件|`drawCircle()`、`drawPath()`|
|**Layout**|自定义布局|`Layout`、`MeasureScope`|
|**drawWithContent**|叠加效果|`drawLine()`、`drawRect()`|
|**AndroidView**|复用传统 View|`AndroidView`|

Compose 提供了灵活的自定义扩展能力，可以满足大多数 UI 设计需求。根据实际情况选择合适的自定义方式，能够帮助你轻松实现多种复杂的 UI 效果。
# 参考链接
1. [妈！Jetpack Compose太难学了，别怕，这里帮你理清几个概念](https://juejin.cn/post/7244420350753144891?searchId=202507052141214BE0E8888B9B55CE12B4#heading-8
2. [鸿蒙 KMP Compose 源码万字简析](https://mp.weixin.qq.com/s?__biz=MzUyMjI4MzE1MA==&mid=2247484967&idx=1&sn=6496e0421d57ea0262ca10f5f7950c43&chksm=f9cf77c6ceb8fed0bd9295a62ed5cf9166b5df2a07e4a83828c4b020e4699cbacabc92604f13&cur_album_id=4008501025152761864&scene=189#wechat_redirect)
3. [鸿蒙 KMP Compose 开发范式理解](https://mp.weixin.qq.com/s?__biz=MzUyMjI4MzE1MA==&mid=2247484957&idx=1&sn=e1f415b0cce0eb0eafcf031aaf7790ed&chksm=f9cf77fcceb8feea493de60229800eee81c01cf7486b99d7abaa4da29fdc2876edd0834a4e31&cur_album_id=4008501025152761864&scene=189#wechat_redirect)
4. [Jetpack Compose 【一】入门：拥抱现代 Android UI 开发](https://juejin.cn/post/7473316836362141696?searchId=202507122209440BB77DF9C87C1B5340FB)
5. [Jetpack Compose 【二】状态管理详解](https://juejin.cn/post/7473278704052994082#heading-12)
6. [Android 架构之 MVI 雏形 | 响应式编程 + 单向数据流 + 唯一可信数据源](https://juejin.cn/post/7087717477246369805)
7. [Jetpack Compose UI创建布局绘制流程+原理 —— 内含概念详解（满满干货）](https://juejin.cn/post/7003606223438217223)
8. [深入浅出 JetPack Compose UI 自动更新原理](https://blog.csdn.net/weixin_61845324/article/details/134268501)
9. [Compose编程思想 -- 深入理解Compose原理](https://juejin.cn/post/7355307547479572521#heading-9)
10. [开源一个企业可用的 Kotlin Multiplatform 项目模板](https://juejin.cn/post/7302338286768635956?searchId=202507292001582A7C0BD3C1624A8E74F1)