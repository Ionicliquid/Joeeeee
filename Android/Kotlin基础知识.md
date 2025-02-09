# kotlin

## 泛型
### 基本介绍
1. Java 允许实例化没有具体类型参数的泛型类。但 Kotlin 要求在使用泛型时需要**显式声明泛型类型**或者是**编译器能够类型推导出具体类型**，任何不具备具体泛型类型的泛型类都无法被实例化。
2.  类型通配符：`Java`中是`？`,` kotlin`是`*`；
3. 上届约束进一步细化其支持的类型：`T extends Object `，`kotlin`使用`:`代替`extends`没有指定上界约束的类型形参会默认使用 `Any?` 作为上界;
4.  不变：`List<Object>` 不是`List<String> `的父类型；
5. 协变/上界通配符: `List<String>`就是的`List<? extend Object>`子类型, `kotlin`中使用`out`表示;
6. 逆变/下界通配符：`List<? super String>`就是的`List<Object>`子类型, `kotlin`中使用`in`表示;
``` kotlin
  var stringList =  mutableListOf<String>()  
  var objectList:MutableList<Any> = mutableListOf()  
//    objectList = list // 编译报错  
  var anyString :MutableList<out Any> = stringList // 协变  
  var inString :MutableList<in String> = objectList //逆变
```
7. PESC：意为「Producer Extend Consumer Super」，当然在Kotlin中，这句话要改为「Consumer in, Producer out」。这个原则是从集合的角度出发的，其目的是为了实现集合的多态。
	-   如果只是从集合中读取数据，那么它就是个生产者，可以使用`extend`
   
	-   如果只是往集合中增加数据，那么它就是个消费者，可以使用`super`
   
	-   如果往集合中既存又取，那么你不应该用`extend`或者`super`
8. 定义型变的位置，分为使用处型变（对象定义）和声明处型变（类定义），`Java`不支持后者。
```kotlin
interface Source<in T, out R,out U> {  
    fun next(): R  //消费者 只能作为返回值
	fun execute(item: T)  // 生产者：方法参数
    fun getType(type: @UnsafeVariance U)  // 注解
}
```
9. 泛型擦除：todo
10. 获取泛型类型：
	-  反射//todo
	-  `inline`内联函数 + `reified`关键字（类型不擦除 )
```
inline fun <reified T : Activity> Activity.startActivity(context: Context) {  
startActivity(Intent(context, T::class.java))  
}  
  
// 调用  
startActivity<MainActivity>(context)
```

### 参考链接
- [ 一文读懂 Java 和 Kotlin 的泛型难点](https://juejin.cn/post/6935322686943920159#heading-2)
- [kotlin修炼指南7之泛型](https://mp.weixin.qq.com/s/9KjbLAB_99jvh80JGNKgYw)
- [换个姿势，十分钟拿下Java/Kotlin泛型](https://mp.weixin.qq.com/s/vSwx7fgROJcrQwEOW7Ws8A)
- [Kotlin泛型的型变之路](https://mp.weixin.qq.com/s/UkgUfdcKCEP8My7jAxOknQ)

## 语法糖
### inline noinline crossinline
#### inline
inline是用来修饰函数的，在编译时，将该方法的函数类型的参数的方法体内联到方法调用处；
#### noinline
noinline 用来修饰函数类型参数，内联后，该参数不在是函数类型，无法作为函数类型参数传递给其他函数；
#### crossinline
**Kotlin中规定，在非内联函数中，lambda 表达式是不允许使用return返回。** crossline修饰参数，搭配inline使用
如果这个函数类型的参数直接在其他非内联中调用，从形式上来说变成了可以直接返回，编译报错；crossinline的作用仅仅是当有被这个修饰的参数会告诉IDE来检查你写的代码中有没有包含return，假如有的话会编译不过，就是这么简单暴力。
### 参考连接
1. [Kotlin的inline、noinline、crossinline全面分析2 - 掘金](https://juejin.cn/post/7050729336080433188)

## 委托属性

## 扩展函数


