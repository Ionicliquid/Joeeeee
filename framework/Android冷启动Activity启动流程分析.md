# Android 冷启动 Activity 启动流程深度分析

> 基于 `system_server` WindowManager 日志，逆向分析从 Launcher 启动应用到目标 Activity 显示的完整流程。

---

## 目录

1. [问题背景](#1-问题背景)
2. [三条 setVisibility 调用链](#2-三条-setvisibility-调用链)
3. [调用链与共同入口的关系](#3-调用链与共同入口的关系)
4. [Launcher 冷启动完整流程（8 阶段）](#4-launcher-冷启动完整流程8-阶段)
5. [核心类方法速查表](#5-核心类方法速查表)
6. [完整时序图](#6-完整时序图)
7. [关键源码文件索引](#7-关键源码文件索引)

---

## 1. 问题背景

从 Launcher 启动应用（com.android.settings/.HWSettings），system_server 在 **同一毫秒**（`21:09:08.923`）输出了 3 条 `setAppVisibility` 日志：

```
setAppVisibility(HWSettings, visible=true)     ← 链①: realStartActivityLocked
setAppVisibility(DrawerLauncher, visible=false) ← 链②: makeInvisible
setAppVisibility(HWSettings, visible=false)     ← 链③: makeVisibleAndRestartIfNeeded
```

每条日志只打印了 6 层调用栈（`Callers=` 截断），本文还原其完整调用链路。

---

## 2. 三条 setVisibility 调用链

### 链①：HWSettings 被设为 `visible=true`

**来源**：`realStartActivityLocked` — 目标进程就绪后，直接显示 Activity。

```
ActivityRecord.setVisibility(visible=true)                          // ActivityRecord.java:6645
  └─ setVisibilityInner()
       ▲ ActivityTaskSupervisor.realStartActivityLocked             // :993
         ▲ ActivityTaskSupervisor.startSpecificActivityLocked       // :1293
           ▲ ActivityTaskSupervisor.startSpecificActivity           // :1274
             ▲ TaskFragment.resumeTopActivity                       // TaskFragment.java:2114
               ▲ Task.resumeTopActivityInnerLocked                  // Task.java:6819
                 ▲ Task.resumeTopActivityUncheckedLocked            // Task.java
                   ▲ RootWindowContainer.resumeFocusedTasksTopActivities
                     ▲ ActivityStarter.startActivityUnchecked
                       ▲ ActivityStarter.startActivityInner
                         ▲ ActivityStarter.executeRequest
                           ▲ ActivityTaskManagerService.startActivityAsUser
                             ▲ ActivityTaskManagerService.startActivity
                               ▲ [Binder call from Launcher]
```

---

### 链②：DrawerLauncher 被设为 `visible=false`

**来源**：`makeInvisible` — `EnsureActivitiesVisibleHelper` 判定 Launcher 被新的前台 Activity 遮挡，需要隐藏。

```
ActivityRecord.setVisibility(visible=false)                          // ActivityRecord.java:6645
  └─ setVisibilityInner()
       ▲ ActivityRecord.makeInvisible                                // ActivityRecord.java:7382
         ▲ EnsureActivitiesVisibleHelper.setActivityVisibilityState  // :302
           ▲ EnsureActivitiesVisibleHelper.process                   // :157
             ▲ TaskFragment.updateActivityVisibilities               // TaskFragment.java:1600
               ▲ TaskFragment.ensureActivitiesVisible                // TaskFragment.java
                 ▲ Task.ensureActivitiesVisible                      // Task.java:6543 (lambda)
                   ▲ RootWindowContainer.ensureActivitiesVisible     // RootWindowContainer.java
                     ▲ RootWindowContainer.ensureActivitiesVisible   // 递归遍历子容器
                       ▲ RootWindowContainer.resumeFocusedTasksTopActivities
                         ▲ ActivityStarter.startActivityUnchecked
                           ▲ ... (同链①上层)
```

---

### 链③：HWSettings 被设为 `visible=false`

**来源**：`makeVisibleAndRestartIfNeeded` — `EnsureActivitiesVisibleHelper` 在全局 visibility 结算时，先修正 HWSettings 的不一致状态（`visible=false` 但 `mVisibleRequested=true`），随后会再设回 `true` 以重启 transition。

```
ActivityRecord.setVisibility(visible=false)                          // ActivityRecord.java:6645
  └─ setVisibilityInner()
       ▲ EnsureActivitiesVisibleHelper.makeVisibleAndRestartIfNeeded // :332
         ▲ EnsureActivitiesVisibleHelper.setActivityVisibilityState  // :260
           ▲ EnsureActivitiesVisibleHelper.process                   // :157
             ▲ TaskFragment.updateActivityVisibilities               // TaskFragment.java:1600
               ▲ TaskFragment.ensureActivitiesVisible                // TaskFragment.java
                 ▲ Task.ensureActivitiesVisible                      // Task.java:6543 (lambda)
                   ▲ RootWindowContainer.ensureActivitiesVisible     // RootWindowContainer.java
                     ▲ ... (同链②上层)
```

---

## 3. 调用链与共同入口的关系

三条链 **共享同一个顶层入口** `RootWindowContainer.resumeFocusedTasksTopActivities()`，但分属两个阶段：

```
RootWindowContainer.resumeFocusedTasksTopActivities()   ← 共同入口
  │
  ├─ [阶段1：Resume 目标 Activity]
  │   Task.resumeTopActivityUncheckedLocked()
  │     └─ Task.resumeTopActivityInnerLocked()
  │         └─ TaskFragment.resumeTopActivity()
  │             └─ ActivityTaskSupervisor.startSpecificActivity()
  │                 └─ startSpecificActivityLocked()
  │                     └─ realStartActivityLocked()
  │                         └─ ActivityRecord.setVisibility(HWSettings, true)   ★ 链①
  │
  └─ [阶段2：全局 Visibility 结算]
      RootWindowContainer.ensureActivitiesVisible()
        └─ Task.ensureActivitiesVisible()
            └─ TaskFragment.ensureActivitiesVisible()
                └─ TaskFragment.updateActivityVisibilities()
                    └─ EnsureActivitiesVisibleHelper.process()
                        ├─ setActivityVisibilityState(HWSettings)
                        │   └─ makeVisibleAndRestartIfNeeded()
                        │       └─ setVisibility(HWSettings, false)            ★ 链③
                        │         (随后设回 true)
                        │
                        └─ setActivityVisibilityState(DrawerLauncher)
                            └─ makeInvisible()
                                └─ setVisibility(DrawerLauncher, false)        ★ 链②
```

### 总结

| 链 | 阶段 | 触发方法 | 目的 |
|----|------|----------|------|
| ① | Resume 阶段 | `realStartActivityLocked` | 新 Activity 进程就绪，直接标记可见 |
| ② | Visibility 结算阶段 | `makeInvisible` | Launcher 被新前台 Activity 遮挡，需隐藏 |
| ③ | Visibility 结算阶段 | `makeVisibleAndRestartIfNeeded` | 修正 HWSettings 不一致的可见性状态 |

### EnsureActivitiesVisibleHelper.process() 核心逻辑

```
for each ActivityRecord in 当前 Display 的所有 TaskFragment:
    1. setActivityVisibilityState()           → 链③
       ├─ 若 shouldBeVisible 但当前 visible=false
       │   └─ makeVisibleAndRestartIfNeeded()
       │       ├─ setVisibility(true)         // 设为可见
       │       └─ 必要时先 setVisibility(false) 重置状态
       │
       └─ 若 shouldBeInvisible 但当前 visible=true
           └─ setVisibility(false)

    2. 符合 shouldBeVisible 条件
       └─ container.ensureActivitiesVisible()  // 递归到下层容器

    3. 符合 shouldBeInvisible 条件
       └─ r.makeInvisible()                    → 链②
```

---

## 4. Launcher 冷启动完整流程（8 阶段）

### 阶段概览

```
┌─────────────────────────────────────────────────────────────────┐
│  1. Launcher 发起请求                                            │
│  2. ATMS 解析 Intent & 路由                                      │
│  3. 暂停 Launcher (pause)                                        │
│  4. 进程创建 (Zygote fork / ProcessList)                         │
│  5. 应用端初始化 (ActivityThread.bindApplication)                │
│  6. ATMS 启动目标 Activity (realStartActivityLocked)             │
│  7. 全局 Visibility 结算 (EnsureActivitiesVisibleHelper)         │
│  8. 应用端生命周期回调 (onCreate → onStart → onResume)           │
└─────────────────────────────────────────────────────────────────┘
```

---

### 阶段 1：Launcher 发起启动请求

```
用户点击桌面图标
  │
  ▼
Launcher.startActivitySafely(intent)
  └─ Activity.startActivity(intent)
       └─ Activity.startActivityForResult()
            └─ Instrumentation.execStartActivity()
                 └─ ActivityTaskManager.getService()
                      .startActivity(...)                  // Binder IPC → system_server
```

| 类 | 方法 | 说明 |
|----|------|------|
| `Activity` | `startActivity()` | 应用层入口 |
| `Instrumentation` | `execStartActivity()` | 拦截层，可被 ATMS 回调 |
| `ActivityTaskManagerService` | `startActivity()` | Binder 入口 |

---

### 阶段 2：ATMS 解析 Intent & 寻找/创建 Task

```
ActivityTaskManagerService.startActivity()
  │
  ▼
ActivityTaskManagerService.startActivityAsUser()
  │
  ▼
ActivityStarter.executeRequest(Request)                    // 构造启动上下文
  │
  ▼
ActivityStarter.startActivityInner()
  ├─ resolveIntent()                                      // 解析 Intent
  ├─ ActivityTaskSupervisor.getActivityOptions()           // 检查 LaunchOptions
  ├─ ActivityTaskSupervisor.resolveActivity()              // 通过 PMS 解析出 ResolveInfo
  ├─ computeLaunchingTaskFlags()                           // 计算 Task 标记
  │   ├─ LAUNCH_SINGLE_INSTANCE / SINGLE_TASK / SINGLE_TOP
  │   ├─ NEW_TASK / CLEAR_TOP / REORDER_TO_FRONT
  │   └─ FLAG_ACTIVITY_NEW_DOCUMENT 等
  └─ getOrCreateRootTask()                                // 找或建对应的显示屏根 Task
```

| ATMS 核心类 | 核心方法 | 说明 |
|-------------|----------|------|
| `ActivityTaskManagerService` | `startActivity()`, `startActivityAsUser()` | Binder 入口 |
| `ActivityStarter` | `executeRequest()`, **`startActivityInner()`** | 总调度枢纽 |
| `ActivityTaskSupervisor` | `resolveActivity()`, `getActivityOptions()` | 解析与全局策略 |
| `ActivityStartInterceptor` | `intercept()` | 拦截（如 work profile） |

---

### 阶段 3：复用/新建 Task & 暂停 Launcher

```
startActivityInner()
  │
  ▼
ActivityStarter.setNewTask()                               // 必要时新建 Task
  │
  ▼
ActivityStarter.resumeOrAddRootTask()
  ├─ 若有复用: moveToFront(reason="startedActivity")
  └─ 若新建: addChild() → positionChildAt()
  │
  ▼
ActivityStarter.startActivityUnchecked()                   // 确认启动参数
  │
  ▼
RootWindowContainer.resumeFocusedTasksTopActivities()
  │
  ├─▶ [目标 Task] resumeTopActivityUncheckedLocked()
  │     ⇒ 若进程不存在 → 走阶段 4 进程创建
  │     ⇒ 若进程存在 → 直接 resume
  │
  └─▶ [Launcher Task] pauseBackTasks()
```

| ATMS 核心类 | 核心方法 | 说明 |
|-------------|----------|------|
| `ActivityStarter` | `startActivityUnchecked()` | 最终参数确认与路由 |
| `Task` | `resumeTopActivityUncheckedLocked()` | Task 级别 resume 入口 |
| `TaskFragment` | `resumeTopActivity()` | Fragment 级（支持分屏/多窗） |
| `RootWindowContainer` | `resumeFocusedTasksTopActivities()` | **系统级 resume 总入口** |

---

### 阶段 4：进程创建（Zygote Fork）

> **关键点**：AMS 收到 Launcher 的 `activityPaused` 回调后，才会启动进程创建。

```
ActivityTaskSupervisor.startSpecificActivity()
  │
  ▼
ActivityTaskSupervisor.startSpecificActivityLocked()
  ├─ 检查 ProcessRecord（冷启动 → 不存在）
  │
  ▼
ActivityTaskManagerService.startProcessAsync()
  │
  ▼ [跨模块调用 → AMS]
  │
ActivityManagerService.startProcessLocked()
  ├─ 构建 ProcessRecord
  ├─ 收集启动所需信息:
  │   ├─ entryPoint = "android.app.ActivityThread"
  │   ├─ processName
  │   ├─ uid / gid
  │   └─ seInfo / mountMode 等
  │
  ▼
ProcessList.startProcessLocked()                            // AMS 中的 ProcessList
  │
  ▼
Process.start()
  ├─ ZygoteProcess.start()  → socket 通信
  │
  ▼
ZygoteServer (daemon 进程)
  └─ Zygote.forkAndSpecialize()                             // Linux fork()
       └─ 子进程执行: RuntimeInit.applicationInit()
            └─ ActivityThread.main()                        // ★ 应用进程入口
```

| AMS 核心类 | 核心方法 | 说明 |
|------------|----------|------|
| `ActivityManagerService` | `startProcessLocked()` | AMS 侧进程创建枢纽 |
| `ProcessList` | `startProcessLocked()` | 管理进程列表与启动 |
| `ProcessRecord` | — | 进程记录对象 |
| `ZygoteProcess` | `start()` | Socket 通信发起 fork 请求 |
| `ZygoteInit` | `forkAndSpecialize()` | Zygote fork 子进程 |

---

### 阶段 5：应用端初始化与绑定

```
[子进程] ActivityThread.main()
  │
  ▼
Looper.prepareMainLooper()
  │
  ▼
ActivityThread.attach(false)
  │
  ▼ [Binder IPC → system_server]
ActivityManagerService.attachApplication(thread)
  │
  ▼
ActivityManagerService.attachApplicationLocked()
  ├─ thread.bindApplication()                              // Binder 回调 → 应用端
  │   └─ [应用端] ActivityThread.handleBindApplication()
  │       ├─ makeApplication()
  │       ├─ installContentProviders()
  │       └─ Application.onCreate()                        // ★ Application 初始化
  │
  ├─ mAtmInternal.attachApplication(app)
  │   └─ ActivityTaskManagerService.attachApplication()
  │
  ▼
RootWindowContainer.attachApplication()
  │
  ▼
ActivityTaskSupervisor.realStartActivityLocked()            // ★ 目标 Activity 启动入口
```

| ATMS 核心类 | 核心方法 | 说明 |
|-------------|----------|------|
| `ActivityTaskSupervisor` | **`realStartActivityLocked()`** | 应用 Activity 真正启动执行点 |
| `ClientLifecycleManager` | `scheduleTransaction()` | 管理客户端生命周期事务 |
| `ClientTransaction` | — | 封装生命周期回调指令 |
| `LaunchActivityItem` | — | onCreate 事务 |
| `ResumeActivityItem` | — | onResume 事务 |

| 应用端核心类 | 核心方法 | 说明 |
|-------------|----------|------|
| `ActivityThread` | `main()`, `handleBindApplication()`, `handleLaunchActivity()` | 应用进程主线程 |
| `Instrumentation` | `newActivity()`, `callActivityOnCreate()` | Activity 实例化与回调调度 |
| `Activity` | `attach()`, `performCreate()`, `performStart()` | Activity 自身 |

---

### 阶段 6：启动目标 Activity（realStartActivityLocked）

```
ActivityTaskSupervisor.realStartActivityLocked()
  │
  ├─ ActivityRecord.setVisibility(true)                     // ★ 您的链①
  │
  ▼
ClientTransaction.obtain(thread, appToken)
  └─ LaunchActivityItem { intent, ... }
  └─ ResumeActivityItem
      │
      ▼ [Binder IPC → 应用端]
  ClientTransaction.schedule()
      │
      ▼
  [应用端] ActivityThread.handleLaunchActivity()
      ├─ performLaunchActivity()
      │   ├─ createBaseContextForActivity()
      │   ├─ Instrumentation.newActivity()
      │   ├─ Activity.attach()
      │   └─ Instrumentation.callActivityOnCreate()         // ★ Activity.onCreate()
      │
      └─ handleStartActivity()
          └─ Activity.performStart()
              └─ Instrumentation.callActivityOnStart()      // ★ Activity.onStart()
```

---

### 阶段 7：全局 Visibility 结算

```
realStartActivityLocked() 返回后
  │
  ▼
RootWindowContainer.ensureActivitiesVisible()
  │
  ▼
EnsureActivitiesVisibleHelper.process()
  ├─ setActivityVisibilityState(HWSettings)                 ← ★ 您的链③
  │   └─ makeVisibleAndRestartIfNeeded()
  │
  └─ setActivityVisibilityState(DrawerLauncher)             ← ★ 您的链②
      └─ makeInvisible()
```

---

### 阶段 8：最终 Resume（onResume 回调）

```
Task.resumeTopActivityInnerLocked()
  │
  ▼
ActivityTaskSupervisor.startSpecificActivity()
  (此时进程已存在，走 resume 路径)
  │
  ▼
TaskFragment.resumeTopActivity()
  │
  ▼
ClientTransaction
  └─ ResumeActivityItem
      │
      ▼ [Binder IPC → 应用端]
  ActivityThread.handleResumeActivity()
      └─ Activity.performResume()
          └─ Instrumentation.callActivityOnResume()         // ★ Activity.onResume()
              └─ WindowManager.addView()
                  └─ 第一帧绘制
```

---

## 5. 核心类方法速查表

### 各阶段关键类总览

| 阶段 | 关键类 (system_server) | 关键方法 |
|------|----------------------|---------|
| **入口** | `ActivityTaskManagerService` | `startActivity()` |
| **解析 & 路由** | `ActivityStarter` | `executeRequest()`, `startActivityInner()`, `startActivityUnchecked()` |
| **Task 管理** | `Task`, `TaskFragment` | `resumeTopActivityUncheckedLocked()` |
| **全局调度** | `RootWindowContainer` | `resumeFocusedTasksTopActivities()`, `ensureActivitiesVisible()` |
| **进程创建** | `ActivityManagerService`, `ProcessList`, `ZygoteProcess` | `startProcessLocked()`, `start()` |
| **启动执行** | `ActivityTaskSupervisor` | `realStartActivityLocked()` ⭐ |
| **可见性** | `EnsureActivitiesVisibleHelper` | `process()` |
| **生命周期事务** | `ClientTransaction`, `ClientLifecycleManager` | `scheduleTransaction()` |
| **应用端入口** | `ActivityThread` | `main()`, `handleBindApplication()`, `handleLaunchActivity()` |

### ActivityStarter 核心流程

| 方法 | 作用 |
|------|------|
| `executeRequest()` | 构造 Request 对象，启动调度 |
| `startActivityInner()` | 解析 Intent、计算 flags、分配 Task |
| `startActivityUnchecked()` | 最终确认后执行 resume |
| `resumeOrAddRootTask()` | Task 复用/新建与置顶 |
| `setNewTask()` | 创建新 Task 容器 |
| `computeLaunchingTaskFlags()` | 解析 launchMode 与 flags |

### 进程创建链路

| 类 (全路径) | 关键方法 | 说明 |
|-------------|----------|------|
| `ActivityTaskSupervisor` | `startSpecificActivityLocked()` | 检测 ProcessRecord |
| `ActivityTaskManagerService` | `startProcessAsync()` | 委托 AMS 创建进程 |
| `ActivityManagerService` | `startProcessLocked()` | 构建 ProcessRecord |
| `ProcessList` | `startProcessLocked()` | 遍历 process list 启动 |
| `ZygoteProcess` | `start()`, `startViaZygote()` | Socket → Zygote |
| `ZygoteInit` | `forkAndSpecialize()` | Daemon fork |

### 可见性管理

| 类 | 方法 | 说明 |
|----|------|------|
| `EnsureActivitiesVisibleHelper` | `process()` | 全局可见性计算入口 |
| `EnsureActivitiesVisibleHelper` | `setActivityVisibilityState()` | 单个 Activity 状态判定 |
| `EnsureActivitiesVisibleHelper` | `makeVisibleAndRestartIfNeeded()` | 重启 transition |
| `ActivityRecord` | `setVisibility()` | 设置可见性标记 |
| `ActivityRecord` | `makeInvisible()` | 隐藏 Activity |
| `ActivityRecord` | `makeVisibleIfNeeded()` | 恢复 Activity 可见性 |

### 应用端

| 类 | 方法 | 说明 |
|----|------|------|
| `ActivityThread` | `main()` | 应用主线程入口 |
| `ActivityThread` | `attach()` | 绑定到 AMS |
| `ActivityThread` | `handleBindApplication()` | Application 初始化 |
| `ActivityThread` | `handleLaunchActivity()` | 执行 onCreate + onStart |
| `ActivityThread` | `handleResumeActivity()` | 执行 onResume |
| `Instrumentation` | `callActivityOnCreate()` | 回调 onCreate |
| `Instrumentation` | `callActivityOnStart()` | 回调 onStart |
| `Instrumentation` | `callActivityOnResume()` | 回调 onResume |

---

## 6. 完整时序图

```
Launcher进程      ATMS/AMS(system_server)      Zygote         目标App进程
     │                    │                      │                │
     │──startActivity──▶  │                      │                │
     │                    │                      │                │
     │                    │──startActivityInner() │                │
     │                    │──解析 Intent          │                │
     │                    │──computeFlags()       │                │
     │                    │──setNewTask()         │                │
     │                    │──startActivityUnchecked()             │
     │                    │                      │                │
     │                    │──resumeFocusedTasksTopActivities()    │
     │                    │──pause Launcher       │                │
     │◀──onPause─────────│                      │                │
     │                    │                      │                │
     │                    │──startSpecificActivityLocked()        │
     │                    │──ProcessRecord == null (冷启动)       │
     │                    │──startProcessAsync()  │                │
     │                    │                      │                │
     │                    │──ZygoteProcess.start() │               │
     │                    │─────────────────────▶│                │
     │                    │                      │──fork()────────▶│
     │                    │                      │                │
     │                    │                      │              ActivityThread.main()
     │                    │                      │                │──prepareMainLooper()
     │                    │                      │                │──attach()
     │                    │                      │                │
     │                    │◀──attachApplication(thread)───────────│
     │                    │                      │                │
     │                    │──bindApplication()────────────────────▶│
     │                    │                      │                │──makeApplication()
     │                    │                      │                │──Application.onCreate()
     │                    │                      │                │
     │                    │──★ realStartActivityLocked()          │
     │                    │──setVisibility(HWSettings,true)[链①]  │
     │                    │                      │                │
     │                    │──ClientTransaction────────────────────▶│
     │                    │  (LaunchActivityItem)  │                │──performLaunchActivity()
     │                    │                      │                │──onCreate()
     │                    │                      │                │──onStart()
     │                    │                      │                │
     │                    │──★ ensureActivitiesVisible()          │
     │                    │──EnsureActivitiesVisibleHelper        │
     │                    │  ├─ setVisibility(HWSettings,false)[链③]
     │                    │  │  (然后设回 true)
     │                    │  └─ makeInvisible(Launcher) [链②]     │
     │                    │                      │                │
     │                    │──ClientTransaction────────────────────▶│
     │                    │  (ResumeActivityItem)  │                │──performResume()
     │                    │                      │                │──onResume()
     │                    │                      │                │
     │                    │                      │                │──WindowManager.addView()
     │                    │                      │                │──第一帧绘制
```

---

## 7. 关键源码文件索引

| 模块 | 文件路径 |
|------|---------|
| ATMS 入口 | `frameworks/base/services/core/java/com/android/server/wm/ActivityTaskManagerService.java` |
| 启动调度 | `frameworks/base/services/core/java/com/android/server/wm/ActivityStarter.java` |
| 全局策略 | `frameworks/base/services/core/java/com/android/server/wm/ActivityTaskSupervisor.java` |
| 可见性结算 | `frameworks/base/services/core/java/com/android/server/wm/EnsureActivitiesVisibleHelper.java` |
| Activity 记录 | `frameworks/base/services/core/java/com/android/server/wm/ActivityRecord.java` |
| Task 容器 | `frameworks/base/services/core/java/com/android/server/wm/Task.java` |
| TaskFragment | `frameworks/base/services/core/java/com/android/server/wm/TaskFragment.java` |
| 窗口容器根 | `frameworks/base/services/core/java/com/android/server/wm/RootWindowContainer.java` |
| 生命周期事务 | `frameworks/base/services/core/java/com/android/server/wm/ClientLifecycleManager.java` |
| 客户端事务 | `frameworks/base/core/java/android/app/servertransaction/ClientTransaction.java` |
| AMS 进程管理 | `frameworks/base/services/core/java/com/android/server/am/ActivityManagerService.java` |
| 进程列表 | `frameworks/base/services/core/java/com/android/server/am/ProcessList.java` |
| Zygote 通信 | `frameworks/base/core/java/android/os/ZygoteProcess.java` |
| Zygote 初始化 | `frameworks/base/core/java/com/android/internal/os/ZygoteInit.java` |
| 应用主线程 | `frameworks/base/core/java/android/app/ActivityThread.java` |
| 插桩 | `frameworks/base/core/java/android/app/Instrumentation.java` |
