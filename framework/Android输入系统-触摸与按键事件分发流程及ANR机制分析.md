# Android 输入系统：触摸与按键事件分发流程及 ANR 机制分析

> 基于 AOSP `frameworks/base` 和 `frameworks/native` 源码分析  
> 仓库路径: `e:\sourcecode\frameworks\`

---

## 目录

1. [架构概览](#1-架构概览)
2. [触摸/按键事件从 InputFlinger 到应用进程的完整流程](#2-事件分发完整流程)
3. [InputDispatcher 查找目标窗口 (focusedWindow)](#3-inputdispatcher-查找-focuswindow)
4. [InputDispatcher 查找触摸窗口 (touchedWindows)](#4-inputdispatcher-查找-touchwindows)
5. [SurfaceFlinger 事务通知 InputFlinger 机制](#5-surfaceflinger-事务通知-inputflinger)
6. [无焦点窗口 ANR 触发机制](#6-无焦点窗口-anr-触发机制)
7. [有窗口不响应 ANR 触发机制](#7-有窗口不响应-anr-触发机制)
8. [事件传递到 ViewRootImpl 的流程](#8-事件传递到-viewrootimpl)
9. [InputChannel 的分配机制](#9-inputchannel-的分配机制)
10. [批量输入事件与 Choreographer CALLBACK_INPUT](#10-批量输入事件与-choreographer-callback_input)
11. [WindowInputEventReceiver.onInputEvent 处理的事件类型](#11-oninputevent-处理的事件类型)
12. [关键源码文件索引](#12-关键源码文件索引)

---

## 1. 架构概览

```
┌──────────────────────────────────────────────────────────────────────┐
│  Linux Kernel (evdev)  —  /dev/input/eventX                          │
└──────────────────────────┬───────────────────────────────────────────┘
                           │ raw input_event
                           ▼
┌──────────────────────────────────────────────────────────────────────┐
│  InputFlinger (native C++) — frameworks/native                       │
│  ┌──────────────┐    ┌───────────────────┐    ┌─────────────────┐   │
│  │ InputReader  │───▶│ InputDispatcher   │    │ PointerChoreo..│   │
│  │  (EventHub)  │    │  - dispatchOnce()  │    │  (光标/指针)    │   │
│  │  (Mappers)   │    │  - findTargets()   │    └─────────────────┘   │
│  └──────────────┘    │  - processAnrs()   │                          │
│                       └─────────┬─────────┘                          │
└─────────────────────────────────┼────────────────────────────────────┘
                                  │ InputChannel (Unix socket pair)
                                  │
┌─────────────────────────────────┼────────────────────────────────────┐
│  Java Framework (system_server) │  — frameworks/base                 │
│  ┌──────────────────────────┐   │                                    │
│  │ InputManagerService      │◀──┤ JNI 回调 (ANR / Focus / ...)       │
│  │ InputManagerCallback     │   │                                    │
│  │ AnrController            │   │                                    │
│  └──────────────────────────┘   │                                    │
└─────────────────────────────────┼────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│  应用进程 (App Process)                                              │
│  ┌─────────────────────┐    ┌──────────────────┐                    │
│  │ InputEventReceiver  │───▶│ ViewRootImpl     │──▶ DecorView → View│
│  │ WindowInputEventRecv│    │ (InputStage链)   │                    │
│  └─────────────────────┘    └──────────────────┘                    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. 事件分发完整流程

### 2.1 InputReader 阶段

| 步骤 | 代码位置 | 说明 |
|------|----------|------|
| 1. EventHub 读取 | `InputReader` (native) | epoll 监听 `/dev/input/eventX`，读取原始 `input_event` |
| 2. Mapper 加工 | `InputReader` (native) | `TouchInputMapper` → `MotionEvent`，`KeyboardInputMapper` → `KeyEvent` |
| 3. Policy 拦截（入队前） | [`com_android_server_input_InputManagerService.cpp:1859`](services/core/jni/com_android_server_input_InputManagerService.cpp#L1859) | `interceptKeyBeforeQueueing()` / `interceptMotionBeforeQueueing()` |

### 2.2 InputDispatcher 阶段

| 步骤 | 代码位置 | 说明 |
|------|----------|------|
| 4. 事件入队 | `InputDispatcher` (native) | 事件放入 `mInboundQueue` |
| 5. 查找目标窗口 | [InputDispatcher.cpp:2348](E:/sourcecode/frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp#L2348) | 触摸：`findTouchedWindowTargets()`，按键：`findFocusedWindowTargetLocked()` |
| 6. Policy 拦截（分发前） | [cpp:1961](services/core/jni/com_android_server_input_InputManagerService.cpp#L1961) | 仅按键：`interceptKeyBeforeDispatching()` |
| 7. 发送到 InputChannel | `InputChannel` (native) | Unix domain socket `send()` |

### 2.3 应用进程接收阶段

| 步骤 | 代码位置 | 说明 |
|------|----------|------|
| 8. Native fd 监听 | `NativeInputEventReceiver` (native) | epoll 监听 client 端 socket fd |
| 9. dispatchInputEvent | [InputEventReceiver.java:292](core/java/android/view/InputEventReceiver.java#L292) | JNI → Java，调用 `onInputEvent()` |
| 10. 递送到控件树 | [ViewRootImpl.java:10682](core/java/android/view/ViewRootImpl.java#L10682) | `processRawInputEvent()` → InputStage 链 → DecorView |
| 11. finishInputEvent | [InputEventReceiver.java:213](core/java/android/view/InputEventReceiver.java#L213) | 通知 InputDispatcher 事件已处理 |

---

## 3. InputDispatcher 查找 focusedWindow

### 3.1 关键代码

- **函数入口**: [InputDispatcher.cpp:2348](E:/sourcecode/frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp#L2348) `findFocusedWindowTargetLocked()`
- **分派入口**: [InputDispatcher.cpp:1002](E:/sourcecode/frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp#L1002) `dispatchOnce()`

### 3.2 查找逻辑

```
findFocusedWindowTargetLocked(currentTime, entry, nextWakeupTime)

1. 获取 focusedWindowHandle = getFocusedWindowHandleLocked(displayId)    [L2351]
2. 获取 focusedApplicationHandle = mFocusedApplicationHandlesByDisplay   [L2352]

3. ★ 三个分支:
   ├─ focusedWindow == null && focusedApp == null
   │   → 丢弃事件，不触发 ANR                                        [L2357-2362]
   │
   ├─ focusedWindow == null && focusedApp != null
   │   ├─ 首次发现无焦点窗口: 启动计时器                             [L2376-2387]
   │   │   mNoFocusedWindowTimeoutTime = now + 5s
   │   │   return PENDING
   │   ├─ 已超时: return FAILED (事件丢弃)                           [L2388-2392]
   │   └─ 等待中: return PENDING                                     [L2393-2396]
   │
   └─ focusedWindow != null
       └─ resetNoFocusedWindowTimeoutLocked()                          [L2400]
       └─ 验证 focusable / NOT_FOCUSABLE / PAUSE_DISPATCHING
       └─ 按键需等待前序事件完成: shouldWaitToSendKeyLocked()          [L2426]
       └─ 返回 focusedWindowHandle (正常分发)
```

### 3.3 focusable 的判断条件

WMS 在 `populateInputWindowHandle()` 中设置:

```java
// InputMonitor.java:278-280
final boolean focusable = w.canReceiveKeys()
        && (mDisplayContent.hasOwnFocus() || mDisplayContent.isOnTop());
h.setFocusable(focusable);
```

```java
// WindowState.java:2821
public boolean canReceiveKeys(boolean fromUserTouch) {
    return isVisibleRequestedOrAdding()     // ① 可见
        && mViewVisibility == VISIBLE        // ② View 可见
        && !mRemoveOnExit                    // ③ 未被移除
        && !FLAG_NOT_FOCUSABLE               // ④ 没有 NOT_FOCUSABLE 标志
        && windowsAreFocusable()             // ⑤ Activity 允许焦点
        && !task.shouldIgnoreInput()         // ⑥ Task 不忽略输入
        && (fromUserTouch || isOnTop());     // ⑦ 不可信 display 需触摸
}
```

---

## 4. InputDispatcher 查找 touchedWindows

### 4.1 关键代码

- **主函数**: [InputDispatcher.cpp:2436](E:/sourcecode/frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp#L2436) `findTouchedWindowTargets()`
- **坐标解析**: [cpp:671](E:/sourcecode/frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp#L671) `resolveTouchedPosition()`
- **命中测试**: [cpp:582](E:/sourcecode/frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp#L582) `windowAcceptsTouchAt()`
- **前景查找**: [cpp:1450](E:/sourcecode/frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp#L1450) `findTouchedWindowAt()`
- **Spy 窗口**: [cpp:1498](E:/sourcecode/frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp#L1498) `findTouchedSpyWindowsAt()`
- **外部触摸**: [cpp:1469](E:/sourcecode/frameworks/native/services/inputflinger/dispatcher/InputDispatcher.cpp#L1469) `findOutsideTargets()`

### 4.2 命中测试条件

```cpp
// InputDispatcher.cpp:582-607
bool windowAcceptsTouchAt(windowInfo, displayId, x, y, isStylus, displayTransform) {
    // ① displayId 匹配
    if (windowInfo.displayId != displayId) return false;
    // ② 非 NOT_VISIBLE
    if (inputConfig.test(NOT_VISIBLE)) return false;
    // ③ 非 NOT_TOUCHABLE（触控笔例外）
    if (inputConfig.test(NOT_TOUCHABLE) && !windowCanInterceptTouch) return false;
    // ④ 坐标在 touchableRegion 内
    if (!touchableRegion.contains(x, y)) return false;
    return true;
}
```

### 4.3 查找流程

```
findTouchedWindowTargets()                                    [L2436]
│
├─ 新手势 (DOWN / SCROLL / HOVER_ENTER / HOVER_MOVE):      [L2489]
│   ├─ resolveTouchedPosition(entry)                         [L671]
│   ├─ findTouchedWindowAt(x, y)                             [L1450]
│   │   └─ Z-order 从高到低遍历，返回第一个命中非 spy 窗口
│   ├─ (DOWN) findOutsideTargets() → WATCH_OUTSIDE_TOUCH     [L1469]
│   ├─ findTouchedSpyWindowsAt() → 收集命中 spy 窗口         [L1498]
│   ├─ 前景窗口 + spy 窗口 → newTouchedWindows
│   ├─ getTargetFlags() → FOREGROUND / SPLIT / OBSCURED      [L7486]
│   ├─ DUPLICATE_TOUCH_TO_WALLPAPER → 加入壁纸窗口
│   └─ tempTouchState.addOrUpdateWindow()
│
├─ 已有手势 (MOVE / UP / CANCEL):                           [L2622]
│   ├─ SLIPPERY + 单指 MOVE → 滑动穿透检查                   [L2650]
│   └─ 非 split POINTER_DOWN → 所有现有窗口接收新指针         [L2711]
│
└─ 输出: for (touchedWindow : tempTouchState.windows)        [L2786]
          addPointerWindowTarget(dispatchMode, targetFlags, ...)
```

### 4.4 最终判断条件汇总

| # | 条件 | Native 字段 | WMS 来源 |
|---|------|------------|----------|
| 1 | displayId 匹配 | `windowInfo.displayId` | 窗口所在 Display |
| 2 | 非 NOT_VISIBLE | `inputConfig` | Surface 可见性 |
| 3 | 非 NOT_TOUCHABLE | `inputConfig` | `FLAG_NOT_TOUCHABLE` |
| 4 | 非 Spy | `info.isSpy()` | `INPUT_FEATURE_SPY` |
| 5 | 坐标在区域内 | `touchableRegion` | `w.getSurfaceTouchableRegion()` |
| 6 | Z-order 最上层 | 遍历顺序 | WMS Z-order |
| 7 | 非 PAUSE_DISPATCHING | `inputConfig` | `w.mActivityRecord.paused` |
| 8 | WATCH_OUTSIDE_TOUCH | 额外接收 OUTSIDE | `FLAG_WATCH_OUTSIDE_TOUCH` |
| 9 | DUPLICATE_TOUCH | 壁纸跟随 | `hasWallpaper` |
| 10 | SLIPPERY | 滑动穿透 | `FLAG_SLIPPERY` |

---

## 5. SurfaceFlinger 事务通知 InputFlinger

### 5.1 通信路径

```
WMS (Java)                    SurfaceFlinger (Native)       InputFlinger (Native)
    │                                │                           │
    │ populateInputWindowHandle()    │                           │
    │ t.setInputWindowInfo(sc,h)     │                           │
    │ merge to PendingTransaction    │                           │
    │ scheduleAnimation()            │                           │
    │                                │                           │
    │══ VSYNC → Binder IPC ════════▶│                           │
    │                                │ 处理事务                  │
    │                                │ 更新 Layer 元数据         │
    │                                │                           │
    │                                │ WindowInfosListener 回调  │
    │                                ├──────────────────────────▶│
    │                                │      InputDispatcher::    │
    │                                │      setInputWindows()    │
```

### 5.2 关键代码

- **写入 Transaction**: [InputMonitor.java:62](services/core/java/com/android/server/wm/InputWindowHandleWrapper.java#L62) `applyChangesToSurface()`
- **JNI**: [android_view_SurfaceControl.cpp:1093](core/jni/android_view_SurfaceControl.cpp#L1093) `nativeSetInputWindowInfo()`
- **通知回调**: [SurfaceControl.java:3495](core/java/android/view/SurfaceControl.java#L3495) `setInputWindowInfo()` / `addWindowInfosReportedListener()`
- **WMS 同步等待**: [WindowManagerService.java:8991](services/core/java/com/android/server/wm/WindowManagerService.java#L8991) `syncInputTransactions()`

### 5.3 关键点

- WMS **不直接**调用 InputFlinger，InputWindowHandle 搭载在 SF Transaction 中
- 变更检测优化：`InputWindowHandleWrapper` 追踪 `mChanged`，仅变化时才写入
- 延迟：最少 1 帧（~16ms @ 60Hz）
- SF 通过**进程内回调**通知 InputFlinger，不是 IPC

---

## 6. 无焦点窗口 ANR 触发机制

### 6.1 两种 ANR 的区别

| | `notifyNoFocusedWindowAnr` | `notifyWindowUnresponsive` |
|---|---|---|
| **含义** | 有焦点应用但无焦点窗口 | 有焦点窗口但不响应 |
| **携带参数** | `InputApplicationHandle`（应用级） | `IBinder token`（窗口级）+ pid |
| **TimeoutRecord** | `INPUT_DISPATCH_NO_FOCUSED_WINDOW` | `INPUT_DISPATCH_WINDOW_UNRESPONSIVE` |
| **计时起点** | `setFocusedApplication()` 调用时间 | 每个事件的 `dispatchTime` |

### 6.2 完整调用链

```
dispatchOnce()                                                   [L1002]
  └─ findFocusedWindowTargetLocked()                              [L2348]
     │  focusedWindow == null && focusedApp != null
     │  启动计时器 mNoFocusedWindowTimeoutTime                     [L2380]
     │
  └─ processAnrsLocked()                                          [L1074]
     │  now >= mNoFocusedWindowTimeoutTime
     └─ processNoFocusedWindowAnrLocked()                          [L1048]
        ├─ 确认焦点应用未变                                          [L1052]
        ├─ 确认焦点窗口仍为 null                                     [L1063]
        └─ onAnrLocked(mAwaitedFocusedApplication)                  [L1066]
           └─ [L6603-6612]
              reason = "xxx does not have a focused window"
              postCommandLocked([app] {
                  mPolicy.notifyNoFocusedWindowAnr(app);            [L6610]
              })
              │
              ▼ NativeInputManager::notifyNoFocusedWindowAnr()
              [com_android_server_input_InputManagerService.cpp:1223]
              │  env->CallVoidMethod(notifyNoFocusedWindowAnr, ...)
              │
              ▼ InputManagerService.notifyNoFocusedWindowAnr()
              [InputManagerService.java:2478]
              │
              ▼ InputManagerCallback.notifyNoFocusedWindowAnr()
              [InputManagerCallback.java:108]
              │  TimeoutRecord.forInputDispatchNoFocusedWindow()
              │  mAnrController.notifyAppUnresponsive(handle, record)
              │
              ▼ AnrController.notifyAppUnresponsive()
              [AnrController.java:68]
              ├─ preDumpIfLockTooSlow()
              ├─ ActivityRecord.forTokenLocked()
              ├─ blamePendingFocusRequest 判断
              ├─ activity.inputDispatchingTimedOut()
              │    → AMS → AnrHelper → ProcessErrorStateRecord
              │    → ANR 对话框 / 杀进程
              └─ dumpAnrStateAsync()
```

### 6.3 丢弃条件

AnrController 中会丢弃 ANR 请求的情况:

- `ActivityRecord == null` → "Unknown app ... Dropping notifyNoFocusedWindowAnr request"
- `activity.mAppStopped` → "App is in stopped state ... Dropping notifyNoFocusedWindowAnr request"
- Native 层二次确认时焦点应用已变化 → 不触发

---

## 7. 有窗口不响应 ANR 触发机制

### 7.1 调用链

```
InputDispatcher: event send() → dispatchTime = now()
  5s 内未收到 finishInputEvent
  → onAnrLocked(connection)                                      [L1109]
    └─ [L6568-6600]
       reason = "xxx is not responding. Waited Xms for MotionEvent..."
       processConnectionUnresponsiveLocked()
       │
       ▼ 通过 JNI → InputManagerService → InputManagerCallback
       InputManagerCallback.notifyWindowUnresponsive(token, pid, reason)
       [InputManagerCallback.java:115]
       │  TimeoutRecord.forInputDispatchWindowUnresponsive(message)
       │  mAnrController.notifyWindowUnresponsive(token, pid, record)
       │
       ▼ AnrController.notifyWindowUnresponsive()
       [AnrController.java:143]
       ├─ preDumpIfLockTooSlow()
       ├─ getInputTargetFromToken(token) → WindowState, pid
       ├─ activity.inputDispatchingTimedOut() 或 amInternal.inputDispatchingTimedOut(pid)
       └─ dumpAnrStateAsync(activity, windowState, reason)
```

### 7.2 超时消息构造

```java
// InputManagerCallback.java:408-423
private String timeoutMessage(OptionalInt pid, String reason) {
    String message = (reason == null)
        ? "Input dispatching timed out."
        : String.format("Input dispatching timed out (%s).", reason);

    // ★ GPU 挂起检测
    StalledTransactionInfo stalled = SurfaceControl.getStalledTransactionInfo(pid);
    if (stalled != null) {
        return message + " Buffer processing is stuck due to unsignaled fence"
               + " (window=" + stalled.layerName + "). Potential GPU hang.";
    }
    return message;
}
```

---

## 8. 事件传递到 ViewRootImpl

### 8.1 完整流程图

```
InputChannel.fd (socket)
    │  epoll 可读
    ▼
[Native] NativeInputEventReceiver::handleEvent()
    │  读取 InputMessage → 解码 → JNI upcall
    ▼
[Java] InputEventReceiver.dispatchInputEvent(seq, event)
    │  [InputEventReceiver.java:292]
    ▼
WindowInputEventReceiver.onInputEvent(event)
    │  [ViewRootImpl.java:10682]
    ▼
ViewRootImpl.processRawInputEvent(event)
    │  [ViewRootImpl.java:10904]
    ├─ 兼容性处理（mInputCompatProcessor）
    ▼
ViewRootImpl.enqueueInputEvent(event, receiver, flags, true)
    │  [ViewRootImpl.java:10445]
    ├─ QueuedInputEvent 包装 → 链表尾部
    ▼
ViewRootImpl.doProcessInputEvents()
    │  [ViewRootImpl.java:10496]
    ├─ 循环取 mPendingInputEventHead
    ▼
ViewRootImpl.deliverInputEvent(q)
    │  [ViewRootImpl.java:10523]
    ├─ 选择起始 InputStage
    ▼
┌─ InputStage 责任链 [ViewRootImpl.java:1797-1808] ────────┐
│                                                              │
│  ⓪ NativePreImeInputStage.onProcess(q)                      │
│  ① ViewPreImeInputStage.onProcess(q)    [L7792]             │
│       └─ 按键: mView.dispatchKeyEventPreIme()               │
│  ② ImeInputStage.onProcess(q)           [L7820]             │
│       └─ 按键: 发给输入法先处理                              │
│  ③ EarlyPostImeInputStage.onProcess(q)  [L7858]             │
│       ├─ 按键: Tooltip键, 退出触摸模式                       │
│       └─ 触摸: 坐标兼容, ensureTouchMode, 滚动偏移           │
│  ④ NativePostImeInputStage.onProcess(q) [L7966]             │
│  ⑤ ★ ViewPostImeInputStage.onProcess(q) [L7995]             │
│       ├─ 按键: mView.dispatchKeyEvent()     ← DecorView     │
│       └─ 触摸: mView.dispatchPointerEvent() ← DecorView     │
│  ⑥ SyntheticInputStage                                      │
└──────────────────────────────────────────────────────────────┘
                           ↓
ViewRootImpl.finishInputEvent(q)
    │  [ViewRootImpl.java:10570]
    └─ q.mReceiver.finishInputEvent(event, handled)
         └─ nativeFinishInputEvent() → socket send() → InputDispatcher
```

### 8.2 关键源码

- **事件入队**: [ViewRootImpl.java:10445](core/java/android/view/ViewRootImpl.java#L10445) `enqueueInputEvent()`
- **事件分发**: [ViewRootImpl.java:10523](core/java/android/view/ViewRootImpl.java#L10523) `deliverInputEvent()`
- **Stage 链构建**: [ViewRootImpl.java:1797](core/java/android/view/ViewRootImpl.java#L1797)
- **InputStage 基类**: [ViewRootImpl.java:7360](core/java/android/view/ViewRootImpl.java#L7360)
- **按键分发到 View**: [ViewRootImpl.java:8156](core/java/android/view/ViewRootImpl.java#L8156) `ViewPostImeInputStage.processKeyEvent()`
- **触摸分发到 View**: [ViewRootImpl.java:8228](core/java/android/view/ViewRootImpl.java#L8228) `ViewPostImeInputStage.processPointerEvent()`

---

## 9. InputChannel 的分配机制

**InputChannel 是按窗口分配的，不是按进程分配的。**

### 9.1 归属关系

```
一个应用进程 (pid=12345):
  MainActivity    → ViewRootImpl → InputChannel_A  (主窗口)
  Dialog          → ViewRootImpl → InputChannel_B  (Dialog 窗口)
  PopupWindow     → ViewRootImpl → InputChannel_C  (弹出窗口)
```

### 9.2 关键源码

- **InputChannel 创建**: [InputChannel.java:109](core/java/android/view/InputChannel.java#L109) `openInputChannelPair(name)`
- **无 InputChannel 的窗口**: `INPUT_FEATURE_NO_INPUT_CHANNEL` → 仅参与遮挡，不接收输入

### 9.3 应用级别 vs 窗口级别

| 级别 | 标识 | 用途 |
|------|------|------|
| 应用 (`InputApplicationHandle`) | ActivityRecord 的 IBinder token | ANR blame 对象 |
| 窗口 (`InputWindowHandle`) | InputChannel 的 IBinder token | 事件分发目标、触摸命中、焦点匹配 |

---

## 10. 批量输入事件与 Choreographer CALLBACK_INPUT

### 10.1 Choreographer 帧回调顺序

```java
// Choreographer.java:288,1101
CALLBACK_INPUT         = 0;  // ① 最先执行 — 输入事件消费
CALLBACK_ANIMATION     = 1;  // ② 动画
CALLBACK_INSETS_ANIMATION = 2; // ③ insets 动画
CALLBACK_TRAVERSAL     = 3;  // ④ measure/layout/draw
```

### 10.2 批量事件机制

```
InputDispatcher 积累多个 MOVE → batch pending
  → NativeInputEventReceiver.hasPendingBatch()
  → onBatchedInputEventPending(source)              [ViewRootImpl.java:10687]
    → scheduleConsumeBatchedInput()
      → mChoreographer.postCallback(CALLBACK_INPUT, mConsumedBatchedInputRunnable)
        → 下一帧 CALLBACK_INPUT:
          ConsumeBatchedInputRunnable.run()          [L10769]
            → doConsumeBatchedInput(frameTimeNanos)  [L10657]
              → nativeConsumeBatchedInputEvents()
              → doProcessInputEvents() → 处理所有 MOVE 事件
```

### 10.3 关键源码

- **scheduleConsumeBatchedInput**: [ViewRootImpl.java:10628](core/java/android/view/ViewRootImpl.java#L10628)
- **ConsumeBatchedInputRunnable**: [ViewRootImpl.java:10769](core/java/android/view/ViewRootImpl.java#L10769)
- **doConsumeBatchedInput**: [ViewRootImpl.java:10657](core/java/android/view/ViewRootImpl.java#L10657)
- **BatchedInputEventReceiver**: [BatchedInputEventReceiver.java](core/java/android/view/BatchedInputEventReceiver.java)

### 10.4 设计目的

1. **减少无用处理**: 两次 VSYNC 间的多个 MOVE 合并到帧开始一次性处理
2. **保证时序**: 输入在动画/绘制之前完成，本帧使用最新坐标
3. **降低功耗**: 减少不必要的 CPU 唤醒
4. **防止饿死**: 消费批次后重新调度到下一帧

---

## 11. onInputEvent 处理的事件类型

### 11.1 Native 层按类型分发

```cpp
// android_view_InputEventReceiver.cpp:390-485
switch (inputEvent->getType()) {
    case KEY:     → dispatchInputEvent(seq, event) → onInputEvent(KeyEvent)      ✅
    case MOTION:  → dispatchInputEvent(seq, event) → onInputEvent(MotionEvent)   ✅
    case FOCUS:   → onFocusEvent(hasFocus)            ❌ 专用回调
    case CAPTURE: → onPointerCaptureEvent(enabled)     ❌ 专用回调
    case DRAG:    → onDragEvent(isExiting, x, y, id)   ❌ 专用回调
    case TOUCH_MODE: → onTouchModeChanged(inTouchMode)  ❌ 专用回调
}
```

### 11.2 事件类型对照表

| 事件类别 | Native 类型 | Java 回调 | 是否走 onInputEvent |
|----------|------------|-----------|---------------------|
| 按键 | KEY | `onInputEvent(KeyEvent)` | ✅ |
| 触摸/运动 | MOTION | `onInputEvent(MotionEvent)` | ✅ |
| 焦点 | FOCUS | `onFocusEvent(hasFocus)` | ❌ |
| 指针捕获 | CAPTURE | `onPointerCaptureEvent(enabled)` | ❌ |
| 拖拽 | DRAG | `onDragEvent(isExiting, x, y, id)` | ❌ |
| 触摸模式 | TOUCH_MODE | `onTouchModeChanged(inTouchMode)` | ❌ |
| 批量 MOVE | MOTION (batch) | `onBatchedInputEventPending` → `consumeBatchedInputEvents` → `onInputEvent` | ✅ (延迟消费) |

### 11.3 ViewRootImpl 中各回调的处理

```java
onInputEvent(event)           → processRawInputEvent() → InputStage 链
onFocusEvent(hasFocus)        → windowFocusChanged(hasFocus)
onPointerCaptureEvent(...)    → dispatchPointerCaptureChanged(...)
onDragEvent(...)              → dragState 更新
onTouchModeChanged(...)       → touchModeChanged(inTouchMode)
onBatchedInputEventPending    → scheduleConsumeBatchedInput()
```

---

## 12. 关键源码文件索引

### 12.1 frameworks/native (Native InputFlinger)

| 文件 | 说明 |
|------|------|
| `services/inputflinger/dispatcher/InputDispatcher.cpp` | InputDispatcher 主逻辑 |
| `services/inputflinger/dispatcher/InputDispatcher.h` | InputDispatcher 类声明 |
| `services/inputflinger/reader/InputReader.cpp` | InputReader 主逻辑 |

### 12.2 frameworks/base (Java Framework)

| 文件 | 说明 |
|------|------|
| [InputManagerService.java](services/core/java/com/android/server/input/InputManagerService.java) | Java 侧输入服务 |
| [InputManagerCallback.java](services/core/java/com/android/server/wm/InputManagerCallback.java) | ANR 回调、Policy 拦截 |
| [AnrController.java](services/core/java/com/android/server/wm/AnrController.java) | ANR 处理、preDump |
| [AnrLatencyTracker.java](core/java/com/android/internal/os/anr/AnrLatencyTracker.java) | ANR 延迟追踪 |
| [InputMonitor.java](services/core/java/com/android/server/wm/InputMonitor.java) | 窗口列表构建、焦点管理 |
| [InputWindowHandleWrapper.java](services/core/java/com/android/server/wm/InputWindowHandleWrapper.java) | 变更检测 + 写入 SF Transaction |
| [InputConfigAdapter.java](services/core/java/com/android/server/wm/InputConfigAdapter.java) | LayoutParams → InputConfig 映射 |
| [ViewRootImpl.java](core/java/android/view/ViewRootImpl.java) | 应用端事件接收、InputStage 链 |
| [InputEventReceiver.java](core/java/android/view/InputEventReceiver.java) | 应用端事件接收基类 |
| [BatchedInputEventReceiver.java](core/java/android/view/BatchedInputEventReceiver.java) | 批量事件 vsync 对齐 |
| [InputChannel.java](core/java/android/view/InputChannel.java) | IPC 通道 (socket pair) |
| [SurfaceControl.java](core/java/android/view/SurfaceControl.java) | SF Transaction 接口 |
| [InputConstants.java](core/java/android/os/InputConstants.java) | ANR 超时常量 |
| [TimeoutRecord.java](core/java/com/android/internal/os/TimeoutRecord.java) | ANR 类型定义 |

### 12.3 JNI 桥

| 文件 | 说明 |
|------|------|
| [com_android_server_input_InputManagerService.cpp](services/core/jni/com_android_server_input_InputManagerService.cpp) | JNI 桥：InputManager ↔ native |
| [android_view_SurfaceControl.cpp](core/jni/android_view_SurfaceControl.cpp) | JNI 桥：SF Transaction |
| [android_view_InputEventReceiver.cpp](core/jni/android_view_InputEventReceiver.cpp) | JNI 桥：应用端事件接收 |
