# 阈值检测
## 产生OOM的原因
1. Java堆内存溢出
2. 无足够连续的内存空间
3. 打开文件数量超出限制
4. 线程数量超出限制
**<font color="#953734">5. 虚拟内存不足</font>**
# dump内存快照
![[koom_dump.png]]
1. dump属于耗时操作，无论在子线程中还是在主线程中，都会挂起当前进程中的所有线程；
2. 通过fork 应用进程，在子进程中进行dump；
	1. 只将发起fork()调用的线程复制到子进程中，但全局变量的状态以及所有的pthreads对象（如互斥量、条件变量等）都会在子进程中得以保留；因此，当子进程中进行dump hprof时，SuspendAll触发暂停是永远等不到其他线程返回结果，从而导致子进程死锁；
3. 先在主进程执行SuspendAll，使ThreadList中保存的所有线程状态为suspend，之后fork，此时子进程执行dump hprof，由于其共享父进程的ThreadList全局变量，因此认为全部线程已经处于suspend状态，避免了子进程dump时SuspendAll触发暂停等不到线程返回结果的情况。fork完毕父进程即可立刻执行ResumeAll恢复运行。
# Hprof文件分析
1. Hprof文件由文件头+record数组组成
2. 文件头
3. Record： 

## 内存泄漏


# 参考链接
- [Android 内存泄露工具 leakcanary 2.6 浅析 | 伪斜杠青年](https://i.lckiss.com/?p=6735#h2-9)
- [Hprof文件解析](https://leo-wxy.github.io/2020/12/14/Hprof%E6%96%87%E4%BB%B6%E8%A7%A3%E6%9E%90/#Hprof%E6%96%87%E4%BB%B6%E8%A3%81%E5%89%AA)
- [fork()和多线程\_fork 多线程-CSDN博客](https://blog.csdn.net/u011608357/article/details/38927305)