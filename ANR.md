1. log中显示： 在Activity1：A1页面，侧滑手势导航栏开启RecentsTransition：t1，同时启动Activity2：A2，开启Open Transition：t2，A2启动成功，马上finish，开启CloseTransition：t3;
2. 其中t1，t3属于同一个track，t2在新的的track，侧滑很快结束，又回到了当前应用。t1,t2结束OPEN Transition最后结束；
3. 但是OPEN Transition 在结束时，在Shell侧，提交结束事务时，就会显示A2，隐藏A1 图层；
4. 同时通知core 将A1的visible设置为false。此时页面显示当前应用，但是所有ActivityRecord都被隐藏，没有焦点窗口了；
5. 看下日志：确认在 finishTransition Info 是否是 ActivityRecord 还是 Task
## 思路
1. ActivityRecord通过2个字段描述可见性，visible和requestVisible。A1启动A2，A1 pause成功后将分别A1,A2的requestVisible置为false，true；在所有的窗口已经就绪调用onTransactionReady准备播放动画前，会将requestVisible的窗口保存在集合中，当动画结束会遍历窗口将不在其中的窗口的visible属性置为false;
2. 方案 1：Server 侧 — `finishTransition` 中校验顶层 resumed activity（推荐）
	在 [Transition.java:1476](vscode-webview://0ehv9b7aq8bm0bkh8crpv71gesmvhg8egl0k1jo51it32i693gcf/services/core/java/com/android/server/wm/Transition.java#L1476) 之前加判断：

```java
// Transition.java finishTransition() — 在 commitVisibility 之前
if ((!visibleAtTransitionEnd || isScreenOff) && !ar.isVisibleRequested()) {
    // 新增：如果该 activity 当前是所在 task 的 resumed activity，跳过隐藏
    final ActivityRecord resumedInTask = ar.getTask() != null 
        ? ar.getTask().getTopResumedActivity() : null;
    if (resumedInTask == ar) {
        // A1 在 finish 时已经是 task 的 top resumed activity，
        // 说明中间发生了 A2 finish 等变化，不应提交 invisible
        ProtoLog.w(WM_DEBUG_WINDOW_TRANSITIONS,
            "Skip commitVisibility(false) for resumed activity %s", ar);
        continue;  // ← 跳过 commitVisibility(false)
    }
    ar.commitVisibility(false, false, true);
}
```

# todo 
1.  dump 下 recent task下的task的显示？
2.  切换焦点时 顶部的 Task 为什么是桌面？
3. finish 时 visible 怎么变化的？