# Shell Transition
## 介绍启动动画
1. WMCore：运行在system_server进程的模块
	1. TransitionController：主要负责管理者整个过渡动画的生命周期，比如动画参与者收集，等待，启动等;
	2. Transition：具体过渡动画的实体类，它的主要生命周期包含收集中（collectiing），启动（started） ，播放中（playing），结束（finished)；
2. WMShell: 运行在SystemUI进程的模块，其核心类包含
	1. Transitions：主要负责相关过渡动画的具体播放相关逻辑
	2. ActiveTransition：具体过渡动画的实体类
3. 启动应用时，Launcher会构造ActivityOptions将RemoteTransition打包，RemoteTransition实现startAnimation方法接受leash用于同步播放窗口动画；
4. ActivityStarter在启动对应Activity前，TransitionController就会创建Transition，将状态置为收集中，同时初始化SyncGroup；
	- 对于启动动画来说，会收集4个WindowContainer，也就是应用的ActivityRecord和Task，Launcher的ActivityRecord和壁纸，同时也会将 StartingWindow 加入到 应用的ActivityRecord中，跟随 ActivityRecord 的Surface 一起动画；
5. 之后通知Shell，完成ActiveTransition和TransitionHandler的初始化，同时Transition进入启动状态；
6. 等待WMS回调BlastSyncEngine判断SyncGroup中所有窗口的Surface已经完成摆放，回调core的onTransactionReady和shell的onTransitionReady，进入播放状态；
7. 获取ActivityOptions将RemoteTransition进行动画的播放，动画结束，回调finishCallback;
8. 准备startTransaction 和finishTransaction 
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

## 闪屏/winscope
1. RelativeLayer/zorder
# WMS/AMS
## 窗口层级
1. 窗口分为 0～36 层，共 37 层；
2. RootWindowContainer → DisplayContent → DisplayArea → Task → ActivityRecord(WindowToken) → WindowState
## 应用窗口的显示流程
1. WMS首次添加Window时会构建一颗窗口层级树，层级分为 0～36 层，共 37 层，根节点为RootWindowContainer，第二层节点为DisplayContent对应屏幕数量，叶子节点为 WindowState，应用Activity对应的窗口路径为RootWindowContainer -> DisplayContent->TaskDisplayArea -> Task ->ActivityRecord -> WindowState;
2. AT在执行完Activity onResume 后，创建 ViewRootImpl，同时将onCreate 中 创建的 decorView 加入到集合中，VRI 是链接 View 体系与 WMS的桥梁；
3. VRI分别会调用add，relayout，draw方法通知 WMS，对应窗口的创建，SurfaceControl 的初始化，View 的绘制完成通知；
	1. 窗口的创建就是新建WindowState 挂载到对应的ActivityRecord下；
	2. SurfaceControl 的初始化：在 SurfaceFlinger 中创建对应的 Layer；
	3. View 绘制完成：将 View 树录制为 DisplayList（RenderNode 树)；
4. 
## ThreadedRenderer 绘制 View 内容的流程

整个管线分为两个阶段：**录制阶段**（UI 线程）和**渲染阶段**（Render 线程）。

### 1. 入口：`ThreadedRenderer.draw()`

由 `ViewRootImpl.performDraw()` 调用 → [ThreadedRenderer.java:828](vscode-webview://0ehv9b7aq8bm0bkh8crpv71gesmvhg8egl0k1jo51it32i693gcf/core/java/android/view/ThreadedRenderer.java#L828)。

```java
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks) {
    updateRootDisplayList(view, callbacks);   // 阶段1: 录制
    syncAndDrawFrame(frameInfo);              // 阶段2: 渲染
}
```

### 2. 录制阶段：将 View 树录制为 DisplayList（RenderNode 树）

`updateRootDisplayList()` → [ThreadedRenderer.java:731](vscode-webview://0ehv9b7aq8bm0bkh8crpv71gesmvhg8egl0k1jo51it32i693gcf/core/java/android/view/ThreadedRenderer.java#L731) 做两件事：

**a) 递归更新每个 View 的 DisplayList**

调用 `view.updateDisplayListIfDirty()` → [View.java:24064](vscode-webview://0ehv9b7aq8bm0bkh8crpv71gesmvhg8egl0k1jo51it32i693gcf/core/java/android/view/View.java#L24064)，该方法：

- 在 View 自己的 `RenderNode` 上打开一个 `RecordingCanvas`
- 调用 `view.draw(canvas)` → [View.java:25251](vscode-webview://0ehv9b7aq8bm0bkh8crpv71gesmvhg8egl0k1jo51it32i693gcf/core/java/android/view/View.java#L25251)，执行经典的 7 步绘制：
    1. `drawBackground()` — 绘制背景
    2. 保存 layer（用于 fading edges）
    3. `onDraw()` — 绘制自身内容
    4. `dispatchDraw()` — 绘制子 View
    5. 恢复 layer / 绘制 fading edges
    6. `onDrawForeground()` — 前景、滚动条等
    7. `drawDefaultFocusHighlight()` — 焦点高亮
- `renderNode.endRecording()` 封存录制

`RecordingCanvas` 不会真正画像素，而是把绘制命令（drawRect, drawText, drawRenderNode 等）记录为 GPU 可重放的指令流。

**b) 将根 View 的 RenderNode 挂到根 RenderNode 上**

```java
mRootNode.beginRecording(surfaceW, surfaceH);
canvas.drawRenderNode(view.updateDisplayListIfDirty());  // 将 View 的 RenderNode 链入
mRootNode.endRecording();
```

这里的 `drawRenderNode()` → [RecordingCanvas.java:179](vscode-webview://0ehv9b7aq8bm0bkh8crpv71gesmvhg8egl0k1jo51it32i693gcf/graphics/java/android/graphics/RecordingCanvas.java#L179) 只是记录一条「引用子 RenderNode」的指令，不实际绘制。

### 3. 渲染阶段：GPU 光栅化

`syncAndDrawFrame()` → [HardwareRenderer.java:554](vscode-webview://0ehv9b7aq8bm0bkh8crpv71gesmvhg8egl0k1jo51it32i693gcf/graphics/java/android/graphics/HardwareRenderer.java#L554) 是 native JNI 调用，将 RenderNode 树从 UI 线程交给 Render 线程：

```
nSyncAndDrawFrame (native)
  ├── 同步 RenderNode 树到 Render 线程
  ├── Render 线程重放 DisplayList 指令到 OpenGL/Skia GPU 上下文
  └── 交换 buffer 到 Surface（显示到屏幕）
```

### 总结：两条流水线

```
UI 线程 (录制)                      Render 线程 (渲染)
─────────────────                   ─────────────────
View.draw(RecordingCanvas)          
  → 记录绘制命令到 RenderNode
                                    syncAndDrawFrame()
                                      → GPU 重放指令
                                      → 光栅化像素
                                      → buffer swap 到屏幕
```

核心设计思想：**UI 线程只录制指令（DisplayList），不碰 GPU；Render 线程离线重放指令做真正的光栅化**。这样 UI 线程不会阻塞在 GPU 上，也能利用 Render 线程做并行渲染。

`RecordingCanvas` 不会真正画像素，而是把绘制命令（drawRect, drawText, drawRenderNode 等）记录为 GPU 可重放的指令流。

### 面试
1. Launcher判断是否需要处理 ActivityResult 后，获取ATMS的服务，启动Activity；
2. ATMS解析Intent参数，进行权限校验， 通过后，创建 ActivityRecord和 Task 信息加入到根节点；
3. 创建 Pause事务暂停 Launcher，同时通过socket请求zygote创建应用进程；
4. zygote fork出子进程后，通过反射创建ActivityThread对象，同时执行其入口main方法；
5. 在main方法中，启动应用的Binder服务ApplicationThread ，并返回给AMS，同时启动主线程Looper，开始消息循环；形成AMS -> ApplicationThread ->Handler通信链；
6. AMS会调用ApplicationThread的bindApplication方法，向主线程中发送bindApplication消息，启动Application；
7. 对于Activity的相关生命周期方法，则封装成对应事务后，统一发送EXECUTE_TRANSACTION消息进行处理;	
8. 当执行到 onResume 时，会调用WindowManager.addView方法，将 DecorView添加到 WindowManager中。触发 View 的测量、布局、绘制流程，此时 Activity 才对用户可见；
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
## 一帧数据的绘制
1. 应用的`View.invalidate()`、动画、数据变化或输入事件触发会调用到ViewRootImpl.scheduleTraversals，
2. 应用接受到app类型的V-sync信号，唤醒等待中的UI线程，`Choroegrapher`回调`onVsync`开始一帧的绘制，依次处理Input事件，animation动画，performTraversals，其中traversal包含View的测量， 布局和绘制；
3. 绘制完成后，更新绘制数据，结束一帧的绘制，继续处理下一帧的Message，同时通过postAndWait唤醒渲染线程执行界面渲染任务。
4. 渲染线程先同步UI线程构建好的绘制命令树，然后通过dequeueBuffer申请一张处于free状态的buffer，进行GPU渲染，渲染完成后swipBuffer触发queueBuffer动作上帧；
5. 渲染线程通过queueBuffer唤醒对端的SurfaceFlinger进程中的Binder工作线程，申请sf类型的vsync信号；
6. sf类型的VSYNC信号到达后后，sf开始执行一帧的合成任务，之后再执行present唤醒HWC service进程执行图层合成送显；

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