# Android Shell Transition 深度分析

> Android 手势导航（Gesture Navigation）中的 Shell Transition 机制全解析  
> 源码路径基于 `E:\sourcecode\frameworks\base`

---

## 目录

1. [Shell Transition 框架概述](#1-shell-transition-框架概述)
2. [上滑进入多任务（Recents）](#2-上滑进入多任务recents)
3. [底部导航栏水平滑动快速切换（QuickSwitch）](#3-底部导航栏水平滑动快速切换quickswitch)
4. [Launcher 如何在三方 App 页面接收触摸事件](#4-launcher-如何在三方-app-页面接收触摸事件)
5. [GestureMonitorSpyWindow — 间谍窗口](#5-gesturemonitorspywindow--间谍窗口)
6. [recents_animation_input_consumer — 独占消费窗口](#6-recents_animation_input_consumer--独占消费窗口)
7. [InputChannel — 触摸事件如何从内核传到 Launcher 进程](#7-inputchannel--触摸事件如何从内核传到-launcher-进程)
8. [附录：源码文件索引](#附录源码文件索引)

---

## 1. Shell Transition 框架概述

### 1.1 架构分层

Android Shell Transition 采用**两阶段架构**：

| 层级 | 进程 | 职责 |
|------|------|------|
| **WMCore** (服务端) | `system_server` | 记录/收集 WindowManager 变更，同步参与动画的 WindowContainer |
| **Shell** (客户端) | `com.android.systemui` | 播放动画，管理动画生命周期 |

### 1.2 生命周期状态机

```
--start--> PENDING --onTransitionReady--> READY --play--> ACTIVE --finish--> |
                                                    --merge--> MERGED --^
```

### 1.3 核心 Transition 类型

```java
// Transitions.java
TRANSIT_START_RECENTS_TRANSITION  = TRANSIT_FIRST_CUSTOM + 21  // 开始 Recents
TRANSIT_END_RECENTS_TRANSITION    = TRANSIT_FIRST_CUSTOM + 22  // 结束 Recents (Bookend)
```

### 1.4 关键文件

| 文件 | 作用 |
|------|------|
| [Transition.java](services/core/java/com/android/server/wm/Transition.java) | WMCore Transition 实体 |
| [TransitionController.java](services/core/java/com/android/server/wm/TransitionController.java) | WMCore 生命周期控制器 |
| [Transitions.java](libs/WindowManager/Shell/src/com/android/wm/shell/transition/Transitions.java) | Shell 端播放器引擎 |
| [RecentsTransitionHandler.java](libs/WindowManager/Shell/src/com/android/wm/shell/recents/RecentsTransitionHandler.java) | Recents/QuickSwitch 动画 Handler |
| [RecentsMixedTransition.java](libs/WindowManager/Shell/src/com/android/wm/shell/transition/RecentsMixedTransition.java) | Recents + 分屏/桌面/锁屏 混合过渡 |
| [WindowOrganizerController.java](services/core/java/com/android/server/wm/WindowOrganizerController.java) | Shell 调用 WMCore 的入口 |

---

## 2. 上滑进入多任务（Recents）

### 2.1 流程图

```
用户在底部导航栏上滑
  ↓
Launcher3 (QuickStep) 检测到上滑手势
  ↓
IRecentTasks.startRecentsTransition(intent, options, animRunner)
  ↓
┌─ frameworks/base 侧开始 ────────────────────────────────────────┐
│ [1] RecentTasksController.startRecentsTransition()               │
│     → libs/WindowManager/Shell/src/.../recents/RecentTasksController.java:1083
│                                                                  │
│ [2] RecentsTransitionHandler.startRecentsTransition()            │
│     → libs/WindowManager/Shell/src/.../recents/RecentsTransitionHandler.java:171
│       ├─ 构造 WindowContainerTransaction                          │
│       ├─ transitionType = TRANSIT_START_RECENTS_TRANSITION       │
│       └─ mTransitions.startTransition(type, wct, handler)        │
│                                                                  │
│ [3] Transitions.startTransition()                                │
│     → libs/WindowManager/Shell/src/.../transition/Transitions.java:1294
│       └─ mOrganizer.startNewTransition(type, wct) → WMCore       │
│                                                                  │
│ [4] WMCore TransitionController.requestStartTransition()         │
│     → services/core/.../wm/TransitionController.java:805         │
│       └─ 通知 Shell: mPlayer.requestStartTransition(token, req)   │
│                                                                  │
│ [5] WMCore 采集完成 → Transition.onTransitionReady()             │
│     → services/core/.../wm/Transition.java:1990                  │
│       └─ mController.getTransitionPlayer().onTransitionReady()   │
│                                                                  │
│ [6] Transitions.onTransitionReady() → dispatchReady()            │
│     → libs/WindowManager/Shell/src/.../transition/Transitions.java:709
│       └─ processReadyQueue() → playTransition()                  │
│                                                                  │
│ [7] playTransition() → handler.startAnimation()                  │
│     → libs/WindowManager/Shell/src/.../transition/Transitions.java:1016
│                                                                  │
│ [8] RecentsTransitionHandler.startAnimation()                    │
│     → libs/WindowManager/Shell/src/.../recents/RecentsTransitionHandler.java:291
│       └─ controller.start(info, startT, finishT, finishCB)       │
│                                                                  │
│ [9] RecentsController.start()                                    │
│     → libs/WindowManager/Shell/src/.../recents/RecentsTransitionHandler.java:664
│       ├─ 构建 RemoteAnimationTarget[]                             │
│       ├─ 三层 Z-order: below / middle / above                     │
│       └─ mListener.onAnimationStart(...) → 通知 Launcher         │
│                                                                  │
│ [10] Launcher3 播放 Overview 动画                                 │
│                                                                  │
│ [11] 用户点击 App 卡片 → IRecentsAnimationController.finish()    │
│                                                                  │
│ [12] RecentsController.finishInner()                             │
│      → libs/WindowManager/Shell/src/.../recents/RecentsTransitionHandler.java:1337
│        ├─ 启动 Bookend: TRANSIT_END_RECENTS_TRANSITION            │
│        └─ finishCallback.onTransitionFinished() → WMCore          │
│                                                                  │
│ [13] Transition.finishTransition()                               │
│      → services/core/.../wm/Transition.java:1280                 │
│        └─ 提交可见性变更，清理资源                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 交互图

```
用户 ──上滑──→ TouchInteractionService ──→ RecentTasks.startRecentsTransition()
                                                  │
               ┌──────────────────────────────────┘
               ▼
     [Shell] RecentsTransitionHandler
        │  TRANSIT_START_RECENTS_TRANSITION
        ▼
     [WMCore] TransitionController → 采集 WindowContainer → onTransitionReady
        │
        ▼
     [Shell] playTransition → RecentsController.start()
        │  RemoteAnimationTarget[] = { pausing app, recents, wallpaper }
        ▼
     [Launcher3] RecentsAnimationRunner.onAnimationStart()
        │  播放 Overview 动画
        ▼
     用户选择: 点击卡片 / 上滑回桌面 / 松手返回
        │
        ▼
     RecentsController.finishInner()
        │  TRANSIT_END_RECENTS_TRANSITION (Bookend)
        ▼
     Transition.finishTransition() → 清理 InputConsumer, 提交可见性
```

---

## 3. 底部导航栏水平滑动快速切换（QuickSwitch）

### 3.1 与上滑场景的对比

QuickSwitch **与上滑进入 Recents 共用完全相同的 Shell Transition 管线**。

| 维度 | 上滑进入 Recents | 水平滑动 QuickSwitch |
|------|-----------------|---------------------|
| Transition 类型 | `TRANSIT_START_RECENTS_TRANSITION` | `TRANSIT_START_RECENTS_TRANSITION`（相同） |
| Shell Handler | `RecentsTransitionHandler` | `RecentsTransitionHandler`（相同） |
| Launcher 动画 | 展开到 Overview 网格 | 立即切换到相邻 Task |
| 内部状态 | `STATE_NORMAL` | `STATE_NEW_TASK`（切换后） |
| 松手回退 | `finish(toHome=true)` 回到桌面 | `returningToApp=true` → reorder 回原 App |
| 结束方式 | `TRANSIT_END_RECENTS_TRANSITION` bookend | `TRANSIT_END_RECENTS_TRANSITION` bookend |

### 3.2 QuickSwitch 关键分支

```java
// RecentsTransitionHandler.java:1337
private void finishInner(boolean toHome, ...) {
    // 情况 B2: 滑动距离不够 → 松手回到当前 App
    boolean returningToApp = !toHome
            && !mWillFinishToHome
            && mPausingTasks != null
            && mState == STATE_NORMAL;  // ← 只有 QuickSwitch 取消时才是 STATE_NORMAL

    if (returningToApp) {
        for (int i = mPausingTasks.size() - 1; i >= 0; --i) {
            wct.reorder(mPausingTasks.get(i).mToken, true);  // 把原 App 放回顶层
            t.show(mPausingTasks.get(i).mTaskSurface);       // 重新显示
        }
    }
}
```

---

## 4. Launcher 如何在三方 App 页面接收触摸事件

### 4.1 三层机制

```
┌──────────────────────────────────────────────────────────────────┐
│                      触摸分发体系                                  │
│                                                                   │
│  [触摸硬件] → [InputDispatcher (system_server)]                    │
│                     │                                             │
│       ┌─────────────┼─────────────┐                               │
│       │             │             │                               │
│  ┌────┴────┐  ┌─────┴─────┐  ┌───┴───┐                            │
│  │ Spy     │  │ Consumer  │  │App 窗口│                            │
│  │ Window  │  │ (动画时)   │  │       │                            │
│  │(持久)   │  │           │  │       │                            │
│  │SPY|NOT_ │  │~NOT_FOCUS │  │       │                            │
│  │FOCUSABLE│  │           │  │       │                            │
│  └────┬────┘  └─────┬─────┘  └───┬───┘                            │
│       │             │             │                                │
│  Launcher 进程    Launcher 进程  App 进程                          │
│  (暗中监听)       (独占交互)     (正常接收)                           │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 平时状态（Recents 动画未运行）

```
SystemUI NavigationBar 窗口（TYPE_NAVIGATION_BAR，Z-order > App）
  → EdgeBackGestureHandler 识别手势
  → LauncherProxyService 通过 Binder IPC 通知 Launcher3
  → Launcher3 调用 IRecentTasks.startRecentsTransition()
```

### 4.3 动画状态（Recents 动画运行中）

```
INPUT_CONSUMER_RECENTS_ANIMATION 接管全屏触摸
  → touchableRegion = App 整个屏幕区域
  → App 完全收不到事件
  → Launcher3 通过 InputConsumerController 独占触摸
```

---

## 5. GestureMonitorSpyWindow — 间谍窗口

### 5.1 作用

在指定 Display 上创建一个**隐藏的间谍窗口**，以最低权限偷听所有触摸事件。

### 5.2 创建流程

```
TouchInteractionService (Launcher3)
  │
  ├─ new InputMonitorCompat("swipe-up", DEFAULT_DISPLAY)
  │
  └─→ [InputMonitorCompat.java]  ← frameworks/base
      packages/SystemUI/shared/src/.../system/InputMonitorCompat.java:42
        │
        └─→ InputManagerGlobal.monitorGestureInput(name, displayId)
              │
              └─→ [InputManagerService.java]
                  services/core/java/com/android/server/input/InputManagerService.java:864
                    │
                    ├─ 检查权限: MONITOR_INPUT
                    ├─ 创建 SurfaceControl (透明，无图形缓冲)
                    └─ GestureMonitorSpyWindow 构造
```

### 5.3 GestureMonitorSpyWindow 关键属性

```java
// GestureMonitorSpyWindow.java:38
class GestureMonitorSpyWindow {
    mWindowHandle.layoutParamsType = TYPE_SECURE_SYSTEM_OVERLAY;
    mWindowHandle.inputConfig = InputConfig.NOT_FOCUSABLE | InputConfig.SPY;
    //                           ↑ 不可获焦                  ↑ 偷听模式
    // 放在 INPUT_OVERLAY_LAYER_GESTURE_MONITOR 层 (高 Z-order)
}
```

| 标志 | 含义 |
|------|------|
| `SPY` | **偷听模式** — 接收所有触摸事件的**副本**，不阻止原目标窗口收到事件 |
| `NOT_FOCUSABLE` | **不可获焦** — 永远不能获得输入焦点，只能旁观 |

### 5.4 两步手势接管

```
阶段 1: SPY (监视)
  GestureMonitorSpyWindow → 接收事件副本 → Launcher 分析手势
  三方 App 仍正常收到事件 ✅

阶段 2: STEAL (抢夺)
  Launcher 识别到上滑手势 → pilferPointers()
  → InputDispatcher 取消派发给 App 的事件 → App 收到 ACTION_CANCEL
  后续事件全部发给 Launcher
```

---

## 6. recents_animation_input_consumer — 独占消费窗口

### 6.1 与 Spy Window 的本质区别

| 维度 | GestureMonitorSpyWindow | recents_animation_input_consumer |
|------|------------------------|----------------------------------|
| 定义常量 | 无 | `WindowManager.INPUT_CONSUMER_RECENTS_ANIMATION` |
| inputConfig | `SPY \| NOT_FOCUSABLE` | `~NOT_FOCUSABLE`（**清除不可获焦** = 可抢占焦点） |
| 行为 | **偷听** — App 继续收到事件 | **独占** — App 完全被屏蔽 |
| 触摸区域 | 整个 Display | 精确设置为 pausing task 的 bounds |
| 生命周期 | 持久存在 | 动画期间动态创建/销毁 |

### 6.2 生命周期

```
┌─ 创建 ────────────────────────────────────────────────────────────┐
│ Launcher3 检测到手势 → 启动 Recents Transition                    │
│   ↓                                                               │
│ InputConsumerController.registerInputConsumer()                   │
│   → mWindowManager.createInputConsumer(                           │
│         token, "recents_animation_input_consumer",                │
│         DEFAULT_DISPLAY, inputChannel)                            │
│   ↓                                                               │
│ InputMonitor.createInputConsumer()                                │
│   → consumer.mWindowHandle.inputConfig &= ~NOT_FOCUSABLE          │
│   → addInputConsumer(consumer)                                    │
└───────────────────────────────────────────────────────────────────┘

┌─ 激活 ────────────────────────────────────────────────────────────┐
│ Transition.onTransitionReady() → handleLegacyRecentsStartBehavior │
│   → recentsAnimationInputConsumer.touchableRegion = App 全屏区域    │
│   → dc.getInputMonitor().setActiveRecents(recentsTask, topApp)    │
│   → consumer 放在 recentsTask 之上，获得 input focus                │
│                                                                    │
│ 效果: 全屏触摸 → consumer 独占 → Launcher 处理                     │
└───────────────────────────────────────────────────────────────────┘

┌─ 销毁 ────────────────────────────────────────────────────────────┐
│ Recents 动画结束                                                   │
│   → dc.getInputMonitor().setActiveRecents(null, null)             │
│   → mWindowManager.destroyInputConsumer(token, displayId)         │
│   → InputChannel dispose                                          │
└───────────────────────────────────────────────────────────────────┘
```

### 6.3 源码文件

| 文件 | 关键代码 |
|------|---------|
| [InputConsumerImpl.java](services/core/java/com/android/server/wm/InputConsumerImpl.java) | **核心** — 创建 InputChannel 对，`copyTo()` 传 FD 给 Launcher |
| ↳ 构造 (line 53) | `createInputChannel(name)` + `copyTo(inputChannel)` |
| [InputMonitor.java](services/core/java/com/android/server/wm/InputMonitor.java) | 管理 consumer 集合 |
| ↳ `createInputConsumer()` (line 221) | `~NOT_FOCUSABLE` 使 consumer 可抢占焦点 |
| ↳ `setActiveRecents()` (line 394) | 激活 consumer，设置 touchableRegion |
| [Transition.java](services/core/java/com/android/server/wm/Transition.java) | 控制 consumer 生命周期 |
| ↳ `handleLegacyRecentsStartBehavior()` (line 2343) | Transition 开始时激活 |
| ↳ `finishTransition()` (line 1545) | Transition 结束时清除 |
| [InputConsumerController.java](packages/SystemUI/shared/src/com/android/systemui/shared/system/InputConsumerController.java) | Launcher 侧创建/销毁 consumer |
| ↳ `getRecentsAnimationInputConsumer()` (line 104) | 工厂方法 |
| ↳ `registerInputConsumer()` (line 138) | 创建 + 监听 InputChannel |

---

## 7. InputChannel — 触摸事件如何从内核传到 Launcher 进程

### 7.1 核心机制

`InputChannel` 底层是一对通过 `socketpair()` 创建的 **Unix Domain Socket (SOCK_SEQPACKET)**。

```
[system_server 进程]                      [Launcher 进程]

InputConsumerImpl 构造:                   InputConsumerController:
  mClientChannel (FD-A)                    inputChannel (初始为空)
  │                                         │
  └── mClientChannel.copyTo(inputChannel) ──┘
       │                          
       │  Binder 将 FD 跨进程传递
       │
  FD-A ←──────── socketpair ────────→ FD-B
 (server)                             (client)
    │                                    │
    │ 注册到 InputDispatcher              │ 注册到 Looper epoll
    │                                    │
  InputDispatcher                        BatchedInputEventReceiver
  将 MotionEvent                         在 VSYNC 回调中
  写入 FD-A                              从 FD-B 读取
```

### 7.2 完整流程

```
步骤 1: Launcher 请求创建 Consumer
  InputConsumerController.registerInputConsumer()
    → WMS.createInputConsumer(token, name, displayId, inputChannel)

步骤 2: system_server 创建 Channel 对
  InputConsumerImpl 构造 (InputConsumerImpl.java:53)
    → mService.mInputManager.createInputChannel(name)
      → JNI → InputDispatcher::createInputChannel()
        → socketpair(AF_UNIX, SOCK_SEQPACKET, ...)
          创建一对互联的 FD
    → mClientChannel.copyTo(inputChannel)
      通过 Binder out 参数将 client 端 FD 传给 Launcher

步骤 3: Launcher 监听 FD
  new InputEventReceiver(inputChannel, looper, choreographer)
    → BatchedInputEventReceiver 将 FD 注册到 Looper epoll
    → 在 VSYNC 回调中读取事件

步骤 4: InputDispatcher 派发事件
  Consumer 获得焦点 (NOT_FOCUSABLE 被清除)
    → InputDispatcher 将 MotionEvent 写入 server 端 FD-A
    → 内核将数据传送到 socket pair 另一端的 FD-B
    → Launcher 进程的 Looper 被唤醒
    → Choreographer VSYNC 回调
    → onInputEvent(event) → 业务逻辑处理
```

### 7.3 两类通道对比

| 维度 | Spy Window 的 InputChannel | Consumer 的 InputChannel |
|------|--------------------------|-------------------------|
| 创建方式 | `InputManagerService.createSpyWindowGestureMonitor()` | `InputConsumerImpl` 构造中 `createInputChannel()` |
| 底层机制 | `socketpair()` | `socketpair()`（相同） |
| 注册位置 | InputDispatcher 的 spy window list | InputDispatcher 的 input window list |
| 事件类型 | 副本（SPY） | 独占（~NOT_FOCUSABLE） |
| Launcher 侧接收 | `InputMonitorCompat.getInputReceiver()` | `InputConsumerController` 的 `InputEventReceiver` |

---

## 附录：源码文件索引

### Shell Transition 核心管线

| 文件 | 绝对路径 |
|------|---------|
| Transition.java | `E:\sourcecode\frameworks\base\services\core\java\com\android\server\wm\Transition.java` |
| TransitionController.java | `E:\sourcecode\frameworks\base\services\core\java\com\android\server\wm\TransitionController.java` |
| Transitions.java | `E:\sourcecode\frameworks\base\libs\WindowManager\Shell\src\com\android\wm\shell\transition\Transitions.java` |
| RecentsTransitionHandler.java | `E:\sourcecode\frameworks\base\libs\WindowManager\Shell\src\com\android\wm\shell\recents\RecentsTransitionHandler.java` |
| RecentsMixedTransition.java | `E:\sourcecode\frameworks\base\libs\WindowManager\Shell\src\com\android\wm\shell\transition\RecentsMixedTransition.java` |
| DefaultMixedHandler.java | `E:\sourcecode\frameworks\base\libs\WindowManager\Shell\src\com\android\wm\shell\transition\DefaultMixedHandler.java` |
| DefaultTransitionHandler.java | `E:\sourcecode\frameworks\base\libs\WindowManager\Shell\src\com\android\wm\shell\transition\DefaultTransitionHandler.java` |
| RecentTasksController.java | `E:\sourcecode\frameworks\base\libs\WindowManager\Shell\src\com\android\wm\shell\recents\RecentTasksController.java` |
| HomeTransitionObserver.java | `E:\sourcecode\frameworks\base\libs\WindowManager\Shell\src\com\android\wm\shell\transition\HomeTransitionObserver.java` |
| WindowOrganizerController.java | `E:\sourcecode\frameworks\base\services\core\java\com\android\server\wm\WindowOrganizerController.java` |

### 输入机制

| 文件 | 绝对路径 |
|------|---------|
| InputMonitor.java | `E:\sourcecode\frameworks\base\services\core\java\com\android\server\wm\InputMonitor.java` |
| InputConsumerImpl.java | `E:\sourcecode\frameworks\base\services\core\java\com\android\server\wm\InputConsumerImpl.java` |
| GestureMonitorSpyWindow.java | `E:\sourcecode\frameworks\base\services\core\java\com\android\server\input\GestureMonitorSpyWindow.java` |
| InputManagerService.java | `E:\sourcecode\frameworks\base\services\core\java\com\android\server\input\InputManagerService.java` |
| InputMonitorCompat.java | `E:\sourcecode\frameworks\base\packages\SystemUI\shared\src\com\android\systemui\shared\system\InputMonitorCompat.java` |
| InputConsumerController.java | `E:\sourcecode\frameworks\base\packages\SystemUI\shared\src\com\android\systemui\shared\system\InputConsumerController.java` |
| InputChannelCompat.java | `E:\sourcecode\frameworks\base\packages\SystemUI\shared\src\com\android\systemui\shared\system\InputChannelCompat.java` |

### SystemUI 通信层

| 文件 | 绝对路径 |
|------|---------|
| LauncherProxyService.java | `E:\sourcecode\frameworks\base\packages\SystemUI\src\com\android\systemui\recents\LauncherProxyService.java` |
| ISystemUiProxy.aidl | `E:\sourcecode\frameworks\base\packages\SystemUI\shared\src\com\android\systemui\shared\recents\ISystemUiProxy.aidl` |
| ILauncherProxy.aidl | `E:\sourcecode\frameworks\base\packages\SystemUI\shared\src\com\android\systemui\shared\recents\ILauncherProxy.aidl` |
| IRecentTasks.aidl | `E:\sourcecode\frameworks\base\libs\WindowManager\Shell\src\com\android\wm\shell\recents\IRecentTasks.aidl` |

### 回退导航

| 文件 | 绝对路径 |
|------|---------|
| BackNavigationController.java | `E:\sourcecode\frameworks\base\services\core\java\com\android\server\wm\BackNavigationController.java` |
| BackAnimationController.java | `E:\sourcecode\frameworks\base\libs\WindowManager\Shell\src\com\android\wm\shell\back\BackAnimationController.java` |

### 触摸事件流（Launcher3 侧，不在本仓库）

| 组件 | 说明 |
|------|------|
| `TouchInteractionService` | 入口，创建 Spy Monitor，监听手势 |
| `RecentsAnimationRunner` | 接收 Consumer 事件，驱动 Overview 动画 |
| `InputConsumerController` | 创建 Consumer + InputEventReceiver |

---

> 文档生成时间：2026-06-16  
> 源码路径：`E:\sourcecode\frameworks\base`
