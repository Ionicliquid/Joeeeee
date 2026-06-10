1. A1 -> A2 创建Transition1  A1 pause成功将mVisibleRequesed置为false,A2的starting Window会绘制完成之后，触发焦点更新，此时mFocusedWindow为空，同时回调onTransitionReady准备播放启动动画，动画结束A1隐藏，A2显示；
2. 启动过程中，侧滑导航栏触发recent，A2启动后马上finish;这时创建Transition2 和Transition3。
3. 日志中显示：Transition1在Transition2和Transition3之后结束。
	1. 在分配TrackId时，启动Transition1 的id为0，Transition2和Transition3为1：merge后Id为1？
4. Transition1在最后结束，提交结束事务，
5. A2 finish，动画结束 A2 的窗口隐藏，A1的显示

# todo 
1.  dump 下 recent task下的task的显示？
2.  切换焦点时 顶部的 Task 为什么是桌面？
3. 