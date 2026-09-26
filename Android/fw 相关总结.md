
## 近期任务
1. InputMonitorCompat("swipe-up", displayId)：注册 InputMonitor 监听事件输入，检测触摸是否是在手势位置  **~~(`isInSwipeUpTouchRegion`) 和位移追踪 (`passedSlop`) 完成，~~** 调用pilferPointers拦截后续事件；
2.  满足条件 启动 RecentTransition，收集对应窗口， Launcher绘制完回调：Transition.onTransactionReady，开始动画的播放，接收对应应用的 Leash完成动画的播放；
	1. Transition:handleLegacyRecentsStartBehavior
3. AbsSwipeUpHandler.startInterceptingTouchesForGesture 
	1. RecentsTransitionHandler#setInputConsumerEnabled
		1. 更新 foucusedWindow

# 图层泄露
1.  dumpSurfaceFlinger 定位具体泄露的 layer ,确定对应的 Surfacecontrol：surfaceControl 会指定名称
2. 看下具体的图层的业务结束有没有调用 release
3. 存在多个进程引用的图层就比较难排除，但是整体思路还是需要看是否有调用 release
4. monkey
## 其他
1. CompositionEngine::present: 打印 Layer
2. 在 事务中增加打印(hide,show,remove)，通过过滤指定的图层名称，图层的名称就是 winscope 中的名称
3. winscope 的 SurfaceFlinger 信息;
4. `onFrameDraw(syncResult, frame)` 在 RenderThread 中 `syncFrameState()` 之后、`context->draw()` 之前


# 自我介绍
你好，我叫李乔，2014 年毕业于湖北大学，毕业后先后从事过智能硬件，互联网经融以及系统应用的开发。上 2 份工作都是系统桌面相关开发，包括桌面的基础体验功能如拖拽，编辑模式。桌面相关的商业化功能，如文件夹底部推荐应用、抽屉智能搜素、浏览器底部插件等。实现文件夹广告的插件化方案，商业化相关应用的定屏、黑屏、闪屏问题以及 ANR 问题攻关。在之前的工作中还对导应用的启动优化、动态化换肤方案的落地。以上就是我的自我介绍。


# 抽屉滑动优化
