## WMS 收到 finishDrawing → 知道客户端已完成此帧绘制 ，一帧绘制了会立即显示吗？还是通知WMS由WMS控制

这个问题很关键。答案是：**不会立即显示**。一帧从绘制到显示经历了以下流水线：

## 三条并行的路径

```
 App 进程                              WMS (system_server)              SurfaceFlinger
 ────────                              ──────────────────              ─────────────
                                       
 ① Buffer                                  (不经过 WMS)
 syncAndDrawFrame()                                                       → BufferQueue
 → RenderThread GPU 渲染                                                    (buffer 就位)
 → BLASTBufferQueue                                                         │
 → SurfaceFlinger BufferQueue                                               │
                                       
 ② Transaction (Window 状态)                                       
 mWmsRequestSyncGroup                finishDrawing()
 合并了的 Transaction  ───────────→   应用到 SurfaceControl        →  SurfaceControl
 (position, crop, visibility...)       (WMS 决定何时 apply)            (状态就位)
                                       
                                                                       ③ 下个 VSYNC
                                                                       SurfaceFlinger
                                                                       合成 + 上屏
```

## 关键时序

1. **Buffer 直接提交**：`syncAndDrawFrame()` 期间，GPU 渲染完成后通过 **BLASTBufferQueue** 直接将像素数据送入 SurfaceFlinger 的 BufferQueue。这条路径**完全绕过 WMS**。
    
2. **Transaction 经 WMS 中转**：`finishDrawing(t, seqId)` 把合并了窗口几何状态（位置、裁剪、变换等）的 Transaction 发给 WMS。WMS 在收到后决定何时 apply 这个 Transaction。
    
3. **SurfaceFlinger 等待两者就绪**：SurfaceFlinger 只有在以下条件**同时满足**时，才会在下一个 VSYNC 合成该帧：
    
    - BufferQueue 中有可用的 buffer
    - SurfaceControl 的状态（由 WMS 的 Transaction 配置）已生效

## `finishDrawing` 的真正作用

`finishDrawing` **不是显示指令**，而是向 WMS 发信号："我这帧已经画完了"。

WMS 用这个信号来控制**多窗口协调**：

- **旋转/Resize 同步**：WMS 需要知道各个窗口都在新尺寸下完成了一帧，才能放心展示新的窗口布局，避免显示中间态或闪烁
- **Activity 启动过渡**：启动窗口（starting window）移除前，WMS 需要确认目标 Activity 已画出第一帧
- **跨进程同步**：多个 SurfaceSyncGroup 可以通过 WMS 作为中介，协调不同进程的帧（如 SurfaceView）

所以，帧的显示时机 = **max(buffer 到达时间, WMS apply Transaction 时间)** + 等到下一个 VSYNC。`finishDrawing` 解决的是后半部分的同步问题。