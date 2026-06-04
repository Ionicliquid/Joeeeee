# 编译

编译阶段目标是方便状态管理，将 UI 组合函数中植入一些串联管理组合函数的角色，方便进行 UI 构建和状态管理。
开发阶段代码：
``` kotlin
@Composable  
fun ConditionalUI(showProfile: Boolean) {  
    Column {  
        Text("Header")  
        if (showProfile) {  
            ProfileView() // 分支 A  
        } else {  
            LoginPrompt() // 分支 B  
        }  
        Text("Footer")  
    }  
}
```


编译之后伪代码：

```kotlin
fun ConditionalUI(showProfile: Boolean, $composer: Composer, $changed: Int) {  
    // 1. 开始一个可重组的作用域 (Restart Group)  
    //    这使得 ConditionalUI 自身可以被单独重组  
    $composer.startRestartGroup(12345) // 12345 是一个编译器生成的唯一键  
  
    // 检查参数是否有变化，这是智能重组和跳过的关键  
    // (这是一个简化的逻辑)  
    val dirty = $changed  
    if (dirty and0b1011 == 0b0010 && $composer.skipping) {  
        $composer.skipToGroupEnd() // 如果参数未变且父级在跳过，则直接跳过整个函数  
    } else {  
        // 2. 调用 Column Composable  
        //    注意，$composer 和 $changed 参数会被传递下去  
        Column(modifier = ..., $composer, $changed, ...) {  
  
            // 3. 调用 Text("Header")  
            //    由于 "Header" 是一个常量，编译器会标记它为静态，进一步优化  
            Text("Header", $composer, 0b0110)  
  
            // 4. 【核心】处理 if/else 控制流  
            //    编译器为 if 语句创建一个匿名组  
            $composer.startGroup(67890) // 67890 是 if 语句的唯一键  
            if (showProfile) {  
                // 5. 为 if 的 true 分支创建一个组  
                $composer.startGroup(22222)  
                ProfileView($composer, 0)  
                $composer.endGroup()  
            } else {  
                // 6. 为 if 的 false 分支创建一个组  
                $composer.startGroup(33333)  
                LoginPrompt($composer, 0)  
                $composer.endGroup()  
            }  
            $composer.endGroup() // 结束 if 语句的组  
  
            // 7. 调用 Text("Footer")  
            Text("Footer", $composer, 0b0110)  
        }  
    }  
  
    // 8. 结束可重组作用域，并提供一个用于重组的 lambda  
    val scope = $composer.endRestartGroup()  
    scope?.updateScope { nextComposer, force ->  
        // 当需要重组时，这个 lambda 会被调用  
        ConditionalUI(showProfile, nextComposer, $changed or0b1)  
    }  
}

```

  

主要工作如下：

1. Composer 编译器会在 Compose 函数中增加 Composer 和 changed 参数，方便 Composer 来管理所有的 UI 层级和状态，Composer 会将这些信息写入 SlotTable
2. Key 的生成与作用：composer.startRestartGroup(key) 中的 key 至关重要。编译器会根据 Composable 在源码中的 位置 （行号、列号）生成一个在当前父函数中唯一的、稳定的整数 Key。当 if/else 或 when 导致某个 Composable 在下一次重组中“消失”时，Composer 正是依靠这个 Key 来识别出“哪个组不见了”，从而高效地从 SlotTable 中移除对应的节点，而不需要进行复杂的树比对。
- $stable  标记 ：编译器会进行深入的类型分析。如果一个类的所有公共属性都是 val 且类型也是稳定的（基本类型、String、或也标记为 @Stable/@Immutable 的类），编译器会自动将其标记为“稳定”。在计算 changed 位图时，如果一个参数是稳定的且其实例引用没有变（===），Compose 就可以假定其内容也没有变，从而获得一个巨大的性能优化—— 直接跳过对该参数的深入比对 。
- Lambda 的“记住”与重写：对于 Composable 中的 Lambda 表达式，尤其是事件回调（如 onClick），编译器会特殊处理。它会隐式地将 Lambda 包裹在一个 remember 调用中，并捕获其依赖的变量。例如：
``` kotlin
    var name by remember { mutableStateOf("World") }Button(onClick = { println("Hello, $name") }) { /*...*/ }
```
    onClick 这个 Lambda 依赖于 name。编译器会生成类似这样的代码：
``` kotlin 
// 伪代码
val onClickLambda = remember(name) { { println("Hello, $name") } }  
Button(onClick = onClickLambda, ...)
```
 这意味着，只有当 name 发生变化时，才会重新创建一个新的 Lambda 实例。如果 name 不变，Button 在重组时会收到一个完全相同的 Lambda 实例。由于 Function 类型在 Compose 中被认为是稳定的，这使得 Button 可以跳过重组，因为它所有的参数都没有变化。
