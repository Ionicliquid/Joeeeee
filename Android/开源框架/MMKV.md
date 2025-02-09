1. sp全量更新？
2. mmap: 页的整数倍
3. 定长编码 / 变长编码：protobuff
## MMKV三大优势
1. mmap
## 非递归锁
1. 不可重入锁
2. 设置可重入的属性

# 写
1. MMKV::doAppendDataWithKey
	1. MMKV::loadFromFile 初始化CodedOutputData m_output 记录mmap的指针 ptr
		1. m_output = new CodedOutputData(ptr + Fixed32Size, m_file->getFileSize() - Fixed32Size);

# 锁
``` cpp
bool FileLock::doLock(LockType lockType, bool wait, bool *tryAgain) {  
if (!isFileLockValid()) {  
return false;  
}  
bool unLockFirstIfNeeded = false;  
  
if (lockType == SharedLockType) {  
// don't want shared-lock to break any existing locks  
if (m_sharedLockCount > 0 || m_exclusiveLockCount > 0) {  
m_sharedLockCount++;  
return true;  
}  
} else {  
// don't want exclusive-lock to break existing exclusive-locks  
if (m_exclusiveLockCount > 0) {  
m_exclusiveLockCount++;  
return true;  
}  
// prevent deadlock  
if (m_sharedLockCount > 0) {  
unLockFirstIfNeeded = true;  
}  
}  
  
auto ret = platformLock(lockType, wait, unLockFirstIfNeeded, tryAgain);  
if (ret) {  
if (lockType == SharedLockType) {  
m_sharedLockCount++;  
} else {  
m_exclusiveLockCount++;  
}  
}  
return ret;  
}
```

1. 文件锁是状态锁，没计数器，无论加了多少次锁，一个解锁操作就全解掉；
2. 锁升级/降级 将已经持有的共享锁，升级为互斥锁

# 参考链接
1. [MMKV for Android 多进程设计与实现](https://github.com/Tencent/MMKV/wiki/android_ipc)