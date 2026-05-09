# Shell Transition
## 介绍启动动画
1. WMCore：运行在system_server进程的模块
	1. TransitionController：主要负责管理者整个过渡动画的生命周期，比如动画参与者收集，等待，启动等;
	2. Transition：具体过渡动画的实体类，它的主要生命周期包含收集中（collectiing），启动（started） ，播放中（playing），结束（finished)；
2. WMShell: 运行在SystemUI进程的模块，其核心类包含
	1. Transitions：主要负责相关过渡动画的具体播放相关逻辑
	2. ActiveTransition：具体过渡动画的实体类
3. 启动应用时，Launcher会构造ActivityOptions将RemoteTransition打包，其startAnimation方法需要接受leash用于同步播放动画；
4. ActivityStarter在启动对应Activity前，TransitionController就会创建Transition，将状态置为收集中，同时初始化SyncGroup；
	- 对于启动动画来说，会收集4个WindowContainer，也就是启动应用的ActivityRecord和Task，Launcher的ActivityRecord和壁纸；
5. 之后通知Shell，完成ActiveTransition和TransitionHandler的初始化，同时Transition进入启动状态；
6. SplashScreen Surface 随 ActivityRecord 的 Surface 层级 一起运动
7. 等待BlastSyncEngine回调判断SyncGroup所有的窗口绘制完成，绘制完成回调Transition的onTransactionReady和Transitions的onTransitionReady的，进入播放状态；
8. 获取ActivityOptions将RemoteTransition进行动画的播放，动画结束，回调finishCallback;
9. 准备startTransaction 和finishTransaction 
	1. startTransaction：过滤收集的WindowContainer，只保留Task信息，创建新的绘制树，将关联Task转移到新的根节点，统一播放；
	2. finishTransaction ，动画结束后回调，将节点reparent到正常窗口树；

## Shell Transition流程
1. TransitionController#createAndStartCollecting：创建Transition，状态置为STATE_COLLECTING；
2. Transition.onTransactionReady: 状态置为STATE_PLAYING

## 收集
1. Transition#mParticipants
![[Pasted image 20260428105118.png]]
2. Transition#mTargets
![[Pasted image 20260428143414.png]]
## 流程
1. 启动时，桌面构造 ActivityOptions 将RemoteTransition打包，其startAnimation方法会传递leash给桌面播放动画；
2. ActivityStarter启动对应Activity前，会创建Transition，将状态置为COLLECTING，同时初始化SyncGroup；
3. Transition通过一个集合保存要收集WindowState对象，一次启动Activity，会收集ActivityRecord,Task，壁纸对应的WindowState;
4. 当BlastBufferQueue
## 闪屏/Winscope
1. RelativeLayer/zorder
# WMS/AMS
1. adb shell dumpsys activity containers： dump 窗口层级树；
2. adb shell dumpsys window： dump window 信息
## 窗口层级
1. 窗口分为 0～36 层，共 37 层；
## 应用窗口的添加流程
- WMS首次添加Window时会构建一颗窗口层级树，层级分为 0～36 层，共 37 层，根节点为RootWindowContainer，第二层节点为DisplayContent，Activity为层级为TaskDisplayArea -> Task ->ActivityRecord -> Windowstate，其他窗口则为DisplayArea .Tokens ->WindowToken -> WindowState;
- 当Activity进入onResume生命周期后，创建ViewRootImpl，并在其performTraversel中完成对窗口add，relayout，draw和
- `ActivityRecord` 作为 app token，早已在 `Task/TaskFragment` 层级中（例如 `TaskFragment.addChild(ActivityRecord)`）。
- 所以最终层级是：`RootWindowContainer -> DisplayContent -> Task/TaskFragment -> ActivityRecord(WindowToken) -> WindowState`。
- `addWindow` 完成加入后会更新焦点、输入窗口、层级分配；真正出图还要后续 `relayout`
## 绘制
1. mDrawState 的 5 个状态

## Activity的启动流程
1. Launcher onPause 与 realStartActivity
2. dump activity container :层级树分析
3. DefaultDisplay
### 面试
1. Launcher判断是否需要处理 ActivityResult 后，获取ATMS的服务，启动Activity；
2. ATMS解析Intent参数，进行权限校验， 通过后，创建 ActivityRecord和 Task 信息加入到根节点；
3. 创建 Pause事务暂停 Launcher，同时通过socket请求zygote创建应用进程；
4. zygote fork出子进程后，通过反射创建ActivityThread对象，同时执行其入口main方法；
5. 在main方法中，启动应用的Binder服务ApplicationThread ，并返回给AMS，同时启动主线程Looper，开始消息循环；形成AMS -> ApplicationThread ->Handler通信链；
6. AMS会调用ApplicationThread的bindApplication方法，向主线程中发送bindApplication消息，启动Application；
7. 对于Activity的相关生命周期方法，则封装成对应事务后，统一发送EXECUTE_TRANSACTION消息进行处理;	
8. 当执行到 onResume 时，会调用WindowManager.addView方法，将 DecorView添加到 WindowManager中。触发 View 的测量、布局、绘制流程，此时 Activity 才对用户可见；

## 1
1. createSurfaceController
2. dump window
3. 应用窗口如何被添加到层级树上？
# SurfaceFlinger
## SurfaceControl
是Layer 的 Java 代理句柄，每个 SurfaceControl 对应 SurfaceFlinger 中的一个 Layer，管理该 Layer 的所有显示元数据（位置、Z 序、透明度、裁剪、缩放、旋转、可见性）。通过`layer_state_t`结构体进行描述；
## Transaction
1. 是一个独立的事务对象，保存 layer_state_t 集合，用于操作一个或多个 SurfaceControl 的属性；
2. merge 时以other 为准；
## perfetto 
1. 抓取命令
## V-sync
1. adb dump surfaceflinger --dispsync
2. 软件 v-sync 与 硬件 v-sync 的时间计算， sw-vsync
## 一帧数据的绘制
1. 应用的`View.invalidate()`、动画、数据变化或输入事件触发会调用到ViewRootImpl.scheduleTraversals，
2. 应用接受到app类型的V-sync信号，唤醒等待中的UI线程，`Choroegrapher`回调`onVsync`开始一帧的绘制，依次处理Input事件，animation动画，performTraversals，其中traversal包含View的测量， 布局和绘制；
3. 绘制完成后，更新绘制数据，结束一帧的绘制，继续处理下一帧的Message，同时通过postAndWait唤醒渲染线程执行界面渲染任务。
4. 渲染线程先同步UI线程构建好的绘制命令树，然后通过dequeueBuffer申请一张处于free状态的buffer，进行GPU渲染，渲染完成后swipBuffer触发queueBuffer动作上帧；
5. 渲染线程通过queueBuffer唤醒对端的SurfaceFlinger进程中的Binder工作线程，申请sf类型的vsync信号；
6. sf类型的VSYNC信号到达后后，sf开始执行一帧的合成任务，之后再执行present唤醒HWC service进程执行图层合成送显；
# 重要类
## SurfaceControl

## SurfaceControl.Transaction
1. reparent(sc, newParent)： 重新设置父图层，子图层**所有属性会继承、跟随父图层**，由父层统一约束。
2. setLayer(sc, z)：设置 Z 轴层级（越大越上层）
## SurfaceFlinger
底层合成，接收 Shell 的 Surface 事务，硬件加速执行，保证帧同步（VSYNC）。

## WindowContainer
包含 SurfaceControl