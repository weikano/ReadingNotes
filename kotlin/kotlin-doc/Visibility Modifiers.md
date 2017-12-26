## Visibility Modifiers

Classes, objects, interfaces, constructors, functions, properties 和 它们的 setters都有访问修饰符(getter的访问修饰符跟property一致)。kotlin有4种访问修饰符：private, protected, internal和public。默认的访问修饰符，如果没有显示指定，是public。

### Packages

functions, properties, classes, objects and interfaces 可以定义为top-level，即直接在package内

```kotlin
package foo
fun baz() {}
class Bar {}
```

- 如果你没有指定任意的访问修饰符，public是默认的，任何地方都可以使用

- 如果设置为private，只在当前文件中可见

- 如果设置为internal，只在当前module可见

  > 1. intellij idea module
  > 2. a maven or gradle project
  > 3. a set of files compiled with one invocation of the ant task

- protected无法在top-level上使用



## Classes and Interfaces

- private 

  > class内可访问(包括所有的member)

- protected 

  > 跟private一样再加上所有subclasses

- internal 

  > module内的所有client，只要能访问到定义的class，就能访问到internal members

- public 

  > 所有能访问定义classes的client都能访问到它的public members

**注意：outer class 不能访问inner class的private member**

**如果你override一个protected member并且没有指定访问修饰符，那么overriding member也是protected的**

### Constructors

使用访问修饰符时，constructor关键字不能省略。

所有constructor默认为public

### Local Declarations

local variables, functions and classes 不能使用访问修饰符

