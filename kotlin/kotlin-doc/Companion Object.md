### Companion Object Extensions

class 内的object对象可以用companion关键字标志

companion object的members可以直接用class名来调用：

```kotlin
class MyClass {
  //Factory can be omitted
  //companion object {fun create():MyClass = MyClass()}  
  companion object Factory {
    fun create():MyClass = MyClass()
  }
}

val instance = MyClass.create()
val x = MyClass.Companion
```

companion object的name可以省略。

虽然看起来像是静态成员，但是运行时仍然是一个实例，所以它可以继承、实现接口:

```kotlin
interface Factory<T> {
  fun create() : T
}

class MyClass {
  companion object : Factory<MyClass> {
    override fun create() : MyClass = MyClass()
  }
}
```

**companion object经过编译后，会生成一个以companion object name命名的静态类(如果声明时省略掉了name，那么叫做Companion)，在持有companion object的class(上述代码中的MyClass)中生成一个public static final Companion 对象**。

**每个class只能有一个companion object**。