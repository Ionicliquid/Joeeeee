以下是为10年经验的Android高级开发工程师准备的大疆和比亚迪面试题集，结合两者的技术侧重点及高频考点整理而成。题目涵盖**基础原理、高级特性、系统底层、项目实战**等维度，并标注了考察重点及参考资料：

---
## 大疆
### **一、Android基础与框架原理**
1. **Activity启动模式与任务栈**  
   - 描述`standard`、`singleTop`、`singleTask`、`singleInstance`的应用场景，并举例说明不同模式下Activity栈的变化（如：A→B→C→D→A→B的栈结构）。  
   - `onNewIntent()`的调用时机及数据更新逻辑。

2. **Fragment生命周期与通信**  
   - Fragment与Activity生命周期的关联性，如何处理Fragment重叠问题？  
   - 跨Fragment通信的几种方式（ViewModel、接口回调、EventBus等）。

3. **Jetpack组件**  
   - LiveData如何避免内存泄漏？与RxJava的区别？  
   - ViewModel的生命周期管理及与SavedState的结合使用场景。

4. **多线程与协程**  
   - Handler、Looper、MessageQueue的关系，如何在子线程创建Handler？  
   - 协程的挂起函数原理，对比`launch`与`async`的异同。

---

### **二、UI与性能优化**
1. **事件分发与自定义View**  
   - 事件分发流程中`onTouch`、`onTouchEvent`、`onInterceptTouchEvent`的执行顺序，若`onTouch`的DOWN返回true而MOVE返回false，后续事件如何处理？  
   - 自定义View时如何优化`onMeasure`、`onLayout`的性能？。

2. **布局与渲染优化**  
   - `ConstraintLayout`的优缺点及替代方案（如Compose布局）。  
   - `RecyclerView`四级缓存机制，滑动10项再滑回时的`onBindViewHolder`调用次数。

3. **性能调优实战**  
   - ANR的定位方法（如分析`/data/anr/traces.txt`）及避免策略。  
   - 内存泄漏检测工具（LeakCanary）的使用及常见泄漏场景（如静态Handler、未注销监听）。

---

### **三、系统底层与跨进程通信**
4. **Binder与AIDL**  
   - Binder的MMAP机制及与Linux IPC的对比。  
   - AIDL中`Stub`与`Proxy`的作用，如何实现单向或双向通信？。

5. **系统服务与启动流程**  
   - AMS、WMS、PMS的核心职责及启动流程。  
   - Android系统从开机到桌面加载的流程，Zygote与SystemServer的作用。

6. **车载开发专项（比亚迪重点）**  
   - Android Automotive OS的架构与手机版Android的差异。  
   - 车载系统中Service的保活策略及与Linux内核的交互（如Binder驱动优化）。

---

### **四、网络与架构设计**
#### **网络请求与安全**  
##### OkHttp的拦截器链实现原理，如何自定义缓存策略？  

##### HTTPS的握手过程，如何防止中间人攻击？。
1. http是超文本传输协议，https是http+SSL/TLS，https的握手过程就是SSL/TLS的握手过程；
2. 以okhttp中https连接创建过程为例
	1. 建议TCP连接后，开始TLS握手
	2. 握手过程分为6步
		1. Client Hello: TLS版本+支持的加密套件+随机数
		2. Server Hello: 选择支持的TLS版本+加密套件、随机数+服务器的数字证书（包含公钥）
		3. 证书校验
		4. 密钥交换
			1. 客户端生成预主密钥，用服务器公钥加密发送给服务器
		5. 生成会话密钥用于后续通信的对称加密
			1. 客户端随机数
			2. 服务端随机数
			3. 预主密钥
		6. 完成握手：客户端与服务器互相发送Change Cipher Spec 和 Finished 消息。
3. 对客户端来说2个常见的操作来防止
	1.  验证证书时，增加域名匹配，检查证书的中域名与访问的域名是否一致
	2. 增加双向认证，强制客户端和服务端互相验证身份
#### **架构模式与设计**  
   ##### MVP与MVVM的对比，DataBinding如何实现双向绑定？  
   ##### 组件化通信方案（ARouter原理及拦截器实现）。
1. ARouter的实现原理分为2个部分
	1. APT技术，生成路由表

####  **依赖注入与框架**  
   - Hilt与Dagger2的差异，如何管理多模块的依赖关系？。

---

### **五、算法与数据结构**
2. **高频手撕代码**  
   - 合并两个有序链表（递归与非递归实现）。  
   - 二叉树层序遍历及变种（如Z字型遍历）。  
   - 动态规划问题（如最长回文子串）。

3. **复杂度与优化**  
   - HashMap扩容机制（为何容量为2的幂次方？）。  
   - 快排的时间复杂度及最坏情况优化策略。

---

### **六、项目与开放性问题**
4. **项目深度挖掘**  
   - 描述一个解决性能瓶颈的案例（如内存抖动优化、卡顿治理）。  
   - 如何设计一个支持高并发请求的图片加载框架（缓存策略、线程池管理）。

5. **开放性问题（比亚迪侧重）**  
   - 如何优化车载系统的稳定性（如LMKD低内存管理、Native层崩溃分析）？  
   - 对SOA架构的理解及在车载系统中的应用。

---

### **参考资料与延伸**
6. **大疆高频题**：HandlerThread原理、View绘制边界问题、APK加固方案。  
7. **比亚迪专项**：Framework层调试、Binder通信优化、QNX系统基础。  
8. **系统学习**：《Android车载操作系统开发揭秘》（涵盖EEA架构、智能座舱等）。

---

**建议准备策略**：  
- **底层原理**：结合源码（如AOSP）理解Binder、AMS等机制。  
- **车载开发**：熟悉Android Automotive OS及车载调试工具（如CANoe）。  
- **算法**：每日LeetCode练习，重点突破链表、树、动态规划。  
- **项目复盘**：梳理过往项目的技术难点与解决方案，量化优化结果（如性能提升百分比）。  

如需完整面试题库或车载开发资料，可参考。
## 腾讯

以下是为具有10年经验的候选人设计的腾讯高级Android开发工程师模拟面试题，结合了腾讯历年面试真题及大厂高频考点，覆盖技术深度、系统设计、性能优化、项目经验等方向，并附上考察重点和参考思路：

---

### **一、Android基础与原理**
1. **Android系统架构分层及核心组件作用**  
   - 详细描述Linux内核、系统运行时（ART/Dalvik）、应用框架层、应用层的核心职责，并举例说明应用框架层如何支撑上层开发。

2. **Activity生命周期与启动模式**  
   - 解释`onSaveInstanceState()`和`onRestoreInstanceState()`的调用时机，以及SingleTask模式下Activity栈管理逻辑。  
   - 场景题：Activity A通过SingleTask模式启动Activity B，描述生命周期变化及可能的Intent传递问题。

3. **Binder机制与进程通信**  
   - 从Linux内核角度解释Binder的MMAP原理，对比共享内存、Socket等IPC方式的优缺点。  
   - AIDL生成的Java类结构是什么？如何实现跨进程回调。

---

### **二、高级开发技术**
4. **Handler机制与线程通信**  
   - 解释MessageQueue的同步屏障（Sync Barrier）和异步消息的应用场景，如何实现消息优先级。  
   - 子线程创建Handler为何需要先调用`Looper.prepare()`？主线程Looper为何不会导致ANR？

5. **View渲染与事件分发**  
   - 分析`onMeasure()`中`MeasureSpec`的生成规则，为什么`wrap_content`在某些布局中会失效？如何自定义ViewGroup实现精准测量。  
   - 事件分发中`requestDisallowInterceptTouchEvent()`的作用，如何解决滑动冲突。

6. **Jetpack组件与架构模式**  
   - LiveData如何避免内存泄漏？粘性事件的产生原因及解决方案。  
   - 对比MVP与MVVM，ViewModel如何与Lifecycle组件协作实现数据持久化。

---

### **三、性能优化实战**
1. **内存泄漏与OOM排查**  
   - 场景题：LeakCanary检测到Activity因静态Handler引用泄漏，如何修复？分析`Handler`持有外部类导致泄漏的根本原因。  
   - Bitmap加载优化：RGB_565与ARGB_8888的适用场景，如何通过`BitmapRegionDecoder`实现大图分块加载。

2. **启动速度与卡顿优化**  
   - 冷启动阶段`Application`和首屏Activity的耗时瓶颈点，如何利用`Traceview`和`Systrace`定位问题。  
   - 解释Choreographer与VSYNC信号的关系，如何通过减少UI线程阻塞避免掉帧。

---

### **四、系统设计与源码解析**
3. **框架源码理解**  
   - OkHttp的拦截器链执行顺序，如何自定义拦截器实现全局签名加密？连接池复用机制如何避免TCP握手开销。  
   - Glide的三级缓存策略，LruCache与DiskLruCache的淘汰算法实现。

4. **组件化与模块化**  
    - ARouter的路由表生成原理，如何通过APT技术实现模块间通信。  
    - 实现一个支持动态加载的插件化框架，需Hook哪些系统方法？资源冲突如何解决。

---

### **五、项目经验与架构设计**
5. **复杂项目挑战**  
    - 描述一个你主导的性能优化案例，包括问题定位（如ANR或卡顿）、工具使用（Profiler/Systrace）、解决方案（算法/线程/UI优化），并量化优化效果。  
    - 在多团队协作的大型项目中，如何设计模块解耦方案？遇到过哪些组件化带来的问题。

6. **系统设计题**  
    - 设计一个支持千万级用户的即时通讯系统，需考虑消息可靠性、离线推送、跨平台同步及安全性（端到端加密）。  
    - 实现一个线程池，支持任务优先级和动态调整核心线程数，需考虑并发安全与资源回收策略。

---

### **六、算法与数据结构**
7. **手写算法**  
    - 实现二叉树中两个节点的最近公共祖先（LCA）。  
    - 一千万个QQ号快速查找：基于位图（BitMap）或布隆过滤器（Bloom Filter）的设计与冲突率计算。

---

### **参考准备建议**
- **源码深挖**：如Handler/Looper、Binder驱动、RecyclerView缓存机制。  
- **腾讯业务结合**：关注音视频（WebRTC/FFmpeg）、跨端（Flutter）、高并发场景（如微信消息队列）。  
- **软技能**：团队协作案例、技术决策冲突解决、新技术预研方法论。

---

以上问题需结合项目实际经历展开，重点体现技术深度、系统化思维和解决问题的能力。建议针对每类问题准备2-3个核心案例，并熟悉腾讯技术栈（如MMKV、X5内核等）。

# 介绍下协程
1. 结构化并发
	1. 通过层级化的作用域和父子关系管理协程的生命周期，控制任务执行，取消和清理。
	2. 目前来说 无论是层级作用域还是父子关系都是Job来实现
	3. LifecycleCoroutineScope/Super