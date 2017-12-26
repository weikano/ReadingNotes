## Generics

kotlin的泛型增加了两个关键字**in**和**out**

- in

  > 代表输入的只能是泛型类型

- out

  > 代表返回值类型只能是泛型类型

Java中通配符<? extends Animal>代表只能使用Animal的子类，<? super Animal>只能使用Animal的父类

```java
List<? extends Animal> list = new ArrayList<>();
list.add(new Animal("animal"));//compile error
list.add(new Bird("bird"));//compile error
list.add(new Cat("cat"));//compile error
```

list类型为List<? extends Animal>, 即List<>中可以是Animal的subclass，但是却无法确定究竟是哪一个类。可以是上述的Bird, Cat，也可以是其他的Tiger类型。所以说上述3个add都可能造成类型兼容问题。所以上述list唯一能add的对象是null。

```java
public void testAdd(List<? super Bird> list){
	list.add(new Bird("bird"));//success
	list.add(new Magpie("magpie"));//success
	list.add(new Animal());//error
}
```

由上述代码可知，list中的对象是Bird的父类，那么Bird继承自Bird的父类，Magpie继承自Bird，所以都可以add。而Animal则不同，因为Bird的父类可能还有其他类型，比如是Animal的子类，所以无法add。

kotlin中通配符如下

```kotlin
fun <T:String> foo()//Java中的? extends String，即String的子类
fun <in String> foo()//java中的? super String, 即String的父类
```