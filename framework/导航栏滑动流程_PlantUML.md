# 导航栏滑动 — 全场景流程 PlantUML 图

## 场景一: 从 App 上滑距离不够 → 回弹到 App (LAST_TASK)

```plantuml
@startuml
!theme plain
skinparam backgroundColor #FEFEFE
skinparam sequenceMessageAlign center

title 场景一: 从App上滑距离不够 → 回弹到App (LAST_TASK)\n动画 = 本地ValueAnimator，非Transition框架

actor "用户手指" as User
participant "SystemUI\nNavigationBar" as NavBar
participant "Launcher进程\nTouchInteractionService" as TIS
participant "OtherActivity\nInputConsumer" as OAIC
participant "AbsSwipeUpHandler\n(Launcher)" as Handler
participant "TaskAnimation\nManager" as TAM
participant "SystemUiProxy\n(Binder桥接)" as Proxy
participant "Shell进程\nIRecentTasks" as Shell
participant "WMS\nWindowManager" as WMS

User -> NavBar: 触摸导航栏 (ACTION_DOWN)
NavBar -> TIS: onInputEvent()
TIS -> OAIC: onMotionEvent(ACTION_DOWN)

OAIC -> OAIC: startTouchTrackingForWindowAnimation()
OAIC -> TAM: mTaskAnimationManager\n.startRecentsAnimation(gestureState, intent)
TAM -> TAM: 检查是否已有运行中的动画\nforceFinish旧动画
TAM -> Proxy: SystemUiProxy.startRecentsActivity(intent, options, listener)
Proxy -> Shell: IRecentTasks.startRecentsTransition(\n  pendingIntent, intent, optionsBundle,\n  IRecentsAnimationRunner)
Shell -> WMS: ★ 启动 Recents Animation ★\n- 获取 App Window Leash (SurfaceControl)\n- 准备 RemoteAnimationTarget[]

User -> NavBar: ACTION_MOVE (滑动中)
NavBar -> TIS -> OAIC: onMotionEvent(ACTION_MOVE)
OAIC -> OAIC: 检查是否 passTouchSlop
OAIC -> Handler: onGestureStarted() & updateDisplacement()
Handler -> Handler: onCurrentShiftUpdated()\n→ applyScrollAndTransform()\n→ TaskViewSimulator 更新 Surface 位置

note right of Handler: 用户实时看到窗口\n跟随手指移动

User -> NavBar: ACTION_UP (松手，距离不够)
NavBar -> TIS -> OAIC: onMotionEvent(ACTION_UP)
OAIC -> OAIC: finishTouchTracking()
OAIC -> Handler: onGestureEnded(velocity, ...)

Handler -> Handler: handleNormalGestureEnd()

== 判定目标 ==

Handler -> Handler: calculateEndTarget()
note right of Handler #FFEBEE: **非Fling + velocity.y > 0\n→ LAST_TASK**

== 回弹动画 (本地，非Transition) ==

Handler -> Handler: animateToProgressInternal()
note right of Handler #E3F2FD: **else分支 (endTarget != HOME):**\nAnimatorSet + ValueAnimator\nmCurrentShift.animateToValue(start, 0)\n\n✅ 这就是回弹动画!\n是本地属性动画，不是Shell Transition

Handler -> Handler: 每帧 onCurrentShiftUpdated()
Handler -> Handler: applyScrollAndTransform()
Handler -> WMS: SurfaceControl.Transaction\n(通过 TaskViewSimulator)

Handler -> Handler: 动画结束\nSTATE_END_TARGET_ANIMATION_FINISHED

== 清理 ==

Handler -> Handler: onSettledOnEndTarget()\n→ case LAST_TASK\n→ resumeLastTask()
Handler -> Handler: mRecentsAnimationController\n.finish(false, null)
Handler -> Shell: IRecentsAnimationController\n.finish(toRecents=false, ...)
Shell -> WMS: 清理 Recents Animation\n恢复 App 前台

@enduml
```

---

## 场景二: 从 App 上滑足够 → 回桌面 (HOME)

```plantuml
@startuml
!theme plain
skinparam backgroundColor #FEFEFE
skinparam sequenceMessageAlign center

title 场景二: 从App上滑足够 → 回桌面 (HOME)\n动画 = RectFSpringAnim(弹簧物理) + HomeAnimationFactory

participant "AbsSwipeUpHandler" as Handler
participant "HomeAnimation\nFactory" as HomeFactory
participant "RectFSpringAnim\n(窗口弹簧动画)" as Spring
participant "QuickstepTransition\nManager" as QTM
participant "RecentsAnimation\nController" as RAC
participant "Shell / WMS" as Shell

... 前期流程同场景一 ...

Handler -> Handler: calculateEndTarget()
note right of Handler #C8E6C9: **isFlingY && isSwipeUp\n→ HOME**\n或 velocity.y < 0\n&& mCanSlowSwipeGoHome → HOME

Handler -> Handler: animateToProgressInternal()
note right of Handler #E3F2FD: **HOME 分支 (line 1658):**

Handler -> HomeFactory: createHomeAnimationFactory()
Handler -> Spring: createWindowAnimationToHome(start, homeAnimFactory)
note right of Spring: 弹簧物理动画:\nApp窗口 → 图标位置

Handler -> HomeFactory: playAtomicAnimation(velocity)
note right of HomeFactory: Launcher 内容动画:\nWorkspace 回位、图标缩放等

Handler -> Handler: if (mHandOffAnimationToHome):\nhandOffAnimation(velocity)
note right of Handler #FFF3E0: **仅当 LongLivedReturnAnimations\n启用时使用 Shell Transition**

Spring -> Spring: start()
Spring -> WMS: 每帧 SurfaceTransaction\n更新 App 窗口位置

Spring -> Handler: onAnimationSuccess()
Handler -> RAC: finishAnimationToHome()
RAC -> Shell: IRecentsAnimationController\n.finish(toRecents=true)

@enduml
```

---

## 场景三: 从 App 上滑 → 暂停 → 进入多任务 (RECENTS)

```plantuml
@startuml
!theme plain
skinparam backgroundColor #FEFEFE
skinparam sequenceMessageAlign center

title 场景三: 从App上滑 → 暂停 → 进入多任务 (RECENTS)\n动画 = ValueAnimator + mCurrentShift 到1

participant "MotionPause\nDetector" as MPD
participant "AbsSwipeUpHandler" as Handler
participant "RecentsView" as RV
participant "Launcher\nStateManager" as SM
participant "RecentsAnimation\nController" as RAC
participant "Shell / WMS" as Shell

... 滑动手势进行中 ...

MPD -> MPD: onMotionPauseDetected()\nmIsMotionPaused = true
MPD -> Handler: maybeUpdateRecentsAttachedState()
note right of Handler: RecentsView 附着到\nApp 窗口下方

User -> Handler: ACTION_UP (松手)

Handler -> Handler: calculateEndTarget()
note right of Handler #C8E6C9: **mIsMotionPaused == true\n→ RECENTS**

Handler -> Handler: animateToProgressInternal()
note right of Handler #E3F2FD: **else分支:**\nendShift = 1 (RECENTS.isLauncher)\nAnimationSet 动画 mCurrentShift 到 1

Handler -> Handler: onSettledOnEndTarget()
note right of Handler: → case RECENTS

Handler -> RAC: detachNavigationBarFromApp(true)
Handler -> RAC: finish(true, null)
RAC -> Shell: IRecentsAnimationController\n.finish(toRecents=true)

Handler -> Handler: setupLauncherUiAfterSwipeUpToRecentsAnimation()
Handler -> SM: Launcher 进入 OVERVIEW 状态
Handler -> RV: RecentsView 显示多任务卡片

@enduml
```

---

## 场景四: 从 App 横向滑动 → 切换到另一个 App (NEW_TASK)

```plantuml
@startuml
!theme plain
skinparam backgroundColor #FEFEFE
skinparam sequenceMessageAlign center

title 场景四: 从App横向滑动 → 切换到另一个App (NEW_TASK)\n涉及 WMS Transition 启动新Activity

participant "AbsSwipeUpHandler" as Handler
participant "RecentsView" as RV
participant "ActivityManager\nWrapper" as AMW
participant "WMS" as WMS
participant "Shell" as Shell

... 滑动手势进行中 ...

Handler -> RV: 用户横向滑动\nRecentsView.scrollTo()

User -> Handler: ACTION_UP

Handler -> Handler: calculateEndTarget()
note right of Handler #C8E6C9: **isScrollingToNewTask()\n→ NEW_TASK**

Handler -> Handler: animateToProgressInternal()
note right of Handler: endShift = 1\n动画到 Recents 视图

Handler -> Handler: onSettledOnEndTarget()
note right of Handler: → case NEW_TASK

Handler -> RV: getNextPageTaskView()
Handler -> AMW: startActivityFromRecents(taskId)
AMW -> WMS: ★ WMS Transition ★\n启动新 App 窗口动画

Handler -> Handler: mRecentsAnimationController\n.finish(false, null)
Handler -> Shell: 清理 Recents Animation

note over WMS: 新 App Activity 启动\n窗口由 WMS 管理

@enduml
```

---

## 场景五: 从 Launcher 桌面 → 上滑进入多任务 (NORMAL → OVERVIEW)

```plantuml
@startuml
!theme plain
skinparam backgroundColor #FEFEFE
skinparam sequenceMessageAlign center

title 场景五: Launcher桌面 → 多任务 (NORMAL → OVERVIEW)\n完全不经过 Shell RecentsAnimation！纯 Launcher 内部状态切换

actor "用户手指" as User
participant "NoButtonNavbarTo\nOverviewTouchController" as NBTC
participant "SingleAxis\nSwipeDetector" as SASD
participant "AnimatorPlayback\nController" as APC
participant "StateManager" as SM

User -> NBTC: ACTION_DOWN (在Launcher桌面)
NBTC -> SASD: 初始化手势检测

User -> NBTC: ACTION_MOVE (上滑)
NBTC -> SASD: onDrag(displacement)
SASD -> APC: setProgress(displacement * mProgressMultiplier)

note right of APC #E3F2FD: **Launcher 内部 View 属性动画:**\n- Workspace 缩放\n- Hotseat 淡出\n- AllApps 准备\n- 图标移动

User -> NBTC: ACTION_UP (松手)
NBTC -> NBTC: 基于进度 + fling 判定目标
note right of NBTC #C8E6C9: 进度 > 阈值 → OVERVIEW\n否则 → NORMAL

NBTC -> APC: animateToValue(start, end)
note right of APC: **本地 ValueAnimator snap**\n不涉及 Shell

APC -> SM: 动画完成 → goToState(OVERVIEW)
SM -> SM: 状态机切换完成

note over NBTC, SM #FFEBEE: **完全不涉及:**\n❌ OtherActivityInputConsumer\n❌ AbsSwipeUpHandler\n❌ RecentsAnimationController\n❌ SystemUiProxy\n❌ Shell Transition 框架

@enduml
```

---

## 场景六: Predictive Back 手势 → 返回桌面

```plantuml
@startuml
!theme plain
skinparam backgroundColor #FEFEFE
skinparam sequenceMessageAlign center

title 场景六: Predictive Back 手势 → 返回桌面\n使用 Shell Transition 框架 (RemoteTransition)

actor "用户手指" as User
participant "WMS\nBackNavigation" as BackNav
participant "LauncherBackAnimation\nController" as LBAC
participant "QuickstepTransition\nManager (QTM)" as QTM
participant "Shell Transition\nFramework" as ShellTF
participant "WMS" as WMS

User -> BackNav: 从屏幕边缘向内滑动
BackNav -> LBAC: onBackStarted(backEvent)
note right of LBAC: ★ 注册了 IOnBackInvokedCallback

BackNav -> LBAC: onBackProgressed(backEvent)
LBAC -> LBAC: 更新窗口位置\n更新 Launcher 预览

User -> BackNav: 松手 (确认返回)

BackNav -> LBAC: onBackInvoked()

LBAC -> QTM: getWallpaperOpenRunner()
note right of QTM: WallpaperOpenLauncher\nAnimationRunner\n(RemoteAnimationFactory)

QTM -> ShellTF: ★ RemoteTransition ★\nTRANSIT_TO_BACK
ShellTF -> QTM: onAnimationStart(\n  transit, appTargets[],\n  wallpaperTargets[], ...)

QTM -> QTM: composeAnimation():
note right of QTM #E3F2FD: - RectFSpringAnim (App窗口 → 图标)\n- WorkspaceRevealAnim\n- 导航栏淡入\n- StatusBar 过渡

QTM -> WMS: SurfaceControl.Transaction\n每帧提交

QTM -> ShellTF: onTransitionFinished()

@enduml
```

---

## 场景七: 从 App 三键导航 → 进入多任务 (Button Mode)

```plantuml
@startuml
!theme plain
skinparam backgroundColor #FEFEFE
skinparam sequenceMessageAlign center

title 场景七: 三键导航 → 进入多任务 (Button Mode)\nAtomic Event + Shell RecentsAnimation

participant "TwoButtonNavbar\nTouchController" as TBTC
participant "AbsSwipeUpHandler" as Handler
participant "SystemUiProxy" as Proxy
participant "Shell" as Shell

User -> TBTC: 点击 Recents 按钮

TBTC -> Handler: onGestureStarted(atomic=true)
TBTC -> Handler: onGestureEnded(0, ...)

Handler -> Handler: calculateEndTarget()
note right of Handler #C8E6C9: **isHandlingAtomicEvent()\n→ RECENTS**

Handler -> Handler: handleNormalGestureEnd()
note right of Handler: isAtomic → 没有滑动过程\n直接动画到 RECENTS

Handler -> Handler: animateToProgressInternal()
note right of Handler: startShift=0, endShift=1\nduration=MAX_SWIPE_DURATION

Handler -> Shell: finish(true) (toRecents)
Shell -> Shell: 清理

@enduml
```

---

## 全场景对比表

```plantuml
@startuml
!theme plain
skinparam backgroundColor #FEFEFE

title 导航栏手势 — 全场景动画控制架构总览

left to right direction

rectangle "Shell进程\n(WMS/SystemUI)" as Shell #FFCDD2 {
  usecase "IRecentTasks\nstartRecentsTransition" as IRT
  usecase "IRecentsAnimationController\nfinish(toRecents)" as IAC
  usecase "Shell Transition\n(RemoteTransition)" as ST
}

rectangle "Launcher进程" as Launcher #C8E6C9 {
  usecase "TouchInteractionService\nInputConsumer" as TIS
  usecase "AbsSwipeUpHandler\n手势处理核心" as Handler
  usecase "QuickstepTransitionManager\n(TM)" as QTM
  usecase "AbstractStateChangeTouch\nController" as ASCTC
}

rectangle "WMS" as WMS #BBDEFB {
  usecase "WindowManager\nSurface管理" as WM
  usecase "Transition生命周期" as TLife
}

' 场景连线
note top of TIS: **场景1234入口**\nOtherActivityInputConsumer

note top of ASCTC: **场景5入口**\nNoButtonNavbarToOverviewTouchController

TIS --> Handler : 场景1-4
Handler --> IRT : startRecentsAnimation
IRT --> IAC : 回调 onAnimationStart
Handler --> IAC : finish(toRecents)
QTM --> ST : RemoteAnimationFactory\n(场景2 handOff + 场景6)
ASCTC --> ASCTC : 纯Launcher内部\n状态机切换

@enduml
```
