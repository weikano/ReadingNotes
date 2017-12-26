## Object Expressions and Declarations

### Object Expressions

> 类似Java中的匿名内部类，kotlin使用object expressions 和 declarations来实现类似功能。

与Java不同的是，kotlin可以指定多个supertype，其中用逗号分割开，如下

```kotlin
open class A(x : Int) {
  public open val y : Int = x
}

interface B {}
//继承A实现B,也可以写成val ab : B = xxx。必须指定类型，因为右边是匿名object
val ab : A = object : A(1), B{
  override val y = 15
}
```

注意匿名object只能在local和private declarations。如果你在public function/type/property中作为返回值使用，实际返回类型会被声明为匿名object的supertype或者Any如果你没有声明supertype，匿名object中的member无法access。

### Object Declarations

单例模式是一种很实用的模式，kotlin定义单利模式的方式更简单

```kotlin
//可以有supertype
object DataProviderManager : DataProviderManagerSub() {
  fun registerDataProvider(provider : DataProvider) {}
  fun allDataProviders:Collection<DataProvider> 
    get() = //...
}

fun useDataProviderManager() {
  DataProviderManager.registerDataProvider()
}
```

Object Declarations不能是local(不能直接嵌套在function里面)，但是可以嵌套在其他object declarations和non-inner classes。

### expressions 和 declarations语义上的不同

- expressions 使用到就马上执行
- 当第一次访问时，declaration是懒加载的
- companion object在class load之后初始化