## Packages

所有源文件都会以

```kotlin
package foo.bar
```

开头。如果没有指定包，那么文件内容属于没有名字的default包

import 关键字不仅仅可以用来import类，你还可以:

- 顶层方法和属性(top level functions and properties)
- enum 常量
- object declarations function and properties

如果top-level 声明标记为private，那么只是单个file内可见

## for 循环

以下情况可以使用for循环：

包含一个成员或者extension-function **iterator()**，返回值为

- 包含一个成员或者extension-function **next()** 并且
- 包含一个成员或者extension-function **hasNext()**返回boolean

以上三个方法必须用operator关键字标示

```kotlin
for ((index, value) in array.withIndex()) {
	println("the element at $index is $value")
}
```

kotlin中的类和方法默认是final的，如果需要override，必须在父类和方法中添加open关键字

Any并不是java.lang.Object；它只有equals, hashCode和toString三个方法，具体可以参考java interoperabillity

用override标识的方法也是open的

override属性值也是同样的，用open标识，子类中override标识同类型同名属性。**注意val可以转为var，但是var不能override成val**。

kotlin中没有static方法，推荐用package-level方法代替