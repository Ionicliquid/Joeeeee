1. 使用场景 ： fragment + Activity
2. 四化：模块化 组件化 层次化 控件化
3. 其他： 泛型？
4. common:WebViewService 接口下沉
5. auto service
6. addJavascript
7. 命令模式
8. Application 启动Activity 为什么要加flag?
-----

1. 利用组件化的思想重构WebView模块
	1. 为什么要重构：
		1. 功能增多 维护起来困难；
		2. 逻辑混乱，不可复用;
	2. 组件化：将代码中重复的部分提炼成以一个个组件供给功能使用，采用分层思想进行拆分 WebView模块将作为业务实现层，提供给宿主曾进行调用；
2. 