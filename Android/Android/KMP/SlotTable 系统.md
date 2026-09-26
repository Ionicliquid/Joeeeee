# 简介
1. 对于 Compose 来说，存在这样一棵渲染树，我们将其称为 Compositiion，树上的节点是 LayoutNode，Composition 通过 LayoutNode 完成 measure/layout/draw 的过程最终将 UI 显示到屏幕上。Composition 依靠 Composable 函数的执行来创建以及更新，即所谓的组合和重组。
2. Composable 函数执行过程中产生的所有数据都会存入 SlotTable， 包括 State、CompositionLocal，remember 的 key 与 value 等等 ，这些数据不随函数的出栈而消失，可以跨越重组存在。
3. SlotTable 的数据存储在 Slot 中，一个或多个 Slot 又归属于一个 Group。可以将 Group 理解为树上的一个个节点。说 SlotTable 是一棵树，实际上是 2 个数组；
## 深入理解 Group
![[slots 和 groups.png]]
groups 是一个 IntArray，每 5 个 Int 为一组构成一个 Group 的信息
1. key : Group 在 SlotTable 中的标识，在 Parent Group 范围内唯
2. Group info: Int 的 Bit 位中存储着一些 Group 信息，例如是否是一个 Node，是否包含 Data 等，这些信息可以通过位掩码来获取。
3. Parent anchor: Parent 在 groups 中的位置，即相对于数组指针的偏移
4. Size: Group: 包含的 Slot 的数量
5. Data anchor：关联 Slot 在 slots 数组中的起始位置


# remember
```kotlin

@Composable  
inline fun <T> remember(crossinline calculation: @DisallowComposableCalls () -> T): T =  
    currentComposer.cache(false, calculation)
    
inline fun <T> Composer.cache(invalid: Boolean, block: @DisallowComposableCalls () -> T): T {  
    @Suppress("UNCHECKED_CAST")  
    return rememberedValue().let {  
        if (invalid || it === Composer.Empty) {  
            val value = block()  
            updateRememberedValue(value)  
            value  
        } else it  
    } as T  
}    

```
1. rememberedValue 会读取 SlotTable 中缓存的当前 value；
2. updateRememberedValue：无效值或者无缓存，创建 value，存入 SlotTable 中；
# rememberedValue
``` kotlin
// ComposerImpl
override fun rememberedValue(): Any? = nextSlotForCache() 

internal fun nextSlotForCache(): Any? {  
    return if (inserting) {  
        validateNodeNotExpected()  
        Composer.Empty  
    } else  
        reader.next().let {  
            if (reusing && it !is ReusableRememberObserver) Composer.Empty  
            else if (it is RememberObserverHolder) it.wrapped else it  
        }  
}  
```
1. 处于插入状态，返回Composer.Empty1. 
# updateRememberedValue
``` kotlin
//ComposerImpl
override fun updateRememberedValue(value: Any?) = updateCachedValue(value)
//ComposerImpl
internal fun updateCachedValue(value: Any?) {  
    val toStore =  
        if (value is RememberObserver) {  
            val holder = RememberObserverHolder(value, rememberObserverAnchor())  
            if (inserting) {  
                changeListWriter.remember(holder)  
            }  
            abandonSet.add(value)  
            holder  
        } else value  
    updateValue(toStore)  
}

//ComposerImpl
internal fun updateValue(value: Any?) {  
    if (inserting) {  
        writer.update(value)  
    } else {  
        if (reader.hadNext) {  
  
            val groupSlotIndex = reader.groupSlotIndex - 1  
            if (changeListWriter.pastParent) {  
                changeListWriter.updateAnchoredValue(  
                    value,  
                    reader.anchor(reader.parent),  
                    groupSlotIndex  
                )  
            } else {  
                changeListWriter.updateValue(value, groupSlotIndex)  
            }  
        } else {  
            changeListWriter.appendValue(reader.anchor(reader.parent), value)  
        }  
    }  
}
```