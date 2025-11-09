# 声明式UI
1. 描述 UI 与状态的关系状态，当状态发生变化下，UI自动更新；
2. 如何实现 Button点击，TextView 计数的功能
	1. 在命令式 UI中，需要监听 Button的点击，点击时，修改 TextView 的显示
	2. 而在声明式 UI 中，Button 点击修改状态，TextView 描述状态的关系，自动响应状态变化。
## MVI
1. Model：表示 UI的状态，是唯一的数据源；
2. View：接收一个 状态对象并渲染 UI，并将用户输入作为 Intent 发送
3. Intent：表示用户的操作意图（如点击按钮、输入文本）
4. ViewModel：接受 Intent，处理业务逻辑更新 State;
5. MVI 强调的是“唯一可信数据源”与“单向数据流”
	1. 唯一可信数据源：
		1. 唯一的意思不是说一个页面一个状态，而是将界面分成若干个相互独立的状态，控件状态不会相互影响；
		2. 可信的意思是它是线程安全且不对外暴露修改方法，在 Android 中的实践就是采用 val 修饰，多线程写变量的时先拷贝原变量，然后对副本写。
		3. 单向数据流是指数据只能沿着一个方法流动，形成闭环。在 MVI 中就是View 产生 Intent ，ViewModel 接收 Intent，触发状态更新后 View 重新渲染；
6. 和 MVVM相比，严格限制单向数据流和唯一可信数据源，结合 Composable 函数，更容易测试
# Compose 的 UI 组织形式
1. @Copmposable 函数经过编译后生成的函数会添加 2 个参数，类型为 Composer 与 Int。
	- Composer 的作用就是就是将 UI 结构，状态信息记录到 SlotTable 中，Int 值用于标记是否需要参与重组；
2. 函数内部插入了很多 `$composer.startXXXGroup` 和 `$composer.endXXXGroup` 模板代码；
	1. 函数首行插入startRestartGroup，插入一个可重复执行的 Group，并会创建  `RecomposeScopeImpl` 并存入 SlotTable，它会记录当前组合函数，重组时通过通过它来重新执行组合函数 。
	2. 在 UI节点插入startReusableNode，插入一个包含 LayoutNode 的 Group;
	3. startReplaceableGroup，插入一个可以被替换的 Group,一个 if/else 代码就是一个ReplaceableGroup；
3. SlotTable 的数据存储在 Slot 中，一个或多个 Slot 又归属于同一个 Group；
	- 多个LayoutNode 组合生成 Compose 的渲染树，完成测量，布局和绘制工作；
	- UI 的变化本质是触发更新 LayoutNode 树，重新渲染布局；
# 副作用
副作用（Side Effects）是指在 UI 渲染之外执行的操作，如日志记录、网络请求、数据库操作等；

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

# 重组
1. 重组的本质：当状态（State）改变时，Recomposer 调度受该状态影响的组合函数重新执行。这个过程就是“重组”。在这个过程中，涉及到 4个对象：State/Recomposer/Composer/RecomposeScopeImpl；
2.  State状态会关联快照，组合成 SnapshotState，快照有 2 个作用
	1. 对状态的读写进行隔离，组合时会创建子快照，子快照内对状态的修改需要 主动调用apply提交到全局快照；
	2. 状态读写感知，组合函数内对状态的读写都会被记录，作为重组的依据；
3. Recomposer ：setContent时创建并初始化，启动时注册全局快照状态监听，并开启协程监听 Vsync 信号，发起并驱动重组流程，无变更内容时会挂起等待；
4. Composer：在组合和重组期间将 UI结构、状态等信息记录到 SlotTable 中，同时会创建  `RecomposeScopeImpl` 并存入 SlotTable
5. RecomposeScopeImpl：重组作用域，记录了对应的组合函数，当其读取的状态变化时，Compose 会从 SlotTable找到它，标记失效，重组时重新执行组合函数；
## 自定义布局
Compose 实现自定义布局有 4 种方式
### Canvas
使用 Canvas自定义绘制，类似于传统 View 的 onDraw 方法，通过 drawRect drawCircle drawPath 等方法绘制各种图形；
### Layout
使用 Layout自定义布局，类似于传统 View 的 onMeasure与 onLayout的结合，核心 API 是measurable与placeable用于测量大小和拜访位置
	1. 流式布局

### Modifier.drawWithContent
Modifier 的drawWithContent方法，在已有布局上叠加效果
### Android View
 通过 `AndroidView`，你可以将传统的 `View` 嵌入 Compose 布局中，复用 XML 布局中已有的自定义视图。
# 测量
1. 和原生视图类似，Compose也存在一棵渲染树，树上的节点是LayoutNode，通过 LayoutNode 完成 measure/layout/draw 的过程最终将 UI 显示到屏幕上；
2. LayoutNode在创建时会指定measurePolicy与 Modifier;
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
## LazyVerticalStaggeredGrid
1. 创建交错网格布局的组件,每个布局对应一个 item；

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
1. 通过 expect/actual 机制声明统一接口与平台实现;
# hilt
## 注入点（类）
1. 除了Application和ViewModel，其它注入点都是通过使用@AndroidEntryPoint修饰。
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
# 参考链接
- [Jetpack Compose 【三】附带效应、协程与异步](https://juejin.cn/post/7473325004031229967#heading-20)
- [Compose学习笔记（六）：CompositionLocal的应用场景](https://juejin.cn/post/7097890697721675813?searchId=20251017213352262EA4AC89474B93ACCB)
- [# 深入浅出 Compose 测量机制](https://juejin.cn/post/7558439013464899594)
- [妈！Jetpack Compose太难学了，别怕，这里帮你理清几个概念](https://juejin.cn/post/7244420350753144891?searchId=202507052141214BE0E8888B9B55CE12B4#heading-80)
- [鸿蒙 KMP Compose 源码万字简析](https://mp.weixin.qq.com/s?__biz=MzUyMjI4MzE1MA==&mid=2247484967&idx=1&sn=6496e0421d57ea0262ca10f5f7950c43&chksm=f9cf77c6ceb8fed0bd9295a62ed5cf9166b5df2a07e4a83828c4b020e4699cbacabc92604f13&cur_album_id=4008501025152761864&scene=189#wechat_redirect)
- [鸿蒙 KMP Compose 开发范式理解](https://mp.weixin.qq.com/s?__biz=MzUyMjI4MzE1MA==&mid=2247484957&idx=1&sn=e1f415b0cce0eb0eafcf031aaf7790ed&chksm=f9cf77fcceb8feea493de60229800eee81c01cf7486b99d7abaa4da29fdc2876edd0834a4e31&cur_album_id=4008501025152761864&scene=189#wechat_redirect)
- [Jetpack Compose 【一】入门：拥抱现代 Android UI 开发](https://juejin.cn/post/7473316836362141696?searchId=202507122209440BB77DF9C87C1B5340FB)
- [Jetpack Compose 【二】状态管理详解](https://juejin.cn/post/7473278704052994082#heading-12)
- [Android 架构之 MVI 雏形 | 响应式编程 + 单向数据流 + 唯一可信数据源](https://juejin.cn/post/7087717477246369805)
- [Jetpack Compose UI创建布局绘制流程+原理 —— 内含概念详解（满满干货）](https://juejin.cn/post/7003606223438217223)
- [深入浅出 JetPack Compose UI 自动更新原理](https://blog.csdn.net/weixin_61845324/article/details/134268501)
- [Compose编程思想 -- 深入理解Compose原理](https://juejin.cn/post/7355307547479572521#heading-9)
- [开源一个企业可用的 Kotlin Multiplatform 项目模板](https://juejin.cn/post/7302338286768635956?searchId=202507292001582A7C0BD3C1624A8E74F1)
- [Compose · 重组的实现原理](https://mp.weixin.qq.com/s/0SgswfO6E7LdTUG9VIiV8Q?click_id=2)
- [深入理解 Jetpack Compose 内核：SlotTable 系统](https://blog.csdn.net/vitaviva/article/details/125478624)
- [Compose Recomposer启动流程分析](https://blog.csdn.net/datian1234/article/details/129042351)
- [Android使用Hilt依赖注入，让人看不懂你代码](https://juejin.cn/post/7294965012749320218?searchId=202511091431260B06F7D694BC937398B2#heading-16)
