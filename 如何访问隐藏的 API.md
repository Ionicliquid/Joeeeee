1. LayoutInflater 虽然提供了接口允许设置创建 View 的工厂，但是通过一个 boolean 属性控制此工厂只能被设置一次；
2. 在前面提到的流程中，默认的 Androidx 的 AppcompatActivity 已经设置了一次。要想自定义这个工厂，需要修改这个 boolean 值，在 Android 10 之前可以直接反射修改，之后，这个值被属性被设置为隐藏 api，不允许应用调用；
3. 我采用的方案是不修改底层代码，完全基于Java 中Unsafe和 MethodHandle实现；
4. Unsafe 直接通过访问和修改对象的内存地址来控制属性的访问；
	1. a. 对象的内存布局：对象头+实例数据+对齐填充，偏移量是相对于对象起始地址的字节距离
5. MethodHandle是反射API 的一种替代，其artFieldOrMethod属性是操作目标类方法和属性的句柄；
6. 具体的实现流程可以分为 3 步：
	1. 获取 Class 的 Class 对象 iFields 属性的偏移，通过此偏移得到目标类的属性数量，它也属性集合的首地址；
	2. 通过在工具类定义 2 个已知属性，计算出每个属性的内存大小,结合属性集合得到每个属性的地址；
	3. 获取工具类中已知属性的 MethodHandle，将属性句柄的地址替换为目标属性的地址，就直接操作 MethodHandle进行修改；