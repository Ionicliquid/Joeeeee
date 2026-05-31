# Android 图形渲染管线深度解析

> 从 App 绘制到屏幕显示的完整链路：ThreadedRenderer → BLASTBufferQueue → SurfaceFlinger → Display

---

## 目录

1. [ThreadedRenderer 与 BLASTBufferQueue 如何协作完成页面绘制](#1-threadedrenderer-与-blastbufferqueue-如何协作完成页面绘制)
2. [SurfaceControl 与 BLASTBufferQueue 的关系](#2-surfacecontrol-与-blastbufferqueue-的关系)
3. [Buffer 与 SurfaceControl 的关系](#3-buffer-与-surfacecontrol-的关系)
4. [ThreadedRenderer.setFrameCallback 的回调触发时机](#4-threadedrenderersetframecallback-的回调触发时机)
5. [用户看到一帧还需要什么条件](#5-用户看到一帧还需要什么条件)
6. [WMS.finishDrawingWindow 与一帧数据显示的关系](#6-wmsfinishdrawingwindow-与一帧数据显示的关系)

---

## 1. ThreadedRenderer 与 BLASTBufferQueue 如何协作完成页面绘制

### 一句话概括

> **ThreadedRenderer 负责「画什么」和「何时画」，BLASTBufferQueue 负责「画在哪」和「如何提交」。两者通过 Surface（IGraphicBufferProducer 接口）衔接，RenderThread 产出 buffer → BLASTBufferQueue 接收并提交到 SurfaceFlinger。**

### 整体架构图

```
┌──────────────────────────────────────────────────────────┐
│                       App 进程                           │
│                                                          │
│  ┌──────────────┐     ┌──────────────────────────┐       │
│  │  UI Thread   │     │      RenderThread        │       │
│  │              │     │                          │       │
│  │ ViewRootImpl │────→│ ThreadedRenderer         │       │
│  │  .draw()     │     │  .syncAndDrawFrame()     │       │
│  │              │     │     ↓                    │       │
│  │              │     │  Skia/GPU 绘制           │       │
│  │              │     │     ↓                    │       │
│  │              │     │  EGL/Vulkan swapBuffers  │       │
│  └──────────────┘     └───────────┬──────────────┘       │
│                                   │                      │
│                          dequeue + queue                  │
│                                   │                      │
│                          ┌────────▼──────────┐           │
│                          │ BLASTBufferQueue  │           │
│                          │  ┌──────────────┐ │           │
│                          │  │BufferQueueProd│ │           │
│                          │  │ucer (内部)    │ │           │
│                          │  └──────┬───────┘ │           │
│                          │         │         │           │
│                          │  BufferQueueCore  │           │
│                          │         │         │           │
│                          │  ┌──────▼───────┐ │           │
│                          │  │ Transaction  │ │           │
│                          │  │  + merge     │ │           │
│                          │  └──────┬───────┘ │           │
│                          └─────────┼─────────┘           │
└────────────────────────────────────┼─────────────────────┘
                                     │ IPC (oneway)
                            ┌────────▼──────────┐
                            │  SurfaceFlinger   │
                            │  (BLASTConsumer)  │
                            │      ↓            │
                            │  合成 + Present    │
                            └───────────────────┘
```

### 五个阶段的详细流程

#### 阶段一：Surface 与 BLASTBufferQueue 的绑定

```
ViewRootImpl.relayoutWindow()
  → 从 WMS 获取新的 SurfaceControl
  → BLASTBufferQueue bbq = new BLASTBufferQueue("view", surfaceControl, w, h, format)
  → Surface surface = bbq.createSurface()  // 拿到 Producer 端 Surface
  → ThreadedRenderer.setSurface(surface)   // 传入渲染器
```

**关键点**：`BLASTBufferQueue.createSurface()` 返回的 `Surface` 内部持有 `BBQBufferQueueProducer`（即 IGraphicBufferProducer），ThreadedRenderer 通过这个 Surface 间接连接到 BLASTBufferQueue 的内部 BufferQueue。

`setSurface(surface)` 内部会：
1. 检测 Surface generation ID / 尺寸是否变化
2. 若变化，销毁旧 EGLSurface，基于新 Surface 重建 EGLSurface
3. 新 EGLSurface 的 `ANativeWindow` 指向的就是 BLASTBufferQueue 的 producer

#### 阶段二：VSync 触发绘制

```
Choreographer.doFrame()          // VSync 信号到达
  → ViewRootImpl.performTraversals()
    → measure → layout → draw
      → ViewRootImpl.draw()
        → ThreadedRenderer.draw(viewRootImpl)
          → 更新 DisplayList
          → nSyncAndDrawFrame()  // JNI → 同步 + 绘制
```

**`nSyncAndDrawFrame()` 是核心同步点**：
- **UI 线程**在此调用后进入 `wait()` 阻塞
- **RenderThread** 被唤醒，执行 GPU 绘制
- 绘制完成后，RenderThread 通知 UI 线程

#### 阶段三：RenderThread 产出 Buffer

在 RenderThread 中，`nSyncAndDrawFrame()` 的 native 实现做了以下工作：

```
RenderThread::drawFrame()
  → 从 BLASTBufferQueue 的 BufferQueue 中 dequeueBuffer()  // 获取一个空闲 buffer
  → 将 buffer 绑定为 EGL/Vulkan 的渲染目标
  → Skia 执行 DisplayList 重放 → GPU 绘制
  → eglSwapBuffers() / vkQueuePresentKHR()
      → 内部调用 queueBuffer() 将绘制完成的 buffer 归还给 BLASTBufferQueue
  → 发送 release 信号，唤醒 UI 线程
```

**核心衔接**：RenderThread 不直接知道 BLASTBufferQueue 的存在，它只操作 `ANativeWindow`（Surface）。而 Surface 的 dequeue/queue/lock 操作，实际转发给了 BLASTBufferQueue 内部的 BufferQueue。

#### 阶段四：BLASTBufferQueue 提交到 SurfaceFlinger

当 GPU 绘制完成、buffer 被 queueBuffer 后：

```
BufferQueueCore::queueBuffer()
  → 触发 BLASTBufferQueue 的 onBufferQueued() 回调
  → BLASTBufferQueue 构造一个 Transaction：
      transaction.setBuffer(surfaceControl, bufferHandle, frameNumber, releaseFence)
      transaction.setDesiredPresentTime(...)
      // 同时合并之前 mergeWithNextTransaction 的附加属性
  → nativeApplyTransaction(transaction)  // oneway IPC 提交给 SF
  → SurfaceFlinger::BLASTConsumer 接收
```

**`mergeWithNextTransaction` 的作用**：

```java
// ViewRootImpl 中，在 draw 之前将 WMS 属性变更与下一帧绑定
mBlastBufferQueue.mergeWithNextTransaction(wmTransaction, frameNumber);
```

这意味着窗口属性变更（如模糊、裁剪、位置）不会单独提交，而是等到该帧 buffer 就绪后**一起**发送给 SF。

#### 阶段五：SurfaceFlinger 合成上屏

```
SurfaceFlinger::onMessageReceived()
  → BLASTConsumer::acquireBuffer()  // 通过 Binder 接收 buffer handle
  → SF 在下一 VSync 周期进行图层合成
  → present → 屏幕显示
  → 释放 buffer，发 releaseFence 给 App
```

### 关键类的关系速查

| 类 | 角色 | 进程 |
|---|---|---|
| **ThreadedRenderer** | Java 层渲染器，管理 RenderThread 生命周期 | App |
| **RenderThread** | 执行 Skia + GPU 绘制 | App |
| **Surface** | ANativeWindow 封装，提供 dequeue/queue 接口 | App |
| **BLASTBufferQueue** | 管理 BufferQueue + Transaction 提交 | App |
| **BufferQueueCore** | buffer 状态机 (FREE → DEQUEUED → QUEUED → ACQUIRED) | App |
| **BLASTConsumer** | 接收 buffer handle，通知 SF 合成 | SurfaceFlinger |

### BLAST 路径相比旧路径的核心变化

```
旧路径 (Android 11-):
  App dequeueBuffer ──IPC──→ SF dequeueBuffer
  App queueBuffer   ──IPC──→ SF queueBuffer
  App Transaction   ──IPC──→ SF applyTransaction
  ❌ buffer 与 Transaction 分离，频繁 IPC

BLAST 路径 (Android 12+):
  App dequeueBuffer (本地，无 IPC)
  App queueBuffer  + mergeWithNextTransaction
    → 一帧一 Transaction ──单次 oneway IPC──→ SF
  ✅ buffer 与属性绑定提交，IPC 大幅减少
```

### 总结

**ThreadedRenderer 与 BLASTBufferQueue 的联系链条**：

1. **Surface** 是两者之间的桥梁 — ThreadedRenderer 持有 Surface，Surface 的 producer 端是 BLASTBufferQueue 的 internal producer
2. ThreadedRenderer 调用 `eglSwapBuffers` → 触发 Surface 的 `queueBuffer` → BLASTBufferQueue 收到 buffer 后组装 Transaction 发给 SF
3. BLASTBufferQueue 通过 `mergeWithNextTransaction` 将窗口属性变更与帧 buffer **绑定提交**，实现「一帧一 Transaction」
4. 整个过程在 App 进程内完成 buffer 管理，只有最终的 Transaction 通过 oneway IPC 发送到 SurfaceFlinger，大幅降低了跨进程开销

---

## 2. SurfaceControl 与 BLASTBufferQueue 的关系

### 一句话概括

> **SurfaceControl 是图层在 SurfaceFlinger 侧的「身份证」，BLASTBufferQueue 是 App 进程内负责「生产 buffer → 封装 Transaction → 送货上门」的管线引擎。两者通过 `bbq-wrapper` 子图层产生绑定关系。**

### 核心架构：一个隐藏的「bbq-wrapper」子图层

`SurfaceControl` 传入 BLASTBufferQueue 后，发生了什么？答案在 `SurfaceControl::generateSurfaceLocked()` 中：

```
Constructor 调用链:

BLASTBufferQueue(name, surfaceControl, w, h, format)
  └→ nativeUpdate(mNativeObject, sc.mNativeObject, w, h, format)
       └→ BLASTBufferQueue::update(surfaceControl, w, h, format)
            ├── 保存 surfaceControl 引用: mSurfaceControl = sc
            └── 更新尺寸/格式
```

但更关键的绑定发生在 `SurfaceControl` **第一次调用 `getSurface()`** 时：

```
SurfaceControl::getSurface()
  └→ generateSurfaceLocked()
       ├── ① 创建子图层 "bbq-wrapper"
       ├── ② 将子图层传给 BLASTBufferQueue
       └── ③ BLASTBufferQueue 产出 Surface 返回给调用方
```

### 详细步骤拆解

#### ① 创建「bbq-wrapper」子图层

```cpp
// frameworks/native/libs/gui/SurfaceControl.cpp
sp<Surface> SurfaceControl::generateSurfaceLocked()
{
    // 从父 SurfaceControl 继承关键属性
    auto flags = mCreateFlags & (ISurfaceComposerClient::eCursorWindow |
                                 ISurfaceComposerClient::eOpaque);

    // ★ 核心：以当前 SurfaceControl 为父节点，创建名为 "bbq-wrapper" 的子图层
    mBbqChild = mClient->createSurface(
        String8("bbq-wrapper"),  // 名称
        0, 0,                    // 初始宽高 (由 BLAST 动态更新)
        mFormat,                 // 像素格式 (继承自父)
        flags,                   // 属性标志 (继承自父)
        mHandle,                 // ★ 父图层句柄 → 建立父子层级
        {},                      // 额外参数
        &ignore                  // 忽略返回的生成 ID
    );
    ...
}
```

**关键点**：`mHandle` 是父 SurfaceControl 的句柄。`bbq-wrapper` 作为**子节点**挂载到父图层下，形成层级关系：

```
SurfaceFlinger 场景图:

parent SurfaceControl (例如 "com.example.app/ViewRootImpl")
  ├── 属性: position, crop, alpha, z-order...
  │
  └── "bbq-wrapper" (隐藏子图层)
        ├── 继承父的 transform / crop / z-order
        ├── ★ 真正承载 buffer 数据的图层
        └── 由 BLASTBufferQueue 管理其 buffer 提交
```

#### ② 将子图层传入 BLASTBufferQueue

```cpp
    // ② 用子图层创建 BLASTBufferQueue 适配器
    mBbq = sp<BLASTBufferQueue>::make(
        "bbq-adapter",  // 名称
        mBbqChild,      // ★ "bbq-wrapper" 的 SurfaceControl
        mWidth,         // 缓冲区宽度
        mHeight,        // 缓冲区高度
        mFormat         // 像素格式
    );
```

在 Java 层等价于：

```java
// ViewRootImpl 中 (简化示意)
BLASTBufferQueue bbq = new BLASTBufferQueue("view", surfaceControl, w, h, format);
// surfaceControl = 这个窗口的「主图层」
// bbq 内部会 createSurface 并持有对应的 native layer handle
```

BLASTBufferQueue 收到 `mBbqChild` 后：
- 将其保存为 `mSurfaceControl`
- 后续每次 `queueBuffer` 时，会用这个 SurfaceControl 构建 Transaction：

```cpp
// BLASTBufferQueue::acquireNextBufferLocked() 中
t.setBuffer(mSurfaceControl, buffer, releaseFence, frameNumber);
t.setDataspace(mSurfaceControl, dataspace);
t.setDesiredPresentTime(mSurfaceControl, timestamp);
```

#### ③ BLASTBufferQueue 产出 Surface

```cpp
    // ③ 从 BLASTBufferQueue 获取 Surface → 返回给 App 用于绘图
    mSurfaceData = mBbq->getSurface(true);
    return mSurfaceData;
}
```

`getSurface(true)` 内部：

```cpp
sp<Surface> BLASTBufferQueue::getSurface(bool includeSurfaceControlHandle) {
    // 创建 BBQSurface（继承自 Surface），持有:
    //   - mGraphicBufferProducer (BBQBufferQueueProducer)
    //   - mSurfaceControlHandle (bbq-wrapper 的 handle)
    return new BBQSurface(mProducer, includeSurfaceControlHandle, 
                         mSurfaceControl, this);
}
```

返回的 `BBQSurface` 就是 **App 的「画布」**，ThreadedRenderer 拿到它后进行 GPU 绘制。

### 完整关系图

```
                          ┌─────────────────────────────────────────┐
                          │         SurfaceFlinger 进程              │
                          │                                         │
    ┌─────────────────────┼─────────────────────────────────────┐   │
    │   Scene Graph       │                                     │   │
    │                     │                                     │   │
    │  Root               │                                     │   │
    │   ├── ...           │                                     │   │
    │   └── parent SC ◄───┼── 主图层 (mHandle)                   │   │
    │         │           │     - position, crop, z, alpha       │   │
    │         │           │     - 不直接承载 buffer               │   │
    │         │           │                                     │   │
    │         └── "bbq-wrapper" ◄── 子图层 (mBbqChild)           │   │
    │               │     │     - 继承父的几何属性                │   │
    │               │     │     - ★ 真正承载 buffer 数据          │   │
    │               │     │     - BLAST 向它 setBuffer()          │   │
    └───────────────┼─────┼─────────────────────────────────────┘   │
                    │     │                                         │
                    │     └── Transaction.setBuffer(bbq_child, ...) │
                    │                                               │
  ──────────────────┼── 进程边界 ─────────────────────────────────────
                    │
    ┌───────────────┼──────────────────────────────────────────┐
    │               │         App 进程                          │
    │               │                                          │
    │  ┌────────────▼──────────┐                               │
    │  │  SurfaceControl (父)  │  ← ViewRootImpl 持有的主 SC    │
    │  │  mHandle ─────────────┼── 对应 SF 侧的 parent SC       │
    │  └───────────┬───────────┘                               │
    │              │                                           │
    │              │ generateSurfaceLocked()                    │
    │              │                                           │
    │  ┌───────────▼───────────┐                               │
    │  │  BLASTBufferQueue     │                               │
    │  │                       │                               │
    │  │  mSurfaceControl ◄────┼── bbq-wrapper 的 SC           │
    │  │  mProducer ◄──────────┼── BBQBufferQueueProducer       │
    │  │  mConsumer ◄──────────┼── BufferQueueConsumer          │
    │  │                       │    (App 进程内部消费!)          │
    │  │  getSurface() ────────┼── 返回 BBQSurface              │
    │  └───────────┬───────────┘                               │
    │              │                                           │
    │  ┌───────────▼───────────┐                               │
    │  │  BBQSurface (Surface) │                               │
    │  │  mProducer ───────────┼── BLASTBufferQueue::mProducer  │
    │  │  mSurfaceControlHandle┼── bbq-wrapper 的 handle        │
    │  └───────────┬───────────┘                               │
    │              │                                           │
    │  ┌───────────▼───────────┐                               │
    │  │  ThreadedRenderer     │                               │
    │  │  setSurface(surface)  │  ← 拿着 BBQSurface 绘图       │
    │  │  GPU 绘制 → buffer    │                               │
    │  │  eglSwapBuffers()     │                               │
    │  └───────────────────────┘                               │
    └──────────────────────────────────────────────────────────┘
```

### 为什么需要「bbq-wrapper」这个中间子图层？

| 设计考虑 | 说明 |
|----------|------|
| **属性继承** | 子图层自动继承父图层的 transform、crop、alpha、z-order，不需要重复设置 |
| **职责分离** | 父 SC 负责「窗口属性」（位置、大小、层级），子 SC 负责「内容承载」（buffer 提交） |
| **子 Surface 支持** | 当 SurfaceView 等创建子 Surface 时，`bbq-wrapper` 的 handle 用于识别父节点 |
| **无缝替换** | 对上层 ThreadedRenderer 透明 — 它只看到 Surface，不知道下面是 BLAST 还是旧 BufferQueue |

### Transaction 提交时用的是哪个 SurfaceControl？

```cpp
// BLASTBufferQueue::acquireNextBufferLocked()
// 向 SF 提交 buffer 时，使用的是 bbq-wrapper 的 SurfaceControl:
t.setBuffer(mSurfaceControl,   // ← bbq-wrapper 的 SC，而非父 SC
            buffer, 
            releaseFence, 
            frameNumber);
```

而窗口属性变更（mergeWithNextTransaction）用的是**父 SurfaceControl**：

```java
// ViewRootImpl 中
Transaction t = new Transaction();
t.setPosition(parentSC, x, y);
t.setCrop(parentSC, rect);
bbq.mergeWithNextTransaction(t, frameNumber);
// → 属性发给 parent SC，buffer 发给 bbq-wrapper SC
```

### 总结

| 对象 | 角色 | 位置 |
|------|------|------|
| **父 SurfaceControl** | 窗口的「外壳」— 定义位置、层级、裁剪、透明度 | ViewRootImpl 持有 |
| **bbq-wrapper SC** | 窗口的「画框」— 承载 buffer 内容的子图层 | BLASTBufferQueue 内部持有 |
| **BLASTBufferQueue** | 「管线引擎」— dequeue/queue buffer，组装 Transaction，提交到 SF | App 进程内 |
| **BBQSurface** | 「画布」— ThreadedRenderer 的绘图目标 | ThreadedRenderer 持有 |

**本质关系**：`SurfaceControl` 是树上的节点，`BLASTBufferQueue` 是往节点上送数据的管道。它们通过 `bbq-wrapper` 这个子节点「对接」— BLAST 负责产出 buffer 并提交到子节点上，子节点继承父节点的几何属性，最终 SurfaceFlinger 在合成时将其作为父图层的内容进行渲染。

---

## 3. Buffer 与 SurfaceControl 的关系

### 一句话概括

> **SurfaceControl 定义「窗口的画框」— 画在哪、多大、第几层、多透明；Buffer 定义「画框里的内容」— 像素数据。两者的关联通过 `Transaction.setBuffer(SurfaceControl, buffer)` 建立，SurfaceFlinger 在合成时将 buffer 内容渲染到对应图层的画框内。**

### 本质类比

```
SurfaceControl = 电影院里的一块银幕
    - 位置 (挂在哪个厅、第几个)
    - 尺寸 (IMAX / 普通)
    - Z 序 (前排/后排)
    - 透明度
    - 裁剪区域

Buffer = 投射到银幕上的影片帧
    - 像素数据 (GraphicBuffer / HardwareBuffer)
    - 数据格式 (RGBA8888 / YUV / ...)
    - Fence (这一帧还没渲染完，等等再投)

Transaction.setBuffer(sc, buffer) = 把第 N 号胶片装到投影机上，对准这块银幕
```

### 关系的两种建立方式

#### 方式 A：通过 Surface / BufferQueue（传统隐式路径）

这是 ThreadedRenderer 使用的路径。Buffer 与 SurfaceControl 的关联是**自动的**：

```
App 进程                                       SurfaceFlinger 进程

Surface (IGraphicBufferProducer)
  │
  ├─ lockCanvas() / dequeueBuffer()
  │    └─ 从 BufferQueueCore 的 64 个 slot 中
  │       取出一个 FREE 的 GraphicBuffer
  │       State: FREE → DEQUEUED
  │
  ├─ GPU 绘图 (Skia / GL / Vulkan)
  │    └─ 像素数据写入 GraphicBuffer 的共享内存
  │
  └─ unlockCanvasAndPost() / eglSwapBuffers()
       └─ queueBuffer(slot, fence)
            State: DEQUEUED → QUEUED
            │
            └── Binder IPC ──→ Layer::onBufferQueued()
                                  │
                                  └─ 该 Layer 对应的 SurfaceControl
                                     就与这个 buffer 关联了
```

**关键**：这里的关联是**隐式的**。BufferQueue 本身就在某一个 Layer 的上下文中创建，queue 进去的 buffer 天然就属于那个 Layer，不需要手动指定 SurfaceControl。

#### 方式 B：通过 Transaction.setBuffer()（显式路径，API 33+）

这是 BLAST 路径使用的机制。Buffer 与 SurfaceControl 的关联是**显式的**：

```java
// 1. 创建 HardwareBuffer
HardwareBuffer buffer = HardwareBuffer.create(
    width, height, HardwareBuffer.RGBA_8888,
    HardwareBuffer.USAGE_GPU_SAMPLED_IMAGE | HardwareBuffer.USAGE_COMPOSER_OVERLAY
);

// 2. 显式绑定到指定的 SurfaceControl
SurfaceControl.Transaction t = new SurfaceControl.Transaction();
t.setBuffer(surfaceControl, buffer, fence, () -> {
    // buffer 被后续 frame 替换时回调，可以复用
});
t.apply();
```

在 BLASTBufferQueue 的 native 实现中：

```cpp
// BLASTBufferQueue::acquireNextBufferLocked()
void BLASTBufferQueue::acquireNextBufferLocked(...) {
    // 获取绘制完成的 buffer
    BufferItem item;
    mBufferItemConsumer->acquireBuffer(&item, 0, false);

    // 构造 Transaction，显式将 buffer 绑定到 mSurfaceControl
    SurfaceComposerClient::Transaction t;
    t.setBuffer(mSurfaceControl,      // ← bbq-wrapper 的 SurfaceControl
                item.mGraphicBuffer,  // ← 绘制完成的 buffer
                item.mFence,          // ← acquire fence
                frameNumber);         // ← 帧序号
    t.setDataspace(mSurfaceControl, item.mDataSpace);
    t.setDesiredPresentTime(mSurfaceControl, timestamp);

    // 提交
    t.apply();
}
```

### 数据结构层面的关系

```
┌──────────────────────────────────────────────────────────┐
│                 SurfaceFlinger 进程中                      │
│                                                          │
│  Layer (每个 SurfaceControl 对应一个 Layer)                │
│  ┌────────────────────────────────────────────────────┐  │
│  │                                                    │  │
│  │  State (来自 SurfaceControl 的属性):                 │  │
│  │  ├── position (x, y)     ← setPosition(sc, x, y)   │  │
│  │  ├── size (w, h)         ← setBufferSize(sc, w, h) │  │
│  │  ├── layer (z-order)     ← setLayer(sc, z)         │  │
│  │  ├── alpha               ← setAlpha(sc, a)         │  │
│  │  ├── crop                ← setCrop(sc, rect)       │  │
│  │  ├── transform           ← setTransform(sc, mtx)   │  │
│  │  ├── visible             ← show() / hide()         │  │
│  │  └── colorTransform      ← setColorTransform(...)  │  │
│  │                                                    │  │
│  │  Content (来自 Buffer):                             │  │
│  │  ┌──────────────────────────────────────────────┐  │  │
│  │  │  BufferQueueCore                             │  │  │
│  │  │  BufferSlot mSlots[64]                       │  │  │
│  │  │  ┌──────┬──────┬──────┬──────┬─────┬──────┐  │  │  │
│  │  │  │Slot 0│Slot 1│Slot 2│Slot 3│ ... │Slot63│  │  │  │
│  │  │  │      │      │      │      │     │      │  │  │  │
│  │  │  │FREE  │QUEUED│DEQ'D │FREE  │ ... │FREE  │  │  │  │
│  │  │  │      │  ↓   │      │      │     │      │  │  │  │
│  │  │  │      │★当前 │      │      │     │      │  │  │  │
│  │  │  │      │显示的│      │      │     │      │  │  │  │
│  │  │  │      │buffer│      │      │     │      │  │  │  │
│  │  │  └──────┴──────┴──────┴──────┴─────┴──────┘  │  │  │
│  │  └──────────────────────────────────────────────┘  │  │
│  │                                                    │  │
│  │  合成时: Layer.position + Layer.crop + Buffer.像素   │  │
│  │          = 屏幕上该窗口的最终图像                     │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### Buffer 的生命周期状态机

```
            dequeueBuffer()          queueBuffer()
    FREE ────────────────→ DEQUEUED ─────────────→ QUEUED
      ↑                                              │
      │                                              │
      └──────────────── FREE ←──────────────────────┘
                  releaseBuffer()            acquireBuffer()
                                              │
                                              ▼
                                          ACQUIRED
                                          (SF 正在持有)
```

| 状态 | 含义 | 谁持有 |
|------|------|--------|
| **FREE** | 空闲，可被 dequeue | 无人 |
| **DEQUEUED** | App 已取出，正在绘制 | App (Producer) |
| **QUEUED** | App 画完已放回，等待 SF 取走 | BufferQueue |
| **ACQUIRED** | SF 已取走，正在合成/显示 | SurfaceFlinger (Consumer) |

**每个 slot 中的 mGraphicBuffer 指针**，在首次使用时由 gralloc 分配，之后持续复用（不释放），避免频繁分配/释放带来的开销。

### 三种路径的对比

```
路径 1: 传统 Surface 绘制 (隐式绑定)
─────────────────────────────────────
Surface.lockCanvas() ─→ dequeue ─→ 绘制 ─→ Surface.unlockCanvasAndPost() ─→ queue
                                                                               │
                                               SurfaceFlinger 自动将 buffer ───┘
                                               与对应 Layer 关联

路径 2: BLASTBufferQueue (隐式绑定 + 显式 Transaction)
─────────────────────────────────────────────────────
eglSwapBuffers() ─→ queueBuffer ─→ BLASTBufferQueue::acquireNextBufferLocked()
                                        │
                                        └─ Transaction.setBuffer(mSurfaceControl, buffer)
                                           .apply() ─→ SF 显式绑定

路径 3: 直接 setBuffer (显式绑定, API 33+)
─────────────────────────────────────────
HardwareBuffer.create()
    └─ Transaction.setBuffer(surfaceControl, buffer).apply()
       └→ SF 收到后直接将该 buffer 设置为 Layer 的当前内容
          跳过 BufferQueue 的 dequeue/queue 循环
```

### 总结

| 维度 | SurfaceControl | Buffer |
|------|---------------|--------|
| **是什么** | Layer 的句柄 + 元数据 | 像素数据的容器 |
| **存什么** | position, size, z, alpha, crop, transform | RGBA 像素 / YUV 数据 |
| **谁创建** | WindowManager / SurfaceComposerClient | gralloc HAL |
| **生命周期** | 窗口存活期间 | dequeue → draw → queue → acquire → release 循环 |
| **关联方式** | — | 通过 `setBuffer(sc, buf)` 绑定到 SurfaceControl |

**一句话**：SurfaceControl 是「银幕的位置和属性」，Buffer 是「投射到银幕上的画面」。没有 SurfaceControl，buffer 无处安放；没有 Buffer，SurfaceControl 只是一块空白。两者通过 `Transaction.setBuffer()` 建立绑定，SurfaceFlinger 在合成时将 buffer 像素按照 SurfaceControl 定义的几何属性渲染到屏幕。

---

## 4. ThreadedRenderer.setFrameCallback 的回调触发时机

### API 名称的演变

`ThreadedRenderer` 是 `HardwareRenderer` 的子类（Android API 24 引入），两者概念重叠。`setFrameCallback` 在不同版本中对应不同的 API：

| API 版本 | 类 | 方法 |
|----------|-----|------|
| Android 13+ (API 33+) | `HardwareRenderer.FrameRenderRequest` | `setFrameCommitCallback(executor, callback)` |
| Android 内部 | `HardwareRenderer.FrameRenderRequest` | `setFrameCompleteCallback(frameNr -> ...)` |

**标准用法**：

```java
HardwareRenderer renderer = ...;
renderer.createRenderRequest()
    .setFrameCommitCallback(executor, () -> {
        // 回调在此触发
    })
    .syncAndDraw();
```

### 核心结论：回调在什么时刻触发？

> **回调在 RenderThread 完成 GPU 绘制、buffer 被 `queueBuffer` 提交到 swap chain（即 BLASTBufferQueue）之后触发。但此时该帧可能还没有被 SurfaceFlinger 合成上屏。**

把这个时刻放到整个渲染管线中：

```
UI Thread                              RenderThread                     SurfaceFlinger
    │                                       │                                │
    │ createRenderRequest()                 │                                │
    │ .setFrameCommitCallback(r)            │                                │
    │ .syncAndDraw() ─────post task────→    │                                │
    │                                       │                                │
    │ syncFrameState() ←─────────────→     │ 同步 View 状态                  │
    │ ◄── unblockUiThread ──────────       │                                │
    │ (UI 线程继续)                          │                                │
    │                                       │ context->draw()               │
    │                                       │   ├── Skia 重放 DisplayList    │
    │                                       │   ├── GPU 渲染                 │
    │                                       │   └── eglSwapBuffers()         │
    │                                       │        └── queueBuffer()       │
    │                                       │             └── BLAST TX 提交   │
    │                                       │                                │
    │                                       │ ★ setFrameCommitCallback 触发! │
    │                                       │                                │
    │                                       │                      ──IPC──→ │ BLASTConsumer
    │                                       │                                │ acquireBuffer()
    │                                       │                                │ ...等 VSync...
    │                                       │                                │ composite()
    │                                       │                                │ present() ──→ 屏幕
```

### 源码级确认：回调挂载和触发点

#### 挂载点：Java 层

```java
// HardwareRenderer.java
public FrameRenderRequest createRenderRequest() {
    return new FrameRenderRequest();
}

public class FrameRenderRequest {
    public @NonNull FrameRenderRequest setFrameCommitCallback(
            @NonNull Executor executor,
            @NonNull Runnable frameCommitCallback) {
        // 内部调用 setFrameCompleteCallback
        setFrameCompleteCallback(frameNr -> executor.execute(frameCommitCallback));
        return this;
    }
}
```

#### 触发点：Native RenderThread 层

```
DrawFrameTask::run() [RenderThread]
    │
    ├── syncFrameState()           // 从 UI 线程同步状态
    ├── unblockUiThread()          // 释放 UI 线程
    │
    ├── context->draw()            // ★ GPU 绘制 + eglSwapBuffers
    │       │
    │       └── 绘制完成后，buffer 被 queue 到 BLASTBufferQueue
    │           BLASTBufferQueue 组装 Transaction → oneway IPC 到 SF
    │
    └── frameCompleteCallback()    // ★★★ 回调在此触发!
            │
            └── 执行你注册的 Runnable
```

**关键时间点**：回调发生在 `context->draw()` 执行完毕之后，即 GPU 绘制完成且 buffer 已经提交到 BLASTBufferQueue 之后。但 SurfaceFlinger 还没有进行合成。

### 为什么是这个时机？

回调研设计成在这个时刻触发，因为它是一个**精确的分水岭**：

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  ◄── 不安全区间 ──►│◄── setFrameCommitCallback ──►│             │
│                     │       触发时刻                │             │
│  buffer 正在被       │ buffer 已提交               │  SF 读取     │
│  GPU 绘制           │ (可安全读取)                 │  buffer      │
│  (不可读取)          │                             │  合成        │
│                                                                 │
│  PixelCopy 不能用    │ PixelCopy 可以用             │             │
│  读到半成品          │ 读到完整帧                   │             │
└─────────────────────────────────────────────────────────────────┘
```

**典型使用场景**：与 `PixelCopy` 配合截取渲染结果：

```java
renderer.createRenderRequest()
    .setFrameCommitCallback(backgroundExecutor, () -> {
        // ★ buffer 已提交，可以安全复制像素
        Bitmap dest = Bitmap.createBitmap(w, h, Bitmap.Config.ARGB_8888);
        PixelCopy.request(surfaceView, dest, copyResult -> {
            if (copyResult == PixelCopy.SUCCESS) {
                // 拿到截图
            }
        }, handler);
    })
    .syncAndDraw();
```

### 与 Choreographer FrameCallback 的对比

这是一个常见的混淆点，两者名称相似但完全不同：

| 维度 | `Choreographer.FrameCallback` | `setFrameCommitCallback` |
|------|------------------------------|--------------------------|
| **运行线程** | UI 线程 (Main Thread) | RenderThread (或自定义 Executor) |
| **触发时机** | VSync 信号到达，即将开始 `doFrame()` | GPU 绘制完毕，buffer 已提交 swap chain |
| **用途** | 调度下一帧的 UI 更新 | 截图、帧耗时统计、buffer 复用 |
| **阻塞影响** | 阻塞会掉帧 | 阻塞会卡 RenderThread |
| **典型场景** | 动画、滚动 | PixelCopy、性能监控 |

```
VSync
  │
  ├─→ Choreographer.FrameCallback.doFrame()    ← UI 线程
  │     └─→ ViewRootImpl.performTraversals()
  │           └─→ ThreadedRenderer.draw()
  │                 └─→ syncAndDraw()
  │                       └─→ [RenderThread 绘制...]
  │                              │
  │                              └─→ setFrameCommitCallback 触发  ← RenderThread
  │
  └─→ 下一个 VSync ──→ Choreographer.FrameCallback.doFrame() ...
```

### 重要注意事项

1. **不要在回调中做耗时操作** — RenderThread 高度敏感，官方强烈建议使用不同的 `Executor` 投递到其他线程
2. **FrameRenderRequest 不能跨帧持有** — 它不是线程安全的，每次请求都要新建
3. **回调可能永远不触发** — 如果帧被推迟到后续 VSync，已注册的 callback 不会执行
4. **回调不保证帧已上屏** — buffer 只是提交到了 swap chain/SurfaceFlinger，实际显示还要等 VSync + 合成 + Present

### 总结

```
setFrameCommitCallback 的触发时刻：

    GPU 绘制完成
      → eglSwapBuffers / queueBuffer
        → buffer 提交到 BLASTBufferQueue
          → BLAST 组装 Transaction → oneway IPC 到 SF
            → ★ 回调触发 ★
              → (异步) SF 收到 buffer
                → 下一个 VSync
                  → composite
                    → present → 屏幕可见
```

它是「App 侧绘制工作已完成」的信号 — GPU 已经画完了，buffer 已经发出去了。但它不是「用户已经看到这一帧」的信号 — 那要到 SF 合成并 Present 之后。

---

## 5. 用户看到一帧还需要什么条件

### 一句话概括

> **buffer 提交到 SF 只是「快递到了仓库」，用户真正看到画面还需要：VSync 开门 → 合成装车 → Present 发车 → Display 送到家，至少还要等 1~2 个 VSync 周期。**

### 完整的「看到一帧」流程图

```
setFrameCommitCallback 触发
      │
      │  buffer 已提交给 SF，但用户还看不到
      │
      ▼
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  ① VSync 唤醒                                                        │
│     SF 在 MessageQueue 中 sleep，等待 SF_VSYNC 信号                    │
│                                                                     │
│  ② Latch Buffer (拴住 buffer)                                        │
│     遍历每个 Layer，从 BufferQueue 取出最新 QUEUED 的 buffer            │
│     ★ 必须等待 acquireFence signal → 确保 GPU 已画完                   │
│                                                                     │
│  ③ Composite (合成)                                                  │
│     决定所有可见 Layer 如何叠在一起：                                   │
│     ├── HWC 硬件合成 (Overlay) → 零拷贝，最省电                         │
│     └── GPU Client 合成 (RenderEngine) → 需要效果时降级                │
│                                                                     │
│  ④ Present (提交到显示硬件)                                            │
│     HWC::presentDisplay() → 将合成结果传给 display driver              │
│                                                                     │
│  ⑤ Present Fence 信号化                                              │
│     显示硬件完成扫描输出 → fence signal → ★ 用户真正看到了               │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 逐个条件详解

#### 条件 ①：VSync 信号 — 等「下一班车」

SurfaceFlinger 不会「随叫随到」。它有自己的 VSync 节奏：

```
HW_VSYNC 时间线 (60Hz, 每 16.67ms 一次):

      VSync 0              VSync 1              VSync 2
    ────┬───────────────────┬───────────────────┬──────────→
        │                   │                   │
        │  App buffer       │  SF 合成          │  显示到屏幕
        │  到达 SF           │  composite        │  present
        │                   │                   │
        │  ←── 等待 ──→    │                   │
        │  (buffer 到了但   │                   │
        │   VSync 还没来)   │                   │
```

```
消息驱动机制:

SurfaceFlinger::onMessageReceived()
  └─ 收到 BLAST Transaction (buffer + 属性)
       └─ 设置 Layer 的 dirty 标记
            └─ 请求下一个 VSync: mScheduler->requestNextVsync()
                 └─ MessageQueue::setVsyncEnabled(true)
                      └─ 等待 HWC 的 VSync 回调唤醒...
```

**条件**：SF 必须等到下一个 `SF_VSYNC` 信号才能开始工作。如果刚好错过，最多等 16.67ms（60Hz）。

#### 条件 ②：Latch Buffer — 从队列中「取件」

VSync 到达后，SF 遍历每个 Layer，执行 `latchBuffer`：

```cpp
// SurfaceFlinger::handlePageFlip()
for (auto& layer : mLayers) {
    // 从 BufferQueue 取出最新的 QUEUED buffer
    status_t result = layer->latchBuffer(&r, &latchedFence);
    
    if (result == OK) {
        // ★ 必须等待 acquireFence signal
        // 保证 GPU 已经把内容渲染到 buffer 中
        addFence(latchedFence);
        
        // 如果没有新 buffer (result == NO_BUFFER_AVAILABLE)
        // 继续使用上一帧 latch 的旧 buffer → 画面静止，不会黑屏
    }
}
```

| 情况 | 行为 |
|------|------|
| 有新 QUEUED buffer，acquireFence 已 signal | ✅ 立即 latch |
| 有新 QUEUED buffer，acquireFence 未 signal | ⏳ **阻塞等待** fence signal |
| 无新 buffer | 复用上一帧的旧 buffer（画面静止） |

**条件**：新 buffer 的 acquireFence 必须 signal，否则 SF 会等。如果一直没有新 buffer，不会黑屏 — SF 会用旧 buffer 继续显示。

#### 条件 ③：Composite — 把所有图层「叠起来」

SF 拿到所有 Layer 的 buffer 后，需要把它们合成到一起：

```
                 ┌──────────────────────────────┐
    StatusBar     │  ████████████████████████    │  Layer Z=3
                 ┌┤                              │
    App Window   ││  ┌──────────────────────┐   │  Layer Z=2
                ││  │                      │   │
                ││  │   你的应用内容         │   │
                ││  │                      │   │
                ││  └──────────────────────┘   │
    Wallpaper   ││                              │  Layer Z=1
                └┤                              │
                 └──────────────────────────────┘
                          │
                          ▼  Composite
                          │
                 ┌────────────────┐
                 │  合成结果 buffer │
                 └────────────────┘
```

**合成策略由 HWC (Hardware Composer) 决定**：

```
validateComposition(layers):
    for each layer:
        if HWC 有可用的 overlay 平面
           and layer 不需要 GPU 处理 (裁剪/混合/圆角):
            → HWC Overlay 合成 (DEVICE)
            → 每个 layer 独占一个硬件平面
            → ★ 零拷贝，最省电
        else:
            → GPU Client 合成 (CLIENT)
            → RenderEngine (GLES) 把多层渲染到一块 buffer
            → 产生额外的 GPU 耗时

    if 任何 HWC 层验证失败:
        → 全部降级为 GPU Client 合成
```

| 合成方式 | 原理 | 延迟 | 功耗 |
|----------|------|------|------|
| **HWC Overlay** | 硬件平面直接叠放，显示控制器合并 | 最低 | 最低 |
| **GPU Client** | RenderEngine GLES 渲染 | +1~5ms | 较高 |

**条件**：合成策略必须确定。如果 HWC 资源不足（太多 Layer 争抢有限的 overlay 平面），部分 Layer 降级为 GPU 合成。如果 GPU 合成也失败 → **掉帧**。

#### 条件 ④：Present — 推送到显示硬件

合成完成后，通过 `presentDisplay()` 提交到显示驱动：

```cpp
// HWC::presentDisplay()
HWC2::Error HardwareComposer::presentDisplay(displayId) {
    // 将所有 layer 的 buffer handle、位置、z-order 等参数
    // 封装后传给 Kernel DRM (Direct Rendering Manager)
    // 
    // DRM 执行 Atomic Commit:
    //   - 原子性地切换所有 plane 的 buffer
    //   - 原子性地更新 CRTC 配置
    //   - 保证所有 layer 在同一帧切换，不会撕裂
    
    mHwcDevice->presentDisplay(displayId, &presentFence);
    
    return presentFence;  // ← 返回 presentFence
}
```

```
Kernel 空间:

DRM Atomic Commit
  ├── Primary Plane:  → App Buffer (或合成结果)
  ├── Overlay Plane 1: → StatusBar Buffer
  ├── Overlay Plane 2: → NavigationBar Buffer
  └── CRTC:           → 配置分辨率、刷新率、时序
      │
      └── VSync 到来时 → 硬件扫描输出 → 屏幕
```

**条件**：DRM Atomic Commit 必须成功。如果 Commit 失败（参数非法、带宽不足等） → 帧无法提交。

#### 条件 ⑤：Present Fence 信号化 → 用户真正看到

这是**最终条件**。Present 提交后，显示硬件并不会立刻显示 — 它要等到自己的扫描周期：

```
Present 提交 (此时 presentFence 未 signal)
      │
      ▼
  ...显示硬件继续扫描当前帧...
      │
      ▼
  下一个 HW_VSYNC ──→ 显示硬件切换到新帧
      │                  │
      │                  └── 开始从上到下逐行扫描新帧的像素
      │                       └── 扫描完成 → presentFence signal
      │                                        │
      │                                        └── ★ 用户看到了!
      ▼
  此时下一帧已经在合成中...
```

```
屏幕扫描过程 (以 60Hz 面板为例):

时间 0ms:        VSync ─ 开始扫描帧 A 的第 1 行
                  │
时间 8ms:        扫描到第 540 行 (屏幕中间)
                  │
时间 16.67ms:    扫描完成帧 A 的最后一行
                  │
                 VSync ─ 开始扫描帧 B
                  │
                 presentFence(帧 A) signal ✓
```

**条件**：显示硬件完成一次完整的扫描周期。面板刷新率决定了这个时间 — 60Hz 需要 16.67ms，120Hz 需要 8.33ms。

### 完整时间线总结

```
  时刻             事件                                   用户能看到吗?
  ────             ────                                  ─────────────
  T+0ms            App 开始绘制                            ❌
  T+Nms            eglSwapBuffers → queueBuffer            ❌
  T+Nms            setFrameCommitCallback 触发              ❌
  T+Nms            BLAST Transaction → SF 收到              ❌
  ...              
  T+16ms (±)       SF_VSYNC 到达                           ❌
  T+16ms (+1ms)    latchBuffer (等 acquireFence)           ❌
  T+16ms (+2ms)    composite (HWC/GPU)                     ❌
  T+16ms (+3ms)    present → DRM Atomic Commit             ❌
  ...              
  T+32ms (±)       下一个 HW_VSYNC                          ❌
                   Display 开始扫描新帧
                   ...
  T+32ms + ~8ms    扫描过半 (面板中线)                       👁 屏幕上半部分看到了
  T+32ms + 16.67ms 扫描完成                                 👁 ✅ 用户完全看到!
                   presentFence signal ✓
```

### 典型延迟

| 刷新率 | 最短延迟 (有 VSync offset) | 典型延迟 |
|--------|---------------------------|----------|
| 60Hz | ~2 帧 ≈ 33ms | 2~3 帧 |
| 90Hz | ~2 帧 ≈ 22ms | 2~3 帧 |
| 120Hz | ~2 帧 ≈ 17ms | 2~3 帧 |

### 可能导致「看不到」的异常情况

| 问题 | 原因 | 症状 |
|------|------|------|
| **掉帧** | buffer 未就绪 (acquireFence 超时) | 重复显示上一帧 |
| **HWC 验证失败** | overlay 平面不足 | 全部降级 GPU 合成，可能超时 |
| **DRM commit 失败** | 参数非法 / 带宽不足 | 帧丢失，fence 永不 signal |
| **被遮挡** | Layer Z 序在另一个不透明窗口后面 | SF 跳过该层，合成资源节省 |
| **被裁剪** | Layer Crop 区域为零 | 内容被剪掉 |
| **Alpha = 0** | 完全透明 | 用户看不到（但合成仍然发生） |

### 一句话总结

```
★ 用户看到一帧 = 5 个条件全部满足 ★

buffer 到 SF → VSync 唤醒 → Latch + fence done → 合成成功 → Present 成功 → 显示面板扫描完毕

任何一个条件不满足，用户看到的要么是旧帧，要么是黑屏/白屏。
```

---

## 6. WMS.finishDrawingWindow 与一帧数据显示的关系

### 一句话概括

> **`finishDrawingWindow` 不是让 buffer 上屏，而是让 WMS「批准」窗口可见。它是一座桥 — 连接「App 画完了第一帧」和「WMS 允许这个窗口被显示」。只有首帧需要这座桥，日常帧直接通过 BLASTBufferQueue 自己上屏。**

### 核心区分：两个独立的概念

很多人的误解在于把「buffer 到了 SF」等同于「窗口可以显示了」。实际上这是两条独立的控制线：

```
 控制线 ①：Buffer 内容
 ┌─────────────────────────────────────────────────┐
 │ App 画了什么 → BLASTBufferQueue → SF 拿到 pixel │
 │ (每帧都走这条线)                                 │
 └─────────────────────────────────────────────────┘

 控制线 ②：窗口可见性
 ┌─────────────────────────────────────────────────┐
 │ WMS 批准显示 → Transaction.show() → SF 开始合成  │
 │ (只有首帧 / 窗口状态变化时才走这条线)              │
 └─────────────────────────────────────────────────┘
```

**`finishDrawingWindow` 属于控制线 ②**。

### finishDrawingWindow 在整体流程中的位置

```
App 进程                                    system_server (WMS)            SurfaceFlinger
   │                                             │                             │
   │ ① addWindow ─────────────────────────────→ │ 创建 WindowState              │
   │                                             │ mDrawState = NO_SURFACE       │
   │                                             │                              │
   │ ② relayoutWindow ─────────────────────────→ │ 创建 SurfaceControl+Surface   │
   │    返回 Surface 给 App                       │ mDrawState = DRAW_PENDING     │
   │                                             │                              │
   │ ③ ThreadedRenderer.setSurface(surface)      │                              │
   │                                             │                              │
   │ ④ performDraw()                             │                              │
   │    DisplayList → GPU → queueBuffer          │                              │
   │    → BLAST tx → SF 已收到 buffer            │   ★ SF 有 buffer 了,          │
   │                                             │     但窗口还是 HIDDEN!        │
   │                                             │                              │
   │ ⑤ reportDrawFinished()                      │                              │
   │    └─ finishDrawing ──────────────────────→ │ finishDrawingWindow()        │
   │                                             │   DRAW_PENDING                │
   │                                             │     → COMMIT_DRAW_PENDING     │
   │                                             │                              │
   │                                             │ ⑥ performSurfacePlacement()   │
   │                                             │   commitFinishDrawingLocked() │
   │                                             │     COMMIT_DRAW_PENDING       │
   │                                             │       → READY_TO_SHOW         │
   │                                             │   performShowLocked()         │
   │                                             │     READY_TO_SHOW             │
   │                                             │       → HAS_DRAWN             │
   │                                             │                              │
   │                                             │   Transaction.show(sc) ────→ │ ★ 现在允许合成了!
   │                                             │   .apply()                   │
   │                                             │                              │
   │                                             │                       ┌──────┴──────┐
   │                                             │                       │ VSync →     │
   │                                             │                       │ composite → │
   │                                             │                       │ present →   │
   │                                             │                       │ 屏幕显示    │
   │                                             │                       └─────────────┘
```

### 关键点 ①：Buffer 到了 SF ≠ 窗口可见

在 `finishDrawingWindow` 之前，状态是这样的：

```
时间点：relayoutWindow 返回后，App 绘制完成，buffer 已 queue 到 BLASTBufferQueue

┌─────────────────────────────────────────────────────┐
│                                                     │
│  BLASTBufferQueue:  buffer ✅ 已提交                 │
│  SF BufferQueue:     buffer ✅ 已 QUEUED              │
│  WMS WindowState:    mDrawState = DRAW_PENDING       │
│  SF Layer:           visible = false ❌               │
│                                                     │
│  结果：SF 有 buffer 但 Layer 被标记为不可见            │
│        合成时跳过该 Layer                             │
│        用户看不到                                     │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 关键点 ②：mDrawState 状态机

`finishDrawingWindow` 的核心作用是推进 `WindowStateAnimator.mDrawState`：

```
   NO_SURFACE          DRAW_PENDING         COMMIT_DRAW_PENDING      READY_TO_SHOW         HAS_DRAWN
  ┌──────────┐       ┌──────────────┐      ┌───────────────────┐    ┌──────────────┐      ┌──────────┐
  │ 没有     │──①──→│ Surface 已创建│──②──→│ App 报告绘制完成   │─③─→│ 已 commit    │─④──→│ 已显示   │
  │ Surface  │       │ 等待 App 绘制 │      │ 等待 WMS commit   │    │ 等待 show    │      │          │
  └──────────┘       └──────────────┘      └───────────────────┘    └──────────────┘      └──────────┘
       ↑                                                                                       │
       │                   ① relayoutWindow                                                    │
       │                   ② finishDrawingWindow ─────── 核心!                                  │
       │                   ③ commitFinishDrawingLocked (在 performSurfacePlacement 中)          │
       │                   ④ performShowLocked ──→ Transaction.show()                          │
       │                                                                                       │
       └───────────────────────────────────────────────────────────────────────────────────────┘
                              窗口销毁时回到 NO_SURFACE
```

每一步的触发者：

| 步骤 | 触发者 | 谁调用 | 做什么 |
|------|--------|--------|--------|
| ① | `relayoutWindow` | ViewRootImpl → WMS | 创建 Surface，设为 `DRAW_PENDING` |
| ② | **`finishDrawingWindow`** | **ViewRootImpl.reportDrawFinished() → WMS** | **上报首帧完成，设为 `COMMIT_DRAW_PENDING`** |
| ③ | `commitFinishDrawingLocked` | WMS 内部 (performSurfacePlacement) | commit，设为 `READY_TO_SHOW` |
| ④ | `performShowLocked` | WMS 内部 (performSurfacePlacement) | `Transaction.show()`，设为 `HAS_DRAWN` |

### 关键点 ③：finishDrawingWindow 不是每帧都调

```java
// ViewRootImpl.java
private void performDraw() {
    // ...
    boolean canSkipDraw = ...;
    
    if (mReportNextDraw) {  // ← 只有这个 flag = true 时才上报
        mReportNextDraw = false;
        
        // 注册回调，绘制完成后调用 reportDrawFinished()
        registerFrameCommitCallback(() -> {
            mWindowSession.finishDrawing(mWindow, postDrawTransaction, seqId);
            //                        ↓
            //                  WMS.finishDrawingWindow()
        });
    }
}
```

`mReportNextDraw` 在以下情况被设为 `true`：
- 首次绘制
- `invalidate()` 之后调用 `reportNextDraw()`
- Window 状态变化（如从不可见变为可见）

**日常滚动/动画的每一帧，不经过 `finishDrawingWindow`。** 那些帧的 buffer 通过 BLASTBufferQueue 直接提交到 SF，SF 直接合成，因为 Layer 已经是 `HAS_DRAWN` + `visible` 状态。

### 完整对比：首帧 vs 日常帧

```
                        首帧 (需要 finishDrawingWindow)              日常帧 (不需要)
                        ──────────────────────────────              ─────────────────

Buffer 产出             ThreadedRenderer.draw()                    ThreadedRenderer.draw()
                           ↓                                          ↓
Buffer 提交到 SF         BLASTBufferQueue queueBuffer → SF          BLASTBufferQueue queueBuffer → SF
                           ↓                                          ↓
WMS 感知                reportDrawFinished()                        ❌ 不经过 WMS
                           ↓
                        finishDrawingWindow()
                           ↓
                        DRAW_PENDING → COMMIT_DRAW_PENDING
                           ↓
                        performSurfacePlacement()
                           ↓
                        Transaction.show(sc) → Layer visible=true
                           ↓                                          ↓
SF 合成                 ✅ 合成                                     ✅ 直接合成 (Layer 已 visible)
                           ↓                                          ↓
用户看到                ✅ 首帧上屏                                 ✅ 新帧上屏
```

### 如果 finishDrawingWindow 不调用会怎样？

```
典型场景：冷启动

relayoutWindow 完成 → Surface 已创建 → DRAW_PENDING
App 开始绘制 → buffer 已经到了 SF
但是...
finishDrawingWindow 没有被调用 → mDrawState 停留在 DRAW_PENDING
                                → Layer 永远不可见
                                → 用户看到白屏/黑屏
                                → Activity 永远收不到 onFirstWindowDrawn
```

这就是为什么冷启动优化中，`finishDrawingWindow` 的时机至关重要 — 它是「首帧可见」的**最后一关**。

### 时序关系图

```
       relayoutWindow          finishDrawingWindow     Transaction.show()
            │                        │                      │
            ▼                        ▼                      ▼
         DRAW_PENDING            COMMIT_DRAW_PENDING      HAS_DRAWN
            │                        │                      │
            │  App 在 Surface        │  WMS 确认            │  Layer 可见
            │  上绘制               │  "画完了"            │
            │                        │                      │
  ───── OR ──────────────────────────────────────────────────────────
            │                        │                      │
            │  buffer 已到 SF        │  buffer 早到了        │  SF 开始合成
            │  (但 Layer hidden)      │  (但还在等)           │  ★ 用户可见
```

### 总结

| 问题 | 答案 |
|------|------|
| `finishDrawingWindow` 做了什么？ | 将窗口状态从 `DRAW_PENDING` → `COMMIT_DRAW_PENDING`，触发 `requestTraversal` |
| 它直接让画面显示了吗？ | **没有**。它只是告诉 WMS「我画好了」，后续 `performShowLocked` → `Transaction.show()` 才是让 SF 开始合成 |
| 每帧都调用吗？ | **不是**。只有首帧和显式 `reportNextDraw()` 时调用。日常帧通过 BLAST 直接上屏 |
| 不调会怎样？ | Layer 永远不可见，用户看白屏，Activity 永远不算 drawn |
| 与 buffer 的关系？ | Buffer 是通过另一条线（BLASTBufferQueue）提交的。finishDrawingWindow 不管 buffer，只管「这个窗口现在可以显示了吗？」 |

**一句话**：`finishDrawingWindow` 是 WMS 侧的「闸门」— buffer 可以先到，但门不开，SF 就不合成。首帧需要开门，日常帧门已经开着，buffer 到了就直接上屏。

---

## 附录：整体渲染管线速查表

| 阶段 | 所在进程 | 关键组件 | 核心操作 |
|------|----------|----------|----------|
| 1. VSync 调度 | App | Choreographer | 等待 VSync，调度 doFrame |
| 2. View 遍历 | App (UI Thread) | ViewRootImpl | measure → layout → draw |
| 3. DisplayList 构建 | App (UI Thread) | ThreadedRenderer | updateRootDisplayList() |
| 4. 同步 + 绘制 | App (RenderThread) | RenderThread | syncFrameState + context->draw() |
| 5. Buffer 提交 | App | BLASTBufferQueue | queueBuffer → Transaction.setBuffer() |
| 6. 首帧上报 | App → WMS | reportDrawFinished | finishDrawingWindow() |
| 7. 窗口显示批准 | system_server (WMS) | WindowStateAnimator | performShowLocked → Transaction.show() |
| 8. VSync 等待 | SurfaceFlinger | MessageQueue | 等待 SF_VSYNC |
| 9. Latch Buffer | SurfaceFlinger | BufferLayer | latchBuffer，等待 acquireFence |
| 10. 合成 | SurfaceFlinger | HWC / RenderEngine | HWC Overlay 或 GPU Client 合成 |
| 11. Present | SurfaceFlinger → Kernel | HWC → DRM | DRM Atomic Commit |
| 12. 显示 | Display Hardware | Panel | 逐行扫描，presentFence signal |

---

> 本文档基于 Android 12~15 源码分析整理，涵盖了从 App 绘制到屏幕显示的完整渲染管线。
