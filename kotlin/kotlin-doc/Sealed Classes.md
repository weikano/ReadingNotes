## Sealed Class

> Sealed Class是用来标识严格的class结构。在某种意义上，它们是class的枚举。
>
> 要定义sealed class，将sealed关键字放在class之前。一个sealed class可以有多个subclass，但是它们必须定义在跟sealed class的同一个文件内。
>
> 注意，sealed class的非直接subclass，比如sealed class的subclass的subclass，可以放在任何地方。

sealed class的好处在于你使用when表达式时，不需要加else语句。

```kotlin
sealed class Expr
data class Const(val number : Double) : Expr()
data class Sum(val e1 : Expr, val e2 : Expr) : Expr()
object NotANumber : Expr()

fun eval(expr : Expr) : Double = when (expr) {
    is Const -> expr.number
    is Sum -> eval(expr.e1) + eval(expr.e2)
    is NotANumber -> Double.NaN
}
```