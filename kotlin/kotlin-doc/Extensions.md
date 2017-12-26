## Extensions

### Extension functions

kotlin支持extension functions和extension properties，示例代码如下

```kotlin
fun <T> MutableList<T>.swap(index1:Int,index2 : Int) {
    val tmp = this[index1]
    this[index1] = this[index2]
    this[index2] = tmp
}

fun Int.formatPage() : String {
    val format = NumberFormat.getInstance()
    val length = toString().length + 1
    format.maximumIntegerDigits = length
    format.minimumIntegerDigits = length
    return format.format(this)
}

fun main(args: Array<String>) {
    val page = 14
    page.formatPage()
}
```

kotlin对extension function的处理是在编译生成的class文件中添加了两个新的静态方法: swap和formatPage

kotlin对extension function的参数处理是根据定义的参数，如下代码

```kotlin
open class C1
class D : C1()
fun C1.foo() = "C"
fun D.foo() = "D"

fun printFoo(c: C1) = println(c.foo())

fun main(args: Array<String>) {
    printFoo(D())//结果为C
}
```

kotlin的extension function是根据声明中的变量类型来选择调用方法，在生成的class文件中，会将D()强转为C，所以调用的是C1.foo()

如果class中的member function，定义的extension function跟member function有一样的签名，那么总是调用member function。

extension function支持重载，即可以跟member functions的参数不同

### Extension Properties

类似对extension functions的处理，同样是在编译后的class文件中插入一个静态方法。因为extension properties并不是真正的插入一个参数到被扩展的class，所以extension property没有backing field, initializer中也不能使用extension properties。只能通过显示提供的setter和getter方法来定义行为。



## Extensions

### Extension functions

kotlin支持extension functions和extension properties，示例代码如下

```kotlin
fun <T> MutableList<T>.swap(index1:Int,index2 : Int) {
    val tmp = this[index1]
    this[index1] = this[index2]
    this[index2] = tmp
}

fun Int.formatPage() : String {
    val format = NumberFormat.getInstance()
    val length = toString().length + 1
    format.maximumIntegerDigits = length
    format.minimumIntegerDigits = length
    return format.format(this)
}

fun main(args: Array<String>) {
    val page = 14
    page.formatPage()
}
```

kotlin对extension function的处理是在编译生成的class文件中添加了两个新的静态方法: swap和formatPage

kotlin对extension function的参数处理是根据定义的参数，如下代码

```kotlin
open class C1
class D : C1()
fun C1.foo() = "C"
fun D.foo() = "D"

fun printFoo(c: C1) = println(c.foo())

fun main(args: Array<String>) {
    printFoo(D())//结果为C
}
```

kotlin的extension function是根据声明中的变量类型来选择调用方法，在生成的class文件中，会将D()强转为C，所以调用的是C1.foo()

如果class中的member function，定义的extension function跟member function有一样的签名，那么总是调用member function。

extension function支持重载，即可以跟member functions的参数不同

### Extension Properties

类似对extension functions的处理，同样是在编译后的class文件中插入一个静态方法。因为extension properties并不是真正的插入一个参数到被扩展的class，所以extension property没有backing field, initializer中也不能使用extension properties。只能通过显示提供的setter和getter方法来定义行为。

### Companion Object Extensions

```kotlin
class MyClass {
  companion object {}  
}

fun MyClass.Companion.foo() {
  //do something
}
fun useCompanionObjectExtensionFunc(){
  MyClass.foo()
}
```

### Scope of Extensions

一般来说，extension都是定义为top-level。要在定义的package外面使用，我们必须导入extension



