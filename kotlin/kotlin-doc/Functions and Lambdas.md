## Functions and Lambdas

### Functions

#### infix notation (中缀符号)

```
infix fun Int.sh1(x : Int) : Int {
  prinltn
}
```

#### 尾递归(tailrec - tail recursive)

```kotlin
tailrec fun findFixPoint(x : Double = 1.0) : Double 
	= if(x == Math.cos(x)) x else findFixPoint(Math.cos(x))
```

用tailrec标识的尾递归函数，必须在函数最尾端调用，后面不能接其它代码。

#### High Order Function

> 接收一个function作为参数或者返回值为一个function的function

```kotlin
//body:()->表示为一个返回值类型为T的函数
fun <T> invoke(msg : String, body:()->T) : T {
    println(msg)
    return body()
}

fun get() = 1

fun main(args: Array<String>) {
    //使用function get，注意前面的两个冒号，方法名后面不需要()
    println(invoke("high order function",::get))
    //lambda表达式
    println(invoke("hello",{"world"}))
  	//如果function最后一个参数接受的是一个function，可以写在外面
    println(invoke("here"){
        "comes"
    })
}
```

#### Lambda表达式

lambda表达式总是被花括号包围，参数用完整形式定义，也可以缩写。lambda表达式body接在->符号后面。如果推测类型不是Unit，那么lambda表达式的最后一句表达式就做为返回值。

```kotlin
val sum = {x : Int, y : Int -> x + y}
val sumNew : {Int, Int} -> Int = {x,y->x+y}
```

如果lambda表达式只有一个参数，那么参数可记做it

#### 匿名function

```kotlin
fun(x:Int, y : Int) : Int = x + y
```

匿名function跟普通function差不多，除了方法名可以忽略。

### 内联函数（inline function）

高阶函数会带来一些运行时的效率损失：每一个函数都是一个对象，并且会捕获一个壁报。即那些在函数体内会访问到的变量。内存分配（对于函数和类）和虚拟调用会引入运行时间开销。

#### 具体化的参数类型

内联函数支持具体化的参数类型，使用关键字**reified**。由于函数是内联的，不需要反射。

```kotlin
inline fun <reified T> membersOf() = T::class.members
```

使用reified修饰符来限定类型参数，可以在函数内部访问T，几乎就像是一个普通的类一样。

**非内联函数不能有具体化参数。不具有运行时标识的类型（例如非具体化的类型参数或者类似于Nothing的虚构类型）不能用作具体化的类型参数的实参**。

#### 内联属性

inline可以用于没有幕后字段（？backup field？）的属性的访问器，也可以标注独立的属性访问器。