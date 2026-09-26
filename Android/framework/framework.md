# Shell Transition

## STATE_COLLECTING ：收集
1. Transition#mParticipants
![[Pasted image 20260428105118.png]]
2. Transition#mTargets
![[Pasted image 20260428143414.png]]

## merge 
1. 同一 Track 内的 Transition 本来就是允许串行的，并且按照类型进行merge
## 闪屏/winscope
1. 挂载在Task下的DimLayer，一个 Task 对应一个 Dim ，会关联到Task 下不同的 Activity
2. 闪白的情况就是 Dim Layer更新不及时；
3. RelativeLayer:Dim  Layer 关联到父图层，也就是对应Activity 图层，会跟随父图层的显示状态
4. zorder : 相对 z 轴，正数，显示在上方，负数显示在下方
5. A2 Window 图层已经隐藏（由父容器控制），但是它的状态没有隐藏
# WMS/AMS
## 窗口层级
1. 窗口分为 0～36 层，共 37 层；
2. RootWindowContainer → DisplayContent → DisplayArea → Task → ActivityRecord(WindowToken) → WindowState
3. 对于 WMS ：
	1. 通过 dump windows信息，获取每个窗口的绘制状态和焦点窗口信息和焦点应用信息，对于普通应用来说，焦点窗口通常就是焦点应用。（例外：下拉通知栏）；
		1. 从日志中过滤 Changing Focus，
		2. Changing Focus调用最常见的来源就是应用启动， relayoutWindow，也就是窗口添加成功之后在WMS 处理窗口属性和 创建 SurfaceControl；
		3. focusedTask 的更新
4. key 事件派发是需要焦点窗口，触摸事件是按照窗口的层级顺序进行查找，
	1.  touch 事件根据点击位置找到目标窗口再分发事件，事件处理超时则触发 ANR，没有找到目标窗口就 drop event。
	2. key 事件分发时，从记录的数据中查找焦点窗口，如果没有找到，inputdispatcher 线程 epoll_wait 休眠，超时时间到达后，从记录的数据中再次查找焦点窗口，如果还没有找到则触发 ANR。
5. ActivityRecord通过2个字段描述可见性，visible和requestVisible。
	1. requestVisible：称为预期可见性，A1启动A2，A1 pause成功后将分别A1，A2的requestVisible置为false，true，false 代表这个窗口不能成为焦点窗口；
	2. Transition 动画开始时，A2 窗口已经绘制完成，将 A2 的 visible 置为 true，动画结束时，将 A1 visible 置为 false，它表示的图层可见性；
## winScope
1. winScope: windowManager 和录屏没有对应，抓 SurfaceFlinger 即可，去掉输入法
## focusedWindows
1. mCurrentFocus：当前有焦点的窗口
2. mFocusedApp ：当前焦点的 Activity
3. WMS -> SurfaceFlinger -> InputDispatcher
4. createSurfaceController
5. dump window
6. 应用窗口如何被添加到层级树上？
7. dump surfaceFlinger  : layer 按照层级，focused
8. 冻屏：根据应用的包名和窗口类型禁止它添加 窗口（addWindow: 在系统层进行修改？， WindowManagerGlobal：addView 应用层修改）
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
# PIP
## 多任务
1. **TouchInteractionService（TIS）的特殊性**：它通过 **InputMonitor** 监听全局触摸，属于**系统级手势监视器**，优先级高于普通应用窗口。
2. **先收到 DOWN**（InputMonitor 优先级高）。
3. **原窗口也会收到 DOWN**（系统先广播 DOWN 给所有监听者，再判定所有权）。
4. 若系统手势拦截成功，**发送 CANCEL 给原窗口**，并将事件流重定向给 TIS。
## 桌面手势

1. Launcher 准备启动 recents 动画
2. 调用 InputConsumerController.getRecentsAnimationInputConsumer()
       .registerInputConsumer()  ←── 注册 input consumer
3. Launcher 启动 recents animation transition
4. 动画运行中，通过 setInputConsumerEnabled(true) 启用触摸接收
5. 动画结束，调用 unregisterInputConsumer() 注销
## 自由窗
遇到的问题：
1. 首次启动闪屏
		1. 来自 Splash Screen - 初始化自有窗大小的时机
2. 如何保持自由窗始终显示在前端
		1. Task 是挂载在 DisplayArea 上，位置的调整是通过它的 positionChildAt
		2. Task 的 setAlwaysOnTop 方法可以让 Task 放置在顶端，优先级比较？
		3. Activityoption？没有生效？
3. FLAG_ACTIVITY_LAUNCH_ADJACENT
4. resumeFocusedTasksTopActvities- >ensureVisibilityAndConfig;
		1. 底部的 Activity 也不会 pause ? resumeTopActivity ?
		2. resumeTop 时 尝试 pause ，遍历所有 Activity，根据其他 Activity  mode 判断，不会隐藏当前的 Activity ;
# Input
1. getevent -lrt ： 驱动往应用空间写入数据；
## InputManager
1. 跟随 系统启动 InputManagerService
	1. 在 native 层还添加了：inputFlinger
	2. 运行InputReader/InputDispatcher 线程
	3. EventHub绑定 InputReader：epoll 机制直接与底层打交道
## InputReader
1. EventHub.getEvents
## InputDispatcher
1. 手势GlobalMonitor :总是会接受到事件
## ANR
1. 应用端没有在指定时间内发送 finish 时间
	1. 第一次卡了 500ms：应用还没准备好
	2. 
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
2. adb shell dumpsys activity containers > /Users/joee/Documents/Joeeeee/framework/containers.txt
3. adb shell dumpsys SurfaceFlinger > /Users/joee/Android/SurfaceFlinger.txt
4. adb shell dumpsys window > /Users/joee/Android/window1.txt
5. adb shell dumpsys window lastanr
6. Proto 日志： adb shell wm logging enable-text WM_DEBUG_BACK_PREVIEW
	1. WM_SHOW_TRANSACTIONS
	2. WM_DEBUG_FOCUS_LIGHT : "Changing Focus"
	3. WM_DEBUG_FOCUS: "Looking Focus"
	4. WM_DEBUG_WINDOW_TRANSITIONS_MIN
	5. core: adb shell wm logging enable-text TAG
	6. shell : adb shell dumpsys activity service SystemUIService WMShell protolog enable-text TAG
7. `input_focus的Event日志` ：adb logcat -b events -c && adb logcat -b events -v threadtime | grep input_focus 
8. aosp 编译
	1. source  build/envsetup.sh
	2. lunch sdk_phone16k_arm64-aosp_current-userdebug
9. 启动模拟器
```
ANDROID_PRODUCT_OUT=/Users/joee/OrbStack/my-x86-vm/home/joee/aosp16/out/target/product/emu64a16k \

ANDROID_BUILD_TOP=/Users/joee/OrbStack/my-x86-vm/home/joee/aosp16 \

emulator -verbose -show-kernel -no-snapshot -gpu swiftshader_indirect
```