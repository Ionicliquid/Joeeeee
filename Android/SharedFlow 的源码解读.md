![[SharedFlow_1.png]]
1. minCollectorIndex：不指定收集者时为head + bufferSize
# emit
``` kotlin
override suspend fun emit(value: T) {  
    if (tryEmit(value)) return // fast-path  
    emitSuspend(value)  
}
```
## tryEmit
``` kotlin
override fun tryEmit(value: T): Boolean {  
    var resumes: Array<Continuation<Unit>?> = EMPTY_RESUMES  
    val emitted = synchronized(this) {  
        if (tryEmitLocked(value)) {  
            resumes = findSlotsToResumeLocked(resumes)  
            true  
        } else {  
            false  
        }  
    }  
    for (cont in resumes) cont?.resume(Unit)  
    return emitted  
}
```
### tryEmitLocked
``` kotlin
private fun tryEmitLocked(value: T): Boolean {  
    if (nCollectors == 0) return tryEmitNoCollectorsLocked(value)  
    if (bufferSize >= bufferCapacity && minCollectorIndex <= replayIndex) {  
        when (onBufferOverflow) {  
            BufferOverflow.SUSPEND -> return false   
BufferOverflow.DROP_LATEST -> return true   
BufferOverflow.DROP_OLDEST -> {}   
        }  
    }  
    enqueueLocked(value)  
    bufferSize++  
    if (bufferSize > bufferCapacity) dropOldestLocked()  
    if (replaySize > replay) {   
        updateBufferLocked(replayIndex + 1, minCollectorIndex, bufferEndIndex, queueEndIndex)  
    }  
    return true  
}
```
1. nCollectors == 0 没有指定收集者，直接返回 true
#### tryEmitNoCollectorsLocked
``` kotlin
private fun tryEmitNoCollectorsLocked(value: T): Boolean {  
    assert { nCollectors == 0 }  
    if (replay == 0) return true   
enqueueLocked(value)   
    bufferSize++  
    if (bufferSize > replay) dropOldestLocked()  
    minCollectorIndex = head + bufferSize   
    return true  
}
```
1. replay == 0 回放容量为 0，直接返回；
2. enqueueLocked：将数据插入缓存数组，更新缓存`bufferSize`大小，如果缓存大小超过回放大小，则丢弃最先发射的数据
3. 
### findSlotsToResumeLocked
``` kotlin
private fun findSlotsToResumeLocked(resumesIn: Array<Continuation<Unit>?>): Array<Continuation<Unit>?> {  
    var resumes: Array<Continuation<Unit>?> = resumesIn  
    var resumeCount = resumesIn.size  
    forEachSlotLocked loop@{ slot ->  
        val cont = slot.cont ?: return@loop // only waiting slots  
        if (tryPeekLocked(slot) < 0) return@loop // only slots that can peek a value  
        if (resumeCount >= resumes.size) resumes = resumes.copyOf(maxOf(2, 2 * resumes.size))  
        resumes[resumeCount++] = cont  
        slot.cont = null // not waiting anymore  
    }  
    return resumes  
}
```
# enqueueLocked
``` kotlin
private fun enqueueLocked(item: Any?) {  
    val curSize = totalSize  
    val buffer = when (val curBuffer = buffer) {  
        null -> growBuffer(null, 0, 2)  
        else -> if (curSize >= curBuffer.size) growBuffer(curBuffer, curSize,curBuffer.size * 2) else curBuffer  
    }  
    buffer.setBufferAt(head + curSize, item)  
}
```
将数据插入缓存buff中，并指定索引