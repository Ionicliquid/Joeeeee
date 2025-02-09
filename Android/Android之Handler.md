# 总结
1. 每个线程只有一个`Looper`，主线程的`Looper`，在应用进程创建时创建并启动；
2. `Looper`启动后，开启轮询从消息队列`MessageQueue`中获取消息`Message`并处理；
3. MessageQueue 名为队列，实际上按照消息的执行时间进行排序的<font color="#ff0000">todo:单向链表？</font>
