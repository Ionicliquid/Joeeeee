# Jetpack

## Dagger2

### 草稿

1. 依赖注入 控制反转
2. APT
3. 单例 DoubleCheck
4. 构造函数上的inject
5. 成员不能私有

## Hilt

对Dagger2的二次封装，

### 采用代理模式带来实现隔离层

不同的网络请求工具的切换

### 原理

APT+字节码插桩，看生成的代码

## LiveData

1. 数据驱动UI
2. 数据粘性

## Lifecycle

1. 事件驱动状态
2. onResume中addObserver会怎么样？

## ViewBinding

不是apt 技术，利用gradle实现

## MVC MVP MVVM

MVP：接口回调问题
MVVM: ViewModel层 与ViewModel组件

## Databinding

将布局拆分

## ViewModel

1. 保存数据稳定性？
2. vm如何保存状态
3. 非Activity context启动Activity  需要加new-task flag+
4. databind.setLifecycleOwner()?
5. 生命周期重建 AMS
