## DataClass

> 用来持有数据而不作其他事情的class，kotlin中用data标志

```kotlin
data class User(val name : String, val age : Int)
```

kotlin compiler会自动生成equals/hashcode、toString()、componentN、copy方法。如果以上任意一个方法有显式的定义在class body或者继承自base type，那么就不会自动生成这个方法。

DataClass一般需要满足下面几条：

- primary constructor必须至少要有一个参数
- 所有的primary constructor参数都必须用val或者var标示
- DataClass不能用open, abstract, sealed 或者inner标示

在JVM中，如果data class要生成无参构造函数，properties必须要有默认值