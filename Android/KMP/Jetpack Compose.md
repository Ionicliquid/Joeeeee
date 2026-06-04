# Jetpack Compose
## 基本概念
### 可组合函数与附带效应
#### 可组合函数
1. `可组合函数`是Compose中描述UI的函数
2. 函数带有 `@Composable` 注释。所有可组合函数都必须带有此注释；此注释可告知 Compose 编译器：此函数旨在将数据转换为界面。
3. 此函数接受数据。可组合函数可以接受一些参数，这些参数可让应用逻辑描述界面；
4. 此函数可以在界面中显示文本。为此，它会调用 `Text()` 可组合函数，该函数实际上会创建文本界面元素。可组合函数通过调用其他可组合函数来发出界面层次结构。
5. -此函数不会返回任何内容。发出界面的 Compose 函数不需要返回任何内容，因为它们描述所需的屏幕状态，而不是构造界面 widget。
6. - **可组合函数快速、幂等且没有附带效应**。
    - 使用同一参数多次调用此函数时，它的行为方式相同，并且它不使用其他值，如全局变量或对 `random()` 的调用。
    - 此函数描述界面而没有任何副作用，如修改属性或全局变量。
#### 附带效应
`附带效应`是指发生在**可组合函数作用域**之外的应用状态的变化。由于可组合项的生命周期和属性（例如不可预测的重组、以不同顺序执行可组合项的重组或可以舍弃的重组），可组合项在理想情况下应该是无附带效应的。
``` kotlin
@Composable
fun MyScreen(
    title:String
){

    Log.d("UI日志","MyScreen")
    Column{
        Text(title)
    }

}

```
### 重组与智能重组
在命令式界面模型中，如需更改某个 widget，您可以在该 widget 上调用 setter 以更改其内部状态。在 Compose 中，您可以使用新数据再次调用可组合函数。

这样做会导致函数进行重组，系统会根据需要使用新数据重新绘制函数发出的 widget。Compose 框架可以智能地仅重组已更改的组件。
例如，假设有以下可组合函数，它用于显示一个按钮：
``` kotlin
@Composable
fun ClickCounter(clicks: Int, onClick: () -> Unit) {
    Button(onClick = onClick) {
        Text("I've been clicked $clicks times")
    }
}

```
每次点击该按钮时，调用方都会更新 `clicks` 的值。Compose 会再次调用 lambda 与 `Text` 函数以显示新值；此过程称为“重组”。不依赖于该值的其他函数不会进行重组。

如前文所述，重组整个界面树在计算上成本高昂，因为会消耗计算能力并缩短电池续航时间。Compose 使用`智能重组`来解决此问题。

所谓的`智能重组`就是：Compose根据可组合函数的参数来决定是否进行重组。

也就是说，每一次可组合函数被调用的时候，他会检查所有传入的参数，如果本次传入的参数和上一次传入的参数都是相同的话（这里指的相同是指结构性相等，在kotlin中指的是==，在java中指的是调用对象的equals()方法) ，那么Compose就会略过调用这个可组合函数，以达到最快的重组效率。

让我们回到这个可组合函数，如果他的父级可组合函数由于某种原因触发了重组，那么Compose就会尝试调用MyScreen()来完成刷新，如果title参数没有发生变化的话，Compose实际上就会略过MyScreen的刷新。
``` kotlin
@Composable
fun MyScreen(
    title:String
){
    Column{
        Text(title)
    }
}

```
### Compose的生命周期
![[compose的生命周期.png]]
组合中可组合项的生命周期。 进入组合，执行 0 次或多次重组，然后退出组合。
每一次composable（重组）就是调用一次可组合函数
## 基础的Compose UI组件
### Text组件
`Text` 组件用于显示文本内容，可以通过 `fontSize`、`color`、`fontWeight` 等属性进行样式定制。你还可以使用 `maxLines` 来控制最大行数，`overflow` 来定义文本溢出的行为。

```kotlin
Text( text = "Hello, Jetpack Compose!", fontSize = 20.sp, color = Color.Blue, fontWeight = FontWeight.Bold, maxLines = 1, overflow = TextOverflow.Ellipsis )

```
- `fontSize`：设置字体大小。
- `color`：设置文本颜色。
- `fontWeight`：设置文本粗细。
- `maxLines`：限制文本最大显示行数。
- `overflow`：控制文本超出部分的显示方式，如 `TextOverflow.Ellipsis` 表示溢出部分显示省略号。
### Button组件
`Button` 组件用于触发用户交互事件。你可以通过 `onClick` 设置按钮的点击事件，并通过 `Modifier` 调整按钮的样式（如背景色、形状等）。
```kotlin
Button( onClick = { /* 执行操作 */ }, shape = RoundedCornerShape(8.dp), colors = ButtonDefaults.buttonColors(backgroundColor = Color.Blue) ) { Text("Click Me", color = Color.White) }
```

- `onClick`：按钮点击时触发的事件。
- `shape`：设置按钮的形状，这里使用了圆角形状。
- `colors`：设置按钮的背景色和其他颜色属性。
- `Text`：按钮内的文本。
### Image组件
`Image` 组件用于显示图片，可以通过 `painterResource` 加载资源图片，或者使用 `rememberImagePainter` 来加载网络图片。`contentDescription` 便于辅助功能的使用。
```kotlin
Image( painter = painterResource(id = R.drawable.ic_launcher_foreground), contentDescription = "App Icon", modifier = Modifier.size(80.dp).clip(CircleShape) )
```
- `painter`：加载图片的来源。
- `contentDescription`：辅助功能描述，帮助屏幕阅读器识别图片。
- `modifier`：用于控制图片的大小、形状等样式。
### Column & Row 组件（布局容器）
```kotlin
Column(
    verticalArrangement = Arrangement.spacedBy(10.dp),
    horizontalAlignment = Alignment.CenterHorizontally
) {
    Text("Item 1")
    Text("Item 2")
    Text("Item 3")
}

Row(
    horizontalArrangement = Arrangement.spacedBy(15.dp),
    verticalAlignment = Alignment.CenterVertically
) {
    Text("Left")
    Text("Right")
}
```
- `verticalArrangement`：设置竖直方向的排列方式。
- `horizontalArrangement`：设置水平方向的排列方式。
- `verticalAlignment` 和 `horizontalAlignment`：控制子组件的对齐方式。
### Box组件
`Box` 组件用于重叠视图，通常用于显示叠加元素。通过 `Modifier.align()` 可以精确控制子组件的位置。
``` kotlin
Box(
    modifier = Modifier.size(100.dp).background(Color.Gray)
) {
    Text(
        text = "Overlay Text",
        modifier = Modifier.align(Alignment.Center),
        color = Color.White
    )
}


```
- `modifier`：设置 `Box` 组件的样式，例如大小、背景色。
- `align`：设置子组件的对齐方式，在此示例中为居中对齐。

### Modifier 使用指南
`Modifier` 是 Compose 中非常核心的概念，它允许你对 UI 组件进行样式、布局、行为等方面的修改。通过 `Modifier`，你可以调整组件的大小、间距、背景、对齐、状态等特性。
#### 基本用法
`Modifier` 最常见的用法是设置组件的外观和布局属性。常用的属性包括 `padding`、`background`、`fillMaxSize`、`size` 等。
``` kotlin
Text( "Hello, Compose!", modifier = Modifier .padding(16.dp) .background(Color.LightGray) .fillMaxWidth() )
```
- **`padding`**：为组件设置内边距。你可以指定四个方向的内边距，或者统一设置。
- **`background`**：设置组件的背景色，可以使用 `Color` 或者 `Brush` 来定义渐变色背景。
- **`fillMaxWidth`**：让组件的宽度充满父容器。类似的属性还有 `fillMaxHeight`、`fillMaxSize` 等。
#### 布局相关
`Modifier` 可以与布局容器（如 `Column`、`Row`、`Box` 等）结合使用，以控制组件的对齐和排列方式。

- **`fillMaxWidth` 和 `fillMaxHeight`**：使组件填充父容器的宽度或高度。
- **`wrapContentSize`**：使组件的大小包裹其内容。
- **`align`**：通过 `Modifier.align()` 可以控制组件在父容器中的对齐方式

```kotlin
Box( modifier = Modifier .size(100.dp) .background(Color.Gray) ) { Text( text = "Center", modifier = Modifier.align(Alignment.Center), color = Color.White ) }
```
- **`align`**：用于设置子组件在 `Box` 内的对齐方式，这里是居中对齐。
#### 状态与交互
`Modifier` 还可以用来处理交互事件，例如响应点击、拖动、滑动等。

- **`clickable`**：使组件变得可点击，并触发点击事件。
- **`indication`**：用于显示点击、焦点等状态的视觉效果。
```kotlin
Box( modifier = Modifier .size(100.dp) .background(Color.Red) .clickable { /* 点击事件 */ } )
```
- **`clickable`**：将组件变为可点击，点击时触发相应事件。你可以传递 `onClick` Lambda 作为点击事件的响应。
- **`indication`**：与点击效果结合，提供可视化反馈，例如点击时的阴影效果。
#### 变换与动画
`Modifier` 还支持常见的变换操作，如旋转、缩放、平移等，常用于动画和动态效果。

- **`graphicsLayer`**：控制旋转、缩放、透明度等变换。
- **`rotate`**：旋转组件。
- **`scale`**：缩放组件。
- **`offset`**：偏移组件的位置。
``` kotlin
Box( modifier = Modifier .size(100.dp) .background(Color.Blue) .graphicsLayer(rotationZ = 45f) // 旋转组件 )
```

- **`graphicsLayer`**：用于对组件应用变换，如旋转、缩放、透明度等。
- **`rotationZ`**：旋转角度。
- **`scaleX`、`scaleY`**：设置水平和垂直缩放比例。
#### 圆形、圆角和裁剪
`Modifier` 允许你轻松将组件裁剪成圆形或圆角矩形，这对于实现现代 UI 效果非常有用。

- **`clip`**：裁剪组件的形状，例如圆形、圆角矩形等。
- **`RoundedCornerShape`**：用于定义圆角的大小。
- **`CircleShape`**：将组件裁剪为圆形。
``` kotlin
Box( modifier = Modifier .size(100.dp) .background(Color.Green) .clip(CircleShape) // 裁剪为圆形 .border(2.dp, Color.Black) // 添加边框 )

Box( modifier = Modifier .size(100.dp) .background(Color.Red) .clip(RoundedCornerShape(16.dp)) // 圆角裁剪 .border(2.dp, Color.Black) // 添加边框 )


```
- **`clip(CircleShape)`** ：将组件裁剪为圆形。
- **`clip(RoundedCornerShape(16.dp))`** ：将组件裁剪为圆角矩形，圆角半径为 16.dp。
- **`border`**：为组件添加边框，可以与 `clip` 配合使用，确保圆角或圆形边框效果。
#### 阴影效果
`Modifier` 也可以为组件添加阴影效果，这通常与 `clip` 和 `background` 配合使用，以增加层次感和视觉效果。

- **`shadow`**：为组件添加阴影，可以设置阴影的半径、颜色等。
```kotlin
Box( modifier = Modifier .size(100.dp) .background(Color.Yellow) .clip(RoundedCornerShape(16.dp)) // 圆角裁剪 
.shadow(8.dp, RoundedCornerShape(16.dp)) // 添加阴影 )

```
- **`shadow`**：设置阴影的半径，通常与 `clip` 配合使用来确保阴影与组件的形状一致。
### 使用LazyColumn构建高效列表
## 状态管理
在 Jetpack Compose 中，状态（State）是驱动 UI 更新的核心概念。理解 Compose 中的状态管理机制，有助于构建响应式界面，并提升应用的稳定性与可维护性。
### 什么是状态
在 Android 开发中，**状态**通常指的是界面中随时间变化、影响 UI 展示的数据。例如：

- 表单输入框的文本
- 按钮的点击次数
- 加载数据的结果

传统 View 系统通过 `findViewById` 获取控件，再手动更新视图。而在 Compose 中，UI 是由数据驱动的，数据变化会触发 UI 重新绘制（即 **重组**）。因此，管理和保存这些变化的数据成为 Compose 状态管理的核心。
### 为什么需要 `mutableStateOf` 和 `remember`？

#### 引入 `mutableStateOf`

在 Compose 中，`mutableStateOf` 是用来创建和管理可变状态的工具。它创建的状态对象可以在 UI 中观察，状态变化时会自动触发 UI 更新。例如，下面的代码使用 `mutableStateOf` 来存储按钮的点击次数
``` kotlin
@Composable fun Counter() { // 使用 mutableStateOf 创建可变的状态 var count = mutableStateOf(0) Column { Text(text = "点击次数: ${count.value}") Button(onClick = { count.value++ }) { Text("点击我") } } }

```
在这个例子中，`mutableStateOf(0)` 创建了一个可观察的状态对象，`count` 变量持有这个状态的值。每当按钮点击时，`count.value++` 会更新这个值，并触发 UI 更新。

然而，在这个代码中存在一个问题：每次 UI 更新（即重组）都会重新执行 `Counter()` 函数，这意味着 `count` 每次都会被重置为 `0`。这就导致每次点击按钮时，`count` 始终不变。

### 引入 `remember`
为了避免每次重组时状态丢失，Compose 提供了 `remember` 函数。`remember` 会在同一次重组中保存状态，使得状态数据能够在重组过程中保持不变。我们可以结合 `remember` 和 `mutableStateOf` 来解决这个问题：
``` kotlin
@Composable 
fun Counter() { // 使用 remember 来保留状态
var count by remember { mutableStateOf(0) }
Column { 
	Text(text = "点击次数: $count") 
	Button(onClick = { count++ }) 
		{ Text("点击我") }
	}
 }
```
在这个代码中，`remember { mutableStateOf(0) }` 确保 `count` 在同一次重组过程中保持状态。当点击按钮时，`count` 会正确增加，而 UI 也会随着 `count` 的变化自动更新。

#### `remember` 和 `mutableStateOf` 的底层原理

- `mutableStateOf` 是一个 `State<T>` 对象，内部使用了观察者模式，当状态变化时，Compose 会通知相关的 Composable 重新执行并更新 UI。
- `remember` 本质是一个缓存机制，能够在当前组合范围（Composition）内保持数据，防止 UI 重组时丢失状态。

### Compose 重组机制（Recomposition）
#### 重组是如何工作的？
在 Compose 中，**重组**（Recomposition）是指当状态发生变化时，Compose 会重新执行受影响的 Composable 函数，并重新绘制 UI。重组是 Compose 的核心特性，它使得 UI 动态响应数据的变化。

当我们修改一个 `State` 对象的值时（例如，通过 `mutableStateOf`），Compose 会检测到这个变化，并标记需要更新的 Composable。随着 Composable 被重新执行，UI 会根据新的数据重新呈现。
##### 重组与UI更新的关系
在传统的 Android 开发中，UI 更新是手动触发的，比如调用 `invalidate()` 或 `setText()` 方法。而在 Compose 中，UI 更新由**数据驱动**，当状态发生变化时，UI 会自动更新。
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Log.d("Compose", "Counter 重组")

    Column {
        Text("点击次数: $count")
        Button(onClick = { count++ }) {
            Text("点击我")
        }
    }
}

```

在这个例子中，每次按钮被点击时，`count` 会更新，Compose 会触发重组。通过 `Log` 输出，我们可以看到每次点击按钮时，`Counter` Composable 会重新执行，并在日志中输出 "Counter 重组"。

#### 重组的精细化控制
Compose 的一个关键优势是**高效的重组机制**，即使状态变化，也不会导致整个 UI 被重新绘制。Compose 会根据需要更新**最小范围**的 UI。

- **局部更新**：Compose 会仅重组受状态变化影响的部分 Composables。例如，如果按钮的点击次数变化，只会更新显示次数的 `Text` 组件，而不会重新创建整个 `Counter` 组件。
- **避免不必要的重组**：Compose 通过智能比较来确定哪些 Composables 需要更新，避免了重复的计算和 UI 渲染，优化了性能。
#### 重组的执行过程
1.  **触发重组**：当 `mutableStateOf` 的值发生变化时，Compose 会标记这个 Composable 需要重新执行。
2. **计算新的 UI**：Compose 会重新执行该 Composable，计算新的 UI 树（UI 结构）。
3. **更新 UI**：Compose 会将新的 UI 树与当前的 UI 树进行对比，只更新发生变化的部分，从而高效地呈现更新后的界面。
#### 为什么要关注重组？

理解 Compose 的重组机制对开发者非常重要，因为它能够帮助你：

- **避免性能问题**：确保不必要的 UI 更新不会发生，优化性能。
- **提高响应性**：确保 UI 始终与状态保持同步，用户体验流畅。
### `remember` vs `rememberSaveable`
- `remember` 只能在 **内存** 中保存状态，适用于**短生命周期**的数据。
- `rememberSaveable` 支持**持久化**，即使在 **进程被杀死或配置更改**（如旋转屏幕）时，也能恢复状态。
#### `rememberSaveable` 与 `remember` 的对比

`remember` 和 `rememberSaveable` 都用于在 `Compose` 中保存和恢复状态，但它们的区别在于如何处理配置变化（如屏幕旋转）和进程销毁。
##### `remember`
`remember` 用于保存状态，只在组件重组时保留状态。配置变化（如屏幕旋转）或进程销毁时，状态会丢失。
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Column {
        Text("点击次数: $count")
        Button(onClick = { count++ }) {
            Text("点击我")
        }
    }
}

```
#####  `rememberSaveable`
`rememberSaveable` 类似 `remember`，但它会将状态保存在 `Bundle` 中，在配置变化时恢复状态。适用于需要保持状态的场景，如表单输入。
```kotlin
@Composable
fun Counter() {
    var count by rememberSaveable { mutableStateOf(0) }

    Column {
        Text("点击次数: $count")
        Button(onClick = { count++ }) {
            Text("点击我")
        }
    }
}

```
- **区别**：`rememberSaveable` 可以在配置变化时恢复状态，而 `remember` 只在组件重组时保存状态。
###### `rememberSaveable` 的原理

`rememberSaveable` 使用 `Bundle` 来保存状态，使得状态能在配置变化时恢复。当屏幕旋转或进程销毁后，状态会自动恢复。

### 状态提升（State Hoisting）
状态提升是将状态从子组件提取到父组件，使 UI 与状态管理解耦。这种做法提升了组件的复用性、可测试性，并且允许多个组件共享相同的状态。
#### 状态提升的实际应用
为了实现计数器功能且保证状态在重组时不丢失，我们将状态提升到父组件中进行管理。如下所示
``` kotlin
@Composable
fun ParentComponent() {
    var count by remember { mutableStateOf(0) } // 状态提升到父组件

    Counter(count, onIncrement = { count++ })
}

@Composable
fun Counter(count: Int, onIncrement: () -> Unit) {
    Column {
        Text("点击次数: $count")
        Button(onClick = onIncrement) {
            Text("点击我")
        }
    }
}

```

在这个例子中：

- `ParentComponent` 组件管理 `count` 状态，并通过 `count` 和 `onIncrement` 回调传递给 `Counter` 组件。
- `Counter` 组件仅负责展示文本框和响应用户输入，实际的状态由父组件控制。

这种方式可以确保 `Counter` 组件的复用性：无论多少个 `Counter` 组件，它们都可以通过父组件共享和管理同一个计数器状态。

优势：

- **复用性**：`Counter` 组件变得独立且无状态，能在多个地方复用。
- **解耦性**：UI 展示和状态管理分离，提升了可维护性和测试性。
#### 什么时候不需要状态提升？
并不是所有情况下都需要进行状态提升。在一些简单的、状态完全局部的组件中，直接在组件内部管理状态更加简洁。例如，如果我们有一个组件用于显示计时器，它的状态只在组件内部有效，不需要与外部共享，那么就没有必要提升状态：
``` kotlin
@Composable
fun Timer() {
    var time by remember { mutableStateOf(0) }
    
    LaunchedEffect(true) {
        while (true) {
            delay(1000)
            time++
        }
    }

    Text("计时器: $time")
}

```
在这个例子中，`Timer` 组件内部管理 `time` 状态，它不需要和父组件交互，因此不需要进行状态提升。状态直接管理在 `Timer` 内部就足够了
### Compose 与 ViewModel 状态结合
通常我们通常会使用 `ViewModel` 来持有和管理状态，确保数据在组件生命周期内得以保存。结合 `Compose` 和 `ViewModel`，可以实现更加灵活和稳定的状态管理。
#### **ViewModel + StateFlow / LiveData**

`ViewModel` 用于管理和存储 UI 相关的数据，而 `StateFlow` 和 `LiveData` 是在 `Compose` 中常用的两种可观察的数据类型。通过 `collectAsState`（对于 `Flow`）或 `observeAsState`（对于 `LiveData`），`Compose` 会自动观察数据的变更并更新 UI。

##### 示例：使用 `StateFlow`
``` kotlin
class CounterViewModel : ViewModel() {
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count

    fun increment() {
        _count.value++
    }
}

@Composable
fun CounterScreen(viewModel: CounterViewModel = viewModel()) {
    // collectAsState 会自动观察 StateFlow 数据，并更新 UI
    val count by viewModel.count.collectAsState()

    Column {
        Text("点击次数: $count")
        Button(onClick = { viewModel.increment() }) {
            Text("点击我")
        }
    }
}

```
在这个例子中，`StateFlow` 被用来管理计数器的状态。`collectAsState` 会自动监听 `StateFlow` 的变化并更新 UI。
##### 示例：使用 `LiveData`
``` kotlin
class CounterViewModel : ViewModel() {
    private val _count = MutableLiveData(0)
    val count: LiveData<Int> = _count

    fun increment() {
        _count.value = (_count.value ?: 0) + 1
    }
}

@Composable
fun CounterScreen(viewModel: CounterViewModel = viewModel()) {
    // observeAsState 会自动观察 LiveData 数据，并更新 UI
    val count by viewModel.count.observeAsState(0)

    Column {
        Text("点击次数: $count")
        Button(onClick = { viewModel.increment() }) {
            Text("点击我")
        }
    }
}

```
在这个例子中，`LiveData` 用于管理计数器的状态。`observeAsState` 会自动监听 `LiveData` 的变化，并在数据变更时更新 UI。

- `collectAsState`（适用于 `StateFlow`）和 `observeAsState`（适用于 `LiveData`）能够自动监听数据的变化，并将变化及时反映到 UI 上。
- `StateFlow` 和 `LiveData` 都是响应式的，当数据变化时，它们会自动通知 `Compose` 来触发 UI 更新。
### 总结
- **状态** 是 Compose 的核心，驱动 UI 更新。
- 使用 `mutableStateOf` 创建可变状态，结合 `remember` 来保留状态，避免重组时数据丢失。
- `rememberSaveable` 适用于需要持久化的状态，如配置更改时需要保留的数据。
- 采用**状态提升**模式，解耦 UI 与数据，提升组件复用性和可测试性。
- 与 **ViewModel** 配合使用，可以在复杂应用中保持数据的长期存活和稳定性。

通过理解 Compose 状态管理机制，可以更高效、优雅地实现响应式 UI，提升应用性能与用户体验。
## 附带效应、协程与异步
## 动画
## 高级布局与绘制技巧
###  使用 Canvas 自定义绘制

`Canvas` 是 Compose 中提供的低层次绘图 API，类似于传统的 `onDraw()` 方法。通过 `drawRect()`、`drawCircle()`、`drawPath()` 等方法，你可以绘制各种图形，满足自定义需求。
#### 示例：绘制一个自定义圆形进度条

```kotlin
@Composable
fun CoolProgressBar(
    progress: Float, // 进度 0f - 1f
    modifier: Modifier = Modifier,
    strokeWidth: Float = 20f, // 进度条宽度
    startColor: Color = Color(0xFFFF5722), // 渐变起始色
    endColor: Color = Color(0xFF2196F3) // 渐变结束色
) {
    Canvas(modifier = modifier) {
        val size = size.minDimension
        val radius = size / 2f
        val center = Offset(size / 2f, size / 2f)

        // 绘制背景圆环
        drawCircle(
            color = Color.LightGray.copy(alpha = 0.3f),
            radius = radius - strokeWidth / 2,
            style = Stroke(width = strokeWidth, cap = StrokeCap.Round)
        )

        // 计算渐变颜色
        val brush = Brush.linearGradient(
            colors = listOf(startColor, endColor),
            start = Offset(0f, 0f),
            end = Offset(size, size)
        )

        // 计算进度角度
        val sweepAngle = progress * 360f

        // 绘制进度条
        drawArc(
            brush = brush,
            startAngle = -90f,
            sweepAngle = sweepAngle,
            useCenter = false,
            style = Stroke(width = strokeWidth, cap = StrokeCap.Round)
        )
    }
}

@Composable
fun CoolProgressBarDemo() {
    var progress by remember { mutableFloatStateOf(0.3f) }

    // 进度平滑动画
    val animatedProgress by animateFloatAsState(
        targetValue = progress,
        animationSpec = tween(durationMillis = 1000, easing = FastOutSlowInEasing), label = ""
    )

    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        CoolProgressBar(
            progress = animatedProgress,
            modifier = Modifier
                .size(150.dp)
                .clip(CircleShape)
        )

        Spacer(modifier = Modifier.height(20.dp))

        Button(onClick = { progress = Random.nextFloat().coerceIn(0.1f, 1f)}) {
            Text("随机进度")
      

```
通过 `Canvas`，我们可以绘制一个带有背景圆环和动态进度的圆形进度条。`drawArc()` 让进度条根据传入的 `progress` 绘制。
### 使用 Layout 自定义布局

Compose 中，`Row`、`Column` 和 `Box` 可能无法满足所有布局需求。此时，可以使用 `Layout` API 创建复杂的自定义布局。`Layout` 像是传统View中`onMeasure()`+`onLayout()`的结合。

#### 示例：实现一个流式布局

```koltin
@Composable
fun FlowLayout(
    modifier: Modifier = Modifier,
    maxWidth: Dp = 300.dp, // 限制整体宽度
    content: @Composable () -> Unit
) {
    Layout(
        content = content,
        modifier = modifier
    ) { measurables, constraints ->

        // 将 maxWidth 转换为 Px，并与父布局宽度取最小值，确保不超出
        val layoutMaxWidth = minOf(maxWidth.roundToPx(), constraints.maxWidth)

        // 测量所有子项
        val placeables = measurables.map { it.measure(constraints.copy(maxWidth = layoutMaxWidth)) }

        var currentX = 0
        var currentY = 0
        var maxHeightInRow = 0

        layout(layoutMaxWidth, constraints.maxHeight) {
            placeables.forEach { placeable ->
                // 超出 maxWidth 时换行
                if (currentX + placeable.width > layoutMaxWidth) {
                    currentX = 0
                    currentY += maxHeightInRow
                    maxHeightInRow = 0
                }

                placeable.placeRelative(currentX, currentY)
                currentX += placeable.width
                maxHeightInRow = maxOf(maxHeightInRow, placeable.height)
            }
        }
    }
}

@Composable
fun TestFlowLayout() {
    FlowLayout(maxWidth = 320.dp) {
        FlowText("MyOwnColumn")
        FlowText("places items")
        FlowText("vertically.")
        FlowText("We've done it by hand!")
        FlowText("final")
    }
}

@Composable
fun FlowText(text: String) {
    Text(
        text = text,
        modifier = Modifier
            .padding(8.dp)
            .border(1.dp, Color.Gray)
            .padding(8.dp)
    )
}

```
通过 `Layout` API，可以控制每个元素的尺寸和位置。这里的布局会将每个标签垂直排列，形成一个简单的标签列表。
### 使用 Modifier.drawWithContent 定制绘制
如果你需要在现有组件上添加额外的绘制效果（如边框或渐变效果），可以使用 `Modifier.drawWithContent`。
#### 示例：为文本添加下划线

```kotlin
@Composable
fun UnderlinedText(text: String) {
    Text(
        text = text,
        modifier = Modifier.drawWithContent {
            drawContent() // 先绘制内容
            drawLine(
                color = Color.Red,
                start = Offset(0f, size.height),
                end = Offset(size.width, size.height),
                strokeWidth = 2f
            )
        }
    )
}

@Preview
@Composable
fun UnderlinedTextPreview() {
    UnderlinedText("带下划线的文本")
}

```
### 使用 AndroidView 复用传统 View
通过 `AndroidView`，你可以将传统的 `View` 嵌入 Compose 布局中，复用 XML 布局中已有的自定义视图。
#### 示例：在 Compose 中嵌入 传统View
```kotlin
@Composable
fun WebViewComponent(url: String) {
    AndroidView(
        modifier = Modifier.fillMaxSize(),
        factory = { context ->
            //WebView、SurfaceView、其它自定义View
        }
    )
}

```
### 五、总结

|自定义方式|适用场景|核心 API|
|---|---|---|
|**Canvas**|自定义图形、控件|`drawCircle()`、`drawPath()`|
|**Layout**|自定义布局|`Layout`、`MeasureScope`|
|**drawWithContent**|叠加效果|`drawLine()`、`drawRect()`|
|**AndroidView**|复用传统 View|`AndroidView`|

Compose 提供了灵活的自定义扩展能力，可以满足大多数 UI 设计需求。根据实际情况选择合适的自定义方式，能够帮助你轻松实现多种复杂的 UI 效果。