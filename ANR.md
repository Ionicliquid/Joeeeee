1. log中显示： 在Activity1：A1页面，侧滑手势导航栏开启RecentsTransition：t1，同时启动Activity2：A2，开启Open Transition：t2，A2启动成功，马上finish，开启CloseTransition：t3;
2. 其中t1，t3属于同一个track，t2在新的的track，侧滑很快结束，又回到了当前应用。t1,t2结束OPEN Transition最后结束；
3. 但是OPEN Transition 在结束时，在Shell侧，提交结束事务时，就会显示A2，隐藏A1 图层；
4. 同时通知core 将A1的visible设置为false。此时页面显示当前应用，但是所有ActivityRecord都被隐藏，没有焦点窗口了；
## 思路
1. ActivityRecord通过2个字段描述可见性，visible和requestVisible。A1启动A2，A1 pause成功后将分别A1,A2的requestVisible置为false，true；在所有的窗口已经就绪调用onTransactionReady准备播放动画前，会将requestVisible的窗口保存在集合中，当动画结束会遍历窗口将不在其中的窗口的visible属性置为false;

# todo 
1.  dump 下 recent task下的task的显示？
2.  切换焦点时 顶部的 Task 为什么是桌面？
3. 