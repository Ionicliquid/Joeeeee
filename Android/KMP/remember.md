## remember
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