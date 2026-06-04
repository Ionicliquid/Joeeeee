# 声明式UI
1. 描述 UI 与状态的关系，当状态发生变化时，UI自动更新；
2. 如何实现 Button点击，TextView 计数的功能
	1. 在命令式 UI中，需要监听 Button的点击，点击时，修改 TextView 的显示
	2. 而在声明式 UI 中，Button 点击修改状态，TextView 描述与状态的关系，自动响应状态变化。
# SlotTable
1. 首先从 @Copmposable 函数说起，可组合函数经过编译后会额外添加 2 个参数，类型为 Composer 与 Int。
2. Composer 的作用就是就是将 UI 结构，状态信息记录到 SlotTable 中，Int 值用于标记是否需要参与重组；
3. 函数内部插入了很多 `$composer.startXXXGroup` 和 `$composer.endXXXGroup` 模板代码;
4. SlotTable 有两个数组，`groups` 数组存储 Group 信息，`slots` 存储 Group 数据，最常见的 Group 有 4 种；
	1. startNode/startReusableNode插入一个包含 Node 的 Group。~~例如文章开头 ReusableComposeNode 的例子中，显示调用了 startReusableNode ，而后调用 createNode 在 Slot 中插入 LayoutNode~~。
	2. startRestartGroup	插入一个可重复执行的 Group，它可能会随着重组被再次执行，因此 RestartGroup 是重组的最小单元。
	3. startReplaceableGroup	插入一个可以被替换的 Group，例如一个 if/else 代码块就是一个 ReplaceableGroup，它可以在重组中被插入后者从 SlotTable 中移除。
	4. startMovableGroup	插入一个可以移动的 Group，在重组中可能在兄弟 Group 之间发生位置移动。
	5. ~~startReusableGroup，插入一个可复用的 Group，其内部数据可在 LayoutNode 之间复用，例如 LazyList 中同类型的 Item。~~
5. startRestartGroup插入在可组合函数头，每次重组都执行，是重组的最小单元，一个组合函数内的其他 Group 都是它的子 Group;
6. startReplaceableGroup可被移除的 Group，插入在 if/else的不同分支，条件变化时，直接移除旧的 Group， 插入更新后的 Group;
7. startMovableGroup，插入一个可以移动的 Group，用于显示列表，用于标记每个 item 的位置，一般会指定唯一 key，用于优化重组性能;
8. startNode/startReusableNode ,插入一个包含 Node 的 Group,Node 就是 LayoutNode,也就是 UI结点；
	- 多个LayoutNode 组合生成 Compose 的渲染树，完成测量，布局和绘制工作；
	- UI 的变化本质是触发重新执行组合函数，更新 LayoutNode 树，重新渲染布局；
9. 重组结束后，调用applyChanges应用修改，SlotTable 中的数据更新使用 Gap Buffer 算法优化
	- 当插入新 Node 时，会将数组中的 Gap 移动到待插入位置，然后再开始插入新 Node，后续插入填充 Gap 的空闲区域，不会移动整个列表；
# 重组
1. 重组的本质：当状态改变时，Recomposer 调度受该状态影响的组合函数重新执行。在这个过程中，涉及到的4个对象：State/Recomposer/Composer/RecomposeScopeImpl；
2.  State状态会关联快照，组合成快照状态（ SnapshotState），快照有 2 个作用
	1. 对状态的读写进行隔离，组合时会创建子快照，子快照内对状态的修改需要主动调用apply提交到全局快照；
	2. 状态读写感知，组合时对状态的读写都会被记录，作为重组的依据；
3. Recomposer ：setContent时创建并初始化，初始化时注册全局快照状态监听，并开启协程监听 VSYNC 信号，发起并驱动重组流程，无变更内容时会挂起等待；
4. Composer：在组合和重组期间将 UI结构、状态等信息记录到 SlotTable 中，同时会创建  `RecomposeScopeImpl`存入 SlotTable中；
5. RecomposeScopeImpl：组合函数的持有者，作为重组的作用域，当组合函数读取的状态变化时，Compose 会从 SlotTable找到它，调用其 invalidate标记失效，重新执行其持有的组合函数；
# 副作用（Side Effects）
副作用是指在 UI 渲染之外执行的操作，如日志记录、网络请求、数据库操作等；
1. **LaunchedEffect**：在Composable中**安全地启动一个协程**，适用于**响应式事件**（如点击一次，加载一次数据）。当其`key`变化或组件首次进入组合时，会重启；
2. **`DisposableEffect`**：用于注册需要在组件**离开组合时进行清理**的资源（如监听器、订阅），类似于`onDestroy`。
3. **`rememberCoroutineScope`**：获取一个与当前Composable生命周期绑定的协程作用域，用于在**非组合作用域**（如回调函数中）启动协程
4. rememberUpdatedState:捕获最新的状态以确保在回调中使用

| **API**                    | **用途**                      | **是否支持协程** | **生命周期绑定** |
| -------------------------- | --------------------------- | ---------- | ---------- |
| **SideEffect**             | 每次重组时执行操作（无清理需求）            |            |            |
| **DisposableEffect**       | 需要清理资源的副作用（监听、注册等）          |            |            |
| **LaunchedEffect**         | 适合异步操作，自动取消协程               |            |            |
| **rememberCoroutineScope** | 启动协程，作用域不受重组影响              |            |            |
| **produceState**           | 从异步数据生成 Compose 状态          |            |            |
| **derivedStateOf**         | 根据其他状态派生计算新状态               |            |            |
| **snapshotFlow**           | 将 Compose 状态转换为 StateFlow   |            |            |
| **rememberUpdatedState**   | 捕获最新的状态以确保在回调中使用            |            |            |
| **observeAsState**         | 将 LiveData 转换为 Compose 状态\| |            |            |
| **collectAsState**         | 将 StateFlow 转换为 Compose 状态  |            |            |
# remember
1. remember 会将对象存储在当前所处的 Group对应的数据槽中，避免重组时对象重新创建；
2. rememberUpdatedState：重组对数据进行重新赋值；
# 动画
##  `AnimateXxxAsState`
`animate*AsState` 系列动画是 Compose 中最常见的动画方式，它允许我们动画化元素的某些属性，如尺寸、颜色和位置等。
``` kotlin
val logoOffsetY by animateDpAsState(  
    targetValue = if (startAnimation) 0.dp else 160.dp,  
    animationSpec = tween(durationMillis = 1000),  
    label = "logoOffsetY"  
)

val logoRotation by animateFloatAsState(  
    targetValue = if (startAnimation) 0f else -15f,  
    animationSpec = tween(durationMillis = 1000),  
    label = "logoRotation"  
)
```
## `AnimatedVisibility`
## `Transition` 动画
## `AnimationSpec`动画
`AnimationSpec` 定义了动画的行为，类似于传统View体系中的差值器`Interpolator`，包括动画的速度、持续时间、缓动曲线等。它适用于所有 `animate*` 系列函数（如 `animateDpAsState`、`updateTransition`、`Animatable` 等），用于控制动画的执行方式。
# 自定义布局
Compose 实现自定义布局有 4 种方式
### Canvas
使用 Canvas自定义绘制，类似于传统 View 的 onDraw 方法，通过 drawRect drawCircle drawPath 等方法绘制各种图形；
### Layout
使用 Layout自定义布局，类似于传统 View 的 onMeasure与 onLayout的结合，核心 API 是measurable与placeable用于测量大小和摆放位置
``` kotlin
@Composable  
fun VerticalGrid(modifier: Modifier = Modifier, columns: Int = 2, content: @Composable () -> Unit) {  
  Layout(content = content, modifier = modifier) { measurables, constraints ->  
    val itemWidth = constraints.maxWidth / columns  
    // Keep given height constraints, but set an exact width  
    val itemConstraints = constraints.copy(minWidth = itemWidth, maxWidth = itemWidth)  
    // Measure each item with these constraints  
    val placeables = measurables.map { it.measure(itemConstraints) }  
    // Track each columns height so we can calculate the overall height  
    val columnHeights = Array(columns) { 0 }  
    placeables.forEachIndexed { index, placeable ->  
      val column = index % columns  
      columnHeights[column] += placeable.height  
    }  
    val height =  
      (columnHeights.maxOrNull() ?: constraints.minHeight).coerceAtMost(constraints.maxHeight)  
    layout(width = constraints.maxWidth, height = height) {  
      // Track the Y co-ord per column we have placed up to  
      val columnY = Array(columns) { 0 }  
      placeables.forEachIndexed { index, placeable ->  
        val column = index % columns  
        placeable.placeRelative(x = column * itemWidth, y = columnY[column])  
        columnY[column] += placeable.height  
      }  
    }  }}
```
1. place(x, y) ： 绝对位置，忽略 RTL
2. placeRelative(x, y)：相对位置，根据 RTL 调整方向；
### Modifier.drawWithContent
Modifier的drawWithContent方法，在已有布局上叠加效果
### Android View
 通过 `AndroidView`，你可以将传统的 `View` 嵌入 Compose 布局中，复用 XML 布局中已有的自定义视图。
# ~~测量~~
1. ~~和原生视图由 View 组成的渲染数类似，Compose也存在一棵由LayoutNode组成的渲染树，通过 LayoutNode 完成 measure/layout/draw 的过程最终将 UI 显示到屏幕上；~~
2. ~~LayoutNode在创建时会指定measurePolicy与 Modifier;~~
# 常见控件
## SubcomposeAsyncImage
## Image
``` kotlin
Image(  
    painter = painterResource(id = pageData.image),  
    contentDescription = "Guide",  
    modifier = Modifier  
        .size(240.dp)  
)
```
## Icon
## IconButton
## Row/Column/Box：X 轴，Y 轴，Z 轴
## HorizontalPager：水平分页器
1. rememberPagerState：创建 分页器状态
2. animateScrollToPage（挂起函数）：滚动到指定页面
``` kotlin
internal fun GuideRoute(  
    viewModel: GuideViewModel = hiltViewModel()  
) {  
    // 协程作用域  
    val scope = rememberCoroutineScope()  
  
    // 从ViewModel获取状态  
    val currentPageIndex by viewModel.currentPageIndex.collectAsState()  
  
    GuideScreen(  
        guidePages = viewModel.guidePages,  
        currentPageIndex = currentPageIndex,  
        isLastPage = viewModel.isLastPage(),  
        onPageChanged = viewModel::updatePageIndex,  
        onNextClick = { pagerState ->  
            if (viewModel.handleNextClick()) {  
                // 需要切换到下一页  
                scope.launch {  
                    val nextPage = viewModel.getNextPageIndex()  
                    pagerState.animateScrollToPage(nextPage)  
                }  
            }  
        },  
        onSkipClick = viewModel::skipGuide  
    )  
}
```
## Compose Navigation
 1. NavController：导航控制器，负责管理应用的导航状态，跟踪当前显示的目的地，处理导航动作
2. NavGraph：导航图，定义所有目的地及其之间的连接关系，是导航的蓝图
3. Destination：目的地，代表应用中的一个屏幕（Composable 函数）
4. Action：导航动作，定义从一个目的地到另一个目的地的跳转
5. NavBackStackEntry存储了导航中回退栈的信息。可以通过以下方式获取
6. popBackStack:回到上一页

``` kotlin
val backstackEntry = navController.currentBackStackEntryAsState() 
val route = backstackEntry.value?.destination?.route //获取当前的路
```
### toPageAndCloseCurrent：关闭当前页面并导航到指定路由
``` kotlin
fun toPageAndCloseCurrent(route: String, currentRoute: String) {  
    viewModelScope.launch {  
        val targetRoute = checkRouteInterception(route)  
        val navOptions = NavOptions.Builder()  
            .setPopUpTo(  
                route = currentRoute,  
                inclusive = true,  // 设为true表示当前页面也会被弹出  
                saveState = false  // 不保存状态  
            )  
            .build()  
        navigator.navigateTo(targetRoute, navOptions)  
    }  
}
```
## Scaffold：脚手架
1. contentWindowInsets：导航栏与状态栏设置
	1. WindowInsets.navigationBars：导航栏
	2. WindowInsets.statusBars：状态栏
2. bottomBar：底部导航栏
3. topBar：顶部导航栏
``` kotlin
Scaffold(  
    // 排除顶部导航栏边距  
    contentWindowInsets = ScaffoldDefaults  
        .contentWindowInsets  
        .exclude(WindowInsets.statusBars),  
    bottomBar = {  
        BottomNavigationBar(  
            destinations = TopLevelDestination.entries,  
            onNavigateToDestination = { index ->  
                // 通知选择了新的导航项  
                onNavigationItemSelected(index)  
                scope.launch {  
                    pageState.scrollToPage(index)  
                }  
            },  
            currentDestination = currentDestination,  
            modifier = Modifier  
        )  
    }  
) { paddingValues ->  
    MainScreenContentView(  
        pageState = pageState,  
        paddingValues = paddingValues,  
        sharedTransitionScope = sharedTransitionScope,  
        animatedContentScope = animatedContentScope  
    )  
}
```

## TopAppBar
## AnimatedContent
内容变化时自动应用进入、退出和大小变换动画
## LazyVerticalStaggeredGrid
1. 创建交错网格布局的组件,每个布局对应一个 item；
## HorizontalPager
水平分页
![[Pasted image 20251122182858.png]]
## SharedTransitionLayout
2 个页面有共同内容时，用于创建共享页面的过渡动画
## 自定义的指示器
``` kotlin
@Composable  
fun Indicator(count: Int, current: Int, modifier: Modifier = Modifier) {  
    Row(  
        modifier  
            .width((count * 16).dp)  
            .height(3.dp)  
            .clip(RoundedCornerShape(2.dp))  
            .background(Primary.copy(0.2f))  
    ) {  
        val offsetX by animateDpAsState(  
            targetValue = (current * 16).dp,  
            label = "OffsetAnimation"  
        )  
        Box(  
            Modifier  
                .width(16.dp)  
                .height(3.dp)  
                .offset(x = offsetX)  
                .clip(RoundedCornerShape(2.dp))  
                .background(Primary.copy(0.6f))  
        )  
    }  
}
```
# Modifier

- `Modifier.clip()` 可以将组件的内容裁剪到指定的形状范围内，超出形状的部分将不可见。
- Modifier.zIndex： 多个组件重叠时，哪个组件显示在最上层。数值越大，组件显示越靠前。Box布局

# KMP
1. KMP 的优势是可以将 kotlin 代码通过 kotlin IR 编译成目标平台原生代码执行，执行时几乎没有性能损耗；
2. 对于系统接口，目前还存在对应平台限制，比如 Java包名和android 包名接口以及一些三方依赖，需要进行大量的抽象改造之后才能正确生成对应平台的产物；
3. 通过 expect/actual 机制声明统一接口与平台实现;
# hilt
## 注入点（类）
1. 除了Application和ViewModel，其它注入点都是通过使用@AndroidEntryPoint。
2. 注入点其实就是依赖注入开始的点，比如Activity里需要注入A依赖，A里又需要注入B依赖，B里又需要注入C依赖，从Activity开始我们就能构建所有的依赖
![[hilt 注入点.jpg]]
## 组件的生命周期
![[组件的生命周期.jpg]]
## 组件的作用域
- Hilt默认不绑定任何作用域，由此带来的结果是每一次注入都是全新的对象
- 组件的作用域要么不指定，要指定那必须和组件的生命周期一致
![[组件的作用域.jpg]]
2. @Singleton 被它修饰的构造函数或是函数，返回的始终是同一个实例  
3. @ActivityRetainedScoped 被它修饰的构造函数或是函数，在Activity的重建前后返回同一实例  
4. @ActivityScoped 被它修饰的构造函数或是函数，在同一个Activity对象里，返回的都是同一实例  
5. @ViewModelScoped 被它修饰的构造函数或是函数，与ViewModel规则一致

## 注入三方类
1. 定义Provides模块，@Provides适用于无法修改类的构造函数的场景，多用于注入第三方的对象
``` kotlin
@Module 
@InstallIn(SingletonComponent::class) 
object HardwareModule { 
	@Provides 
	fun provideHardware():Hardware { 
		return Hardware()
	} 
}
```
1. @Module和@InstallIn 注解是必须的
2. 定义object类
3. 定义函数，方法名随意，返回类型为我们需要注入的类型
4. 函数体里通过构造或是其它方式创建具体实例
5. 使用@Provides注解函数
## 总结
1. 核心是依赖编译时注解生成的辅助类完成属性的赋值；
2. 根据注入点类型生成对应的 Component组件。注入点分为
	1. 注入点有Application和ViewModel以及 Activity，Service Fragment等，Application 对应 hiltAndroidApp，ViewModel 对应 hiltViewModel，其他对应 AndroidEntryPoint;
3. 组件的作用就是完成依赖注入，在 Dagger 中需要手动调用，hilt 通过注解生成对应 Hilt_Activity 和 Application后，编译时通过 字节码插桩 修改 Activity 和 Application 的父类完成注入；
4. 对于被注入的对象则生成对应的Factory类；
# 其他
## Jetpack Compose 与 View 系统混用
1. 在传统 View 系统中使用 Compose
	- `ComposeView`，它是一个继承了 `ViewGroup` 的类，可以装载 Composable 函数。
2. 在 Compose 中使用传统 View
	- 使用 `AndroidView` Composable 函数，它允许我们往 Compose 中嵌入一个传统的 View
## 状态提升
1. 状态提升是将状态从子组件提取到父组件，使 UI 与状态管理解耦。这种做法提升了组件的复用性、可测试性，并且允许多个组件共享相同的状态。

## rememberScrollState
`rememberScrollState` 用于**创建和管理可滚动组件的滚动状态**。
```kotlin
val scrollState = rememberScrollState()

LaunchedEffect(scrollState) {  
    snapshotFlow {  
        scrollState.value  
    }.collectLatest { scrollY ->  
        var alpha = scrollY  
        if (alpha > 255) {  
            alpha = 255  
        }  
        onTopBarAlphaChanged(alpha)  
    }  
}

Column(  
    modifier = Modifier  
        .fillMaxSize()  
        .verticalScroll(scrollState)  
)
```
# Gap Buffer算法
1. 一种文本编辑算法；
2. 在光标移动处，预留buffer用于编辑，当光标移动到新的位置时，只需要更新新光标与旧间隙之间的文本位置，移动光标的时间复杂度是 **O(|新位置 - 旧位置|)**，是Gap 主要开销
3. 优点：局部编辑快（在光标附近连续的插入）；文本通过数组存储在内存中连续，访问更快；
4. 缺点：预留内存空间用于buffer 开销 ；光标移动距离过大时，复杂度为O(n)
# 应用
## 如何实现滚动歌词
1. 通过LazyColumn展示歌词列表，也就是LRC 文件列表；
2. 根据播放时间获取
## Compose的性能优化
1. 从 4 个方面来说
	1. 减少可组合函数的执行时间
		1.  通过 `remember` 缓存计算结果；
	2. 减小状态变更的次数
		1. 使用 `derivedStateOf` 派生状态；
	3. 缩小状态变更的造成重组的范围
		1. 为列表每个 item 设置唯一 key，重组时只需要修改单个 item；
	4. 减小重组时的变更量；
		1. 保持 SlotTable结构的稳定，比如常见的隐藏显示操作，通过AnimatedVisibility实现
	``` kotlin
	@Composable  
fun GoodExample(showDetails: Boolean) {  
    Column {  
        Text(”Title”)  
          
        // ✅ 结构是稳定的，只有可见性会变化  
        AnimatedVisibility(visible = showDetails) {  
            Column {  
                Text(”Detail 1”)  
                Text(”Detail 2”)  
                Button(onClick = {}) { Text(”Action”) }  
            }  
        }  
          
        Text(”Footer”)  
    }  
}
	
	```

# 参考链接
- [Jetpack Compose 【一】入门：拥抱现代 Android UI 开发](https://juejin.cn/post/7473316836362141696?searchId=202507122209440BB77DF9C87C1B5340FB)
- [Jetpack Compose 【二】状态管理详解](https://juejin.cn/post/7473278704052994082#heading-12)
- [Jetpack Compose 【三】附带效应、协程与异步](https://juejin.cn/post/7473325004031229967#heading-20)
- [Compose学习笔记（六）：CompositionLocal的应用场景](https://juejin.cn/post/7097890697721675813?searchId=20251017213352262EA4AC89474B93ACCB)
- [# 深入浅出 Compose 测量机制](https://juejin.cn/post/7558439013464899594)
- [妈！Jetpack Compose太难学了，别怕，这里帮你理清几个概念](https://juejin.cn/post/7244420350753144891?searchId=202507052141214BE0E8888B9B55CE12B4#heading-80)
- [鸿蒙 KMP Compose 源码万字简析](https://mp.weixin.qq.com/s?__biz=MzUyMjI4MzE1MA==&mid=2247484967&idx=1&sn=6496e0421d57ea0262ca10f5f7950c43&chksm=f9cf77c6ceb8fed0bd9295a62ed5cf9166b5df2a07e4a83828c4b020e4699cbacabc92604f13&cur_album_id=4008501025152761864&scene=189#wechat_redirect)
- [鸿蒙 KMP Compose 开发范式理解](https://mp.weixin.qq.com/s?__biz=MzUyMjI4MzE1MA==&mid=2247484957&idx=1&sn=e1f415b0cce0eb0eafcf031aaf7790ed&chksm=f9cf77fcceb8feea493de60229800eee81c01cf7486b99d7abaa4da29fdc2876edd0834a4e31&cur_album_id=4008501025152761864&scene=189#wechat_redirect)
- [Android 架构之 MVI 雏形 | 响应式编程 + 单向数据流 + 唯一可信数据源](https://juejin.cn/post/7087717477246369805)
- [Jetpack Compose UI创建布局绘制流程+原理 —— 内含概念详解（满满干货）](https://juejin.cn/post/7003606223438217223)
- [深入浅出 JetPack Compose UI 自动更新原理](https://blog.csdn.net/weixin_61845324/article/details/134268501)
- [Compose编程思想 -- 深入理解Compose原理](https://juejin.cn/post/7355307547479572521#heading-9)
- [开源一个企业可用的 Kotlin Multiplatform 项目模板](https://juejin.cn/post/7302338286768635956?searchId=202507292001582A7C0BD3C1624A8E74F1)
- [Compose · 重组的实现原理](https://mp.weixin.qq.com/s/0SgswfO6E7LdTUG9VIiV8Q?click_id=2)
- [深入理解 Jetpack Compose 内核：SlotTable 系统](https://blog.csdn.net/vitaviva/article/details/125478624)
- [Compose Recomposer启动流程分析](https://blog.csdn.net/datian1234/article/details/129042351)
- [Android使用Hilt依赖注入，让人看不懂你代码](https://juejin.cn/post/7294965012749320218?searchId=202511091431260B06F7D694BC937398B2#heading-16)
- [Jetpack Compose：理解 GapBuffer & SlotTable](https://mp.weixin.qq.com/s/4B_CeawUaZA94Dy3SsdlOg)
- [# Android ComposeUI详解](https://juejin.cn/post/7545936437914763315?searchId=20251119144539C3AACE0CFE8AD06BD801)
- [扒一扒 Jetpack Compose 实现原理](https://blog.csdn.net/weixin_43440181/article/details/128032539)
