# Shell Transition

## STATE_COLLECTING ：收集
1. Transition#mParticipants
![[Pasted image 20260428105118.png]]
2. Transition#mTargets
![[Pasted image 20260428143414.png]]

## 闪屏/winscope
1. 挂载在Task下的DimLayer，一个 Task 对应一个 Dim ，会关联到Task 下不同的 Activity
2. 闪白的情况就是 Dim Layer更新不及时；
3. RelativeLayer:Dim  Layer 关联到父图层，也就是对应Activity 图层，会跟随父图层的显示状态
4. zorder : 相对 z 轴，正数，显示在上方，负数显示在下方
5. A2 Window 图层已经隐藏（由父容器空气），但是它的状态没有隐藏
# WMS/AMS
## 窗口层级
1. 窗口分为 0～36 层，共 37 层；
2. RootWindowContainer → DisplayContent → DisplayArea → Task → ActivityRecord(WindowToken) → WindowState


## focusedWindows
1. mCurrentFocus：当前有焦点的窗口
2. mFocusedApp ：当前焦点的 Activity
3. WMS -> SurfaceFlinger -> InputDispatcher
## tips
1. createSurfaceController
2. dump window
3. 应用窗口如何被添加到层级树上？
4. dump surfaceFlinger  : layer 按照层级，focused
5. 冻屏：根据应用的包名和窗口类型禁止它添加 窗口（addWindow: 在系统层进行修改？， WindowManagerGlobal：addView 应用层修改）
# SurfaceFlinger

## fence

## perfetto 
1. 抓取命令
## V-sync
1. adb dump surfaceflinger --dispsync
2. 软件 v-sync 与 硬件 v-sync 的时间计算， sw-vsync
## BLASTBufferQueue
1. 初始化时同时创建生产者和消费者
2. BufferQueueCore
	1. mSlots：`list<BufferSlot>`，
		1. mGraphicBuffer
		2. mBufferState
	2. mQueue：`list<BufferItem>`
# 多任务
1. **TouchInteractionService（TIS）的特殊性**：它通过 **InputMonitor** 监听全局触摸，属于**系统级手势监视器**，优先级高于普通应用窗口。
2. **先收到 DOWN**（InputMonitor 优先级高）。
3. **原窗口也会收到 DOWN**（系统先广播 DOWN 给所有监听者，再判定所有权）。
4. 若系统手势拦截成功，**发送 CANCEL 给原窗口**，并将事件流重定向给 TIS。

# 重要类

## SurfaceControl
是Layer 的 Java 代理句柄，每个 SurfaceControl 对应 SurfaceFlinger 中的一个 Layer，管理该 Layer 的所有显示元数据（位置、Z 序、透明度、裁剪、缩放、旋转、可见性）。通过`layer_state_t`结构体进行描述；
## Transaction
1. 是一个独立的事务对象，保存 layer_state_t 集合，用于操作一个或多个 SurfaceControl 的属性；
2. merge 时以other 为准；
3. reparent(sc, newParent)： 重新设置父图层，子图层**所有属性会继承、跟随父图层**，由父层统一约束。
4. setLayer(sc, z)：设置 Z 轴层级（越大越上层）

## SurfaceFlinger
底层合成，接收 Shell 的 Surface 事务，硬件加速执行，保证帧同步（VSYNC）。

## WindowContainer
包含 SurfaceControl

# 其他

1. adb shell dumpsys window windows > /Users/joee/Documents/Joeeeee/framework/window.txt
2. adb shell dumpsys activity containers > /Users/joee/Documents/Joeeeee/frameworkcontainers.txt
3. dump surfaceFlinger
4. adb shell dumpsys window > /Users/joee/Android/window1.txt
5. adb shell dumpsys window lastanr