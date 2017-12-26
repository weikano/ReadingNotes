## Property and Field

类似下面的java代码

```java
class Foo {
	private int bar;
	public int getBar() {
    	return bar;
	}
	public void setBar(int bar) {
      this.bar = bar;
	}
}
```

其中private int bar即定义了一个名为bar的field，而它的getter和setter即为properties。

**kotlin当中只有properties，没有field（有backing field来实现，使用关键字field）**。类似下面代码

```kotlin
class Property {
  val stringRepresentation : String = "abc"
  set(value) {
    if(value.length>0) {
      field = value
//    stringRepresentation = value  
    }
  }
}
```

其中field关键字为编译后会调用putfield指令，类似java中的this.stringRepresentation = value。而如果使用**stringRepresentation = value**，在编译后的class文件中可以看到是调用了setStringRepresentation方法。所以说kotlin中只有properties，没有field。