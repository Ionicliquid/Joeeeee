
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


## 布局

布局部分和具体的 UI 节点相关，不同平台实现大同小异不再冗余阐述。

- 触发布局：一旦 LayoutNode 树被修改（无论是属性更新、插入、删除还是移动），这些被影响的节点及其父节点会被标记为“需要重新布局”。
- 执行布局：布局阶段开始，Compose 会从被标记的最高层节点开始，重新测量和定位受影响的 LayoutNode。
- 执行绘制：布局完成后，任何在屏幕上位置或外观发生变化的 LayoutNode 会被重新绘制到屏幕上。  
## 绘制

绘制阶段的核心思想是 录制 和 回放，将绘制指令录制到 RenderNode displayList 中，在 RenderThread 中完成回放，RenderThread 调用 Skia 转译指令绘制到 Vulkan
![[绘制.png]]

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



