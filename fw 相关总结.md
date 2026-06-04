## Shell Transition 之Launcher冷启动应用
1. WMCore：运行在system_server进程的模块
	1. TransitionController：主要负责管理者整个过渡动画的生命周期，比如动画参与者收集，等待，启动等;
	2. Transition：具体过渡动画的实体类，它的主要生命周期包含收集中（collectiing），启动（started） ，播放中（playing），结束（finished)；
2. WMShell: 运行在SystemUI进程的模块，其核心类包含
	1. Transitions：主要负责相关过渡动画的具体播放相关逻辑
	2. ActiveTransition：具体过渡动画的实体类
3. 启动应用时，Launcher会构造ActivityOptions将RemoteTransition打包，RemoteTransition实现startAnimation方法接收leash用于同步播放窗口动画；
4. 进入到ATMS，ActivityStarter在启动对应Activity前，通过TransitionController创建Transition，将状态置为收集中，同时初始化SyncGroup；
	- 启动动画来说，会收集4个WindowContainer，也就是应用的ActivityRecord和Task，Launcher的ActivityRecord和壁纸，同时也会将 StartingWindow 加入到 应用的ActivityRecord中，跟随 ActivityRecord 的Surface 一起动画，这些窗口容器都会保存在 SyncGroup中，用于检测窗口的状态；
5. 之后通知Shell，完成ActiveTransition和TransitionHandler的初始化，同时Transition进入启动状态；
6. 等待WMS回调BlastSyncEngine判断SyncGroup中所有窗口都绘制完成，此时回调的来源Starting Window的绘制完成，回调core的onTransactionReady和shell的onTransitionReady，进入播放状态；
7. 获取ActivityOptions中RemoteTransition进行动画的播放，动画结束，回调finishCallback;
	1. 动画播放时操作启动事务
		1. startTransaction：过滤收集的WindowContainer，只保留Task信息和壁纸窗口，创建新的绘制树，将关联窗口转移到新的根节点，统一播放；
	2. 动画结束操作结束事务
		1. 将节点reparent到正常窗口树；
	3. 以启动过程为例：A ->B，动画播放前A,B都会显示；动画结束B显示，A隐藏；
## 首帧页面的显示流程
1. WMS首次添加Window时会构建一颗窗口层级树，层级分为 0～36 层，共 37 层，根节点为RootWindowContainer，第二层节点为DisplayContent对应屏幕数量，叶子节点为 WindowState，应用Activity对应的窗口路径为RootWindowContainer -> DisplayContent->TaskDisplayArea -> Task ->ActivityRecord -> WindowState;
2. AT在执行完Activity onResume 后，创建 ViewRootImpl，同时将onCreate 中 创建的 decorView 加入到集合中统一管理，VRI 是链接 View 体系与 WMS的桥梁；
3. VRI分别会调用addWindow，relayout，drawFinish方法通知 WMS；
4. addWindow 对应了窗口的创建，会新建WindowState 挂载到对应的ActivityRecord下；
5. SurfaceControl 的初始化：在 SurfaceFlinger 中创建对应的 Layer；
6. View 绘制完成：
	1. UI线程将 View 树录制为 DisplayList（RenderNode 树)，并通过将syncAndDrawFrame将 RenderNode 树从 UI 线程交给 渲染线程，
	2. 渲染线程申请 Buffer（dequeueBuffer/requestBuffer）完成 GPU渲染后， 再将Buffer传递（queueBuffer）给 SurfaceFlinger;
	3. 同时通过把合并了窗口几何状态（位置、裁剪、变换等）的 Transaction 发给 WMS；
7. SurfaceFlinger 等待两者就绪：
	- BufferQueue 中有可用的 buffer
    - SurfaceControl 的状态（由 WMS 的 Transaction 配置）已生效
## Activity 的启动流程
1. Launcher发起启动请求，Instrumentation 判断是否需要返回值，获取 ATMS 服务，发起 Binder 请求对应HOME Activity;
2. ATMS解析Intent参数，进行权限校验，并计算 Task 标记，Launcher启动应用都会携带 new_task 标记，新建Task加入到DisplayArea 中；
3. 创建 Pause事务暂停 Launcher，同时通过socket请求zygote创建应用进程；
4. zygote fork出子进程后，通过反射创建ActivityThread对象，同时执行其入口main方法；
5. 在main方法中，启动应用的Binder服务ApplicationThread ，并返回给AMS，同时启动主线程Looper，开始消息循环；形成AMS -> ApplicationThread ->Handler通信链；
6. AMS会调用ApplicationThread的bindApplication方法，向主线程中发送bindApplication消息，启动Application；
7. 当Launcher pause完成同时Application启动完成，AMS才开始真正启动 Activity；
8. 将启动事务和Resume事务打包后，调用ApplicationThread，发送EXECUTE_TRANSACTION消息进行处理;	
9. 当执行到 resume 时，新建 ViewRootImpl，将 View与 Window 绑定，申请 V-sync 信号，触发 View 的测量、布局、绘制流程，之后 Activity 才真正对用户可见；

## 一帧数据的绘制流程
1. 应用的`View.invalidate()`、动画、数据变化或输入事件触发会调用到ViewRootImpl.scheduleTraversals，
2. 应用接受到app类型的V-sync信号，唤醒等待中的UI线程，`Choroegrapher`回调`onVsync`开始一帧的绘制，依次处理Input事件，animation动画，performTraversals，其中traversal包含View的测量， 布局和绘制；
3. 绘制完成后，更新绘制数据，结束一帧的绘制，继续处理下一帧的Message，同时通过postAndWait唤醒渲染线程执行界面渲染任务。
4. 渲染线程先同步UI线程构建好的绘制命令树，然后通过dequeueBuffer申请一张处于free状态的buffer，进行GPU渲染，渲染完成后swipBuffer触发queueBuffer动作上帧；
5. 渲染线程通过queueBuffer唤醒对端的SurfaceFlinger进程中的Binder工作线程，申请sf类型的vsync信号；
6. sf类型的VSYNC信号到达后后，sf开始执行一帧的合成任务，之后再执行present唤醒HWC service进程执行图层合成送显；
## 无焦点窗口的ANR问题
1. 首先还是确认 ANR的时间点： 结合dump window input 信息的 lastAnr 信息进一步确认；  
2. 焦点的流转涉及 WMS -> SurfaceFlinger  -> InputDispatcher
3. 对于 WMS ：
	1. 通过 dump windows信息，获取每个窗口的绘制状态和焦点窗口信息和焦点应用信息，对于普通应用来说，焦点窗口通常就是焦点应用。（例外：下拉通知栏）；
	2. 从日志中过滤 Changing Focus，从桌面启动应用，focus 变化从 Launcher 到 null 再到对应的 App;
		1. Activity的可见性通过2个字段描述mVisibleRequested 和visible
		2. A1启动A2，A1的pause成功回调，调用A1和A2对应ActivityRecord 的setVisibility方法，将mVisibleRequested分别置为false和true;
		3. A2的startingWindow 添加成功，由于startingWindow 不处理事件，将当前的焦点窗口置为空；
		4. 当A2应用进程创建成功并完成Application创建与绑定后，将焦点应用设置为A2，执行真正的启动Activity方法，执行完resume方法，通过VRI，完成窗口的创建和relayout后 将焦点设置为应用窗口；
		5. 真正表示应用窗口可见的visible属性，
			1. 对于A2来说，需要等到startingWindow绘制完成，在启动的Transition动画开始前，设置为true;
			2. 对于A1来说，需要Transition动画就结束后，通过Transition动画结束后，finishTransition回调来提交隐藏；
4. 对于 SurfaceFlinger
	1. 焦点信息作为 Transaction 的一部分，与窗口属性一起**原子性地**提交给 SurfaceFlinger，由 SF 统一转发给 InputFlinger，用来保证了渲染与输入的同步一致；
5. InputDispatcher 结合event 日志中的 `input_focus` tag
	1. Requesting ：Launcher pause执行完，请求将focus置空；
	2. Focus leaving ：焦点离开Launcher
	3. Focus Request ： 应用resume完成，窗口relayout完成，请求焦点；
	4. Focus entering： InputDispatcher完成焦点的更新；
## 闪屏问题
1. main -> DialogActivity1 -> DialogActivity2 : D2 退出
2. Winscope: indowManager 和录屏没有对应，抓 SurfaceFlinger 即可，去掉输入法