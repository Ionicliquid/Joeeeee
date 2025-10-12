
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

# MVI
![[mvi.png]]

### 核心思想：构建一个完全封闭和可预测的数据循环
1. Model：表示 UI的状态（State），是唯一的数据源；
2. View：Composable 函数，负责渲染 State，并将用户的操作转换为意图（Intent）;
3.  Intent：用户的操作意图（如点击按钮、输入文本），它不会直接修改状态；
### 优点
1. 极度可预测：数据永远单向流动，状态来源清晰，调试非常方便；
2. 唯一数据源：整个 UI 只依赖一个 State 对象，状态一致性得到保证
3. 易于测试：可以独立测试ViewModel 逻辑：给定一个初始 State和一个 Intent，断言最终的 State 是否符合预期；
## MVVM
![[mvvm.png]]
## 对比

| 特性   | MVI                    | MVVM (在Compose中)               |
| ---- | ---------------------- | ------------------------------ |
| 数据流  | 严格单向 (Intent -> State) | 通常也是单向 (Event -> State)，但约束更松散 |
| 状态管理 | 强制单一State对象            | 推荐单一State对象，但也可以暴露多个StateFlow  |
| 核心理念 | 强调“意图”，用户行为驱动          | 强调“数据绑定”，状态驱动                  |
| 官方推荐 | Google官方首推             | 广泛接受且完全兼容                      |
| 适用场景 | 复杂状态管理、需要高可追溯性的页面      | 各种场景，尤其适合从传统View迁移的项目          |
# 声明式 UI

1. 你只需要描述 UI 在特定状态下应该是什么样子，而不需要关心它是如何从上一个状态变化过来的；
2. 如何实现 Button点击，TextView 计数的功能
	1. 在命令式 UI中，需要监听 Button的点击，修改 TextView 的显示
	2. 而在声明式 UI 中，Button 点击修改状态，TextView 描述状态的关系，自动响应状态变化。

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
11. [Compose · 重组的实现原理](https://mp.weixin.qq.com/s/0SgswfO6E7LdTUG9VIiV8Q?click_id=2)
12. [# 深入理解 Jetpack Compose 内核：SlotTable 系统](https://blog.csdn.net/vitaviva/article/details/125478624)
13. [# 19.2 Compose Recomposer启动流程分析](https://blog.csdn.net/datian1234/article/details/129042351)



