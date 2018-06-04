### ARouter原理
> 基于api版本1.3.0，compiler版本1.1.4，annotation版本1.0.4，register版本1.0.0

#### 一、ARouter初始化有两种方式：
1. ARouter.init方法会遍历所有的dex文件，找到对应的类，使用loadInfo方式将类添加至Warehouse中的集合中
2. 通过gradle插件动态注入代码到LogisticsCenter.loadRouterMap方法中，具体的可以看arouter-register详解

#### 二、ARouter工程解析
- arouter-annotation：ARouter中所用注解，包括Route、Autowired、Interceptor等
- arouter-api：ARouter对外暴露的接口
- arouter-compiler：annotationProcessor，用来在编译期根据arouter-annotation生成对应的class文件
- arouter-register：gradle插件，在gradle任务的transform期间进行特殊处理。

#### 三、arouter-annotation详解
1. Autowired
> 用来标记field，只在class文件中保留该注解。如果某个类A中有用Autowired标记的field，那么会在编译阶段，通过apt生成对应类名{package.A}$$$ARouter$$$Autowired的实现了ISyringe接口的Java文件并实现了Isyringe接口提供了inject方法。Autowired注解提供了name来表示参数名或者service名，required表示是否必要(如果required为true，而参数又不存在，那么会crash掉，当然primitive类型不会检查)，desc用来描述，没什么用。
2. Interceptor
> 只能用来标记IInterceptor类，提供了priority，用来表示interceptor的优先级，name可以用来生成javadoc
3. Route
> path路由的路径，group用来合并route，name用来生成javadoc，extras没弄懂，priority优先级（没弄懂）
4. RouteType
> Route可以分为好几种类型，定义在RouteType中
5. TypeKind
> 用来区分用autowired标记的field的类型，定义在TypeKind中 
6. RouteMeta
> Route相关的一些数据，RouteType, class, path, group之类的
7. TypeWrapper
> 用来解析泛型之类的？获取真实类型

#### 四、arouter-compiler详解
compiler提供了三种Processor：AutowiredProcessor、InterceptorProcessor以及RouteProcessor。
1. AutowiredProcessor
> 对使用了autowired注解的类A生成A$$$ARouter$$$Autowired并实现了ISyringe接口的帮助方法。ARouter.inject方法会根据path找到对应的Autowired类，并调用AutoWiredServiceImpl.autowire方法，AutowiredServiceImpl最终会找到对应的$$$ARouter$$$Autowired方法，并最终调用该类的inject方法。最后的inject方法，其实就是根据intent以及autowired注解中的name来给对应的字段赋值。
2. InterceptorProcessor
> ARouter.init->LogisticsCenter.init方法会搜索app的dex文件，multidex也不例外，然后找出所有符合的类，根据类是否以$$$Interceptors开头来判断是否为IIntercetporGroup，然后调用IInterceptorGroup的loadInto方法，将对应@Interceptor注解标记的IInterceptor实现类添加至Warehourse.interceptorsIndex中。
>
> InterceptorProcessor就是将同一个module下面用@Interceptor注解标记过并实现了IInterceptor的类，生成对应的ARouter$$$Interceptor&&{gradle-module-name}类，并将所有的标记过的类通过interceptors.put(priority, ClassAnnotated.class)添加至Warehouse.interceptorsIndex中，比如
```java
//其中类名后的app为interceptor所在module的名字
public class ARouter$$Interceptors$$app implements IInterceptorGroup {
    public void loadInto(Map<Integer, class<? extends IInterceptor> interceptors) {
        interceptors.put(priorityA, AInterceptor.class);
        interceptors.put(priorityB, BInterceptor.class);
    }
}
```
3. RouteProcessor
> 根据Route注解生成对应的文件，其中group的默认值通过RouteProcessor.routeVerify设置(第一个/到第二个/之间的值，path必须以/开头)，然e后生成类似ARouter$$$Group$$$group并且实现了IRouteGroup的类，如下
```java
public class ARouter$$Group$$test implements IRouteGroup {
    public void loadInto(Map<String,RouteMeta> atlas) {
        //其中HashMap的值为用autowired标记的field的相关属性, 对应的Integer值为TypeKind中的枚举的值
        atlas.put("/test/activity1", RouteMeta.build(RouteType.ACTIVITY, Test1Activity.class, "/test/activity1", "test", new java.util.HashMap<String, Integer>(){{put("pac", 9); put("ch", 5); put("fl", 6); put("obj", 10); put("name", 8); put("dou", 7); put("boy", 0); put("objList", 10); put("map", 10); put("age", 3); put("url", 8); put("height", 3); }}, -1, -2147483648));
    }
}
```
> IRouteGroup保存在Warehouse中的groupsIndex，初始化过程同样是在ARouter.init->LogisticsCenter.init

#### 五、arouter-register详解
arouter-register是一个gradle插件，那么先找到实现Plugin<Project>的类，即PluginLaunch。
PluginLaunch内容很简单：
> 1. 生成一个RegisterTransform类，并添加至gradle transform中
> 2. 设置RegisterTransform的registerList，包括
com/alibaba/android/arouter/facade/template/{IRouter/IInterceptorGroup/IProviderGroup}三种ScanSetting

接下来看RegisterTransform做了什么来跳过ARouter.init初始化。
1. 遍历所有jar文件，如果jar文件中的类以com/alibaba/android/arouter/routes/开头，那么接着调用scanClass方法；如果是com/alibaba/android/arouter/core/LogisticsCenter.class文件，那么就设置RegisterTransform.fileContainsInitClass为该文件
2. scanClass文件会将RegisterTransform.registerList中的classList，根据interfaceName，填充至对应的ScanSetting中。
3. 浏览所有jar后会开始遍历所有class文件，同样调用scanClass方法
4. 调用RegisterCodeGenerator.insertInitCodeTo方法，最后是使用RouteMethodVisitor在LogisticsCenter.loadRouterMap方法中调用LogisticsCenter.register方法，将RegisterTransform.registerList中的所有class注册进去

#### 六、arouter-api详解
```java
ARouter.getInstance().inject();
ARouter.getInstance().build().navigtaion();
```
1. ARouter.inject方法会找到对应的autowire类，调用其inject方法来初始化用autowired注解标记的字段的值。
2. ARouter.getInstance().build()方法会创建一个Postcard类，用来描述path的信息？
3. Postcard.navigation最终会调用_ARouter.navigation方法，其中LogisticsCenter.completion方法是用来完善Postcard信息，其中PROVIDER和FRAGMENT因为调用了postcard.greenChannel方法，会跳过interceptor处理。接着调用_ARouter._navigation方法
4. _ARouter._navigation方法：
> - 如果是Activity，那么就startActivity(ForResult)方法处理，并且处理回调NavigationCallback.onArrival
> - PROVIDER就直接返回postCard.getProvider(provider是通过LogisticsCenter.completion方法设置的)
> BROADCAST/CONTENT_PROVIDER/FRAGMENT：反射调用构造方法生成实例，如果是fragment，还会调用setArgument将postCard中的extras设置进去
> METHOD/SERVICE：直接返回null
5. Interceptors拦截
> 拦截部分在_ARouter.navigation中，会根据postcard的greenChannel来判断是否启用。具体的拦截实现在InterceptorServiceImpl中，它会用责任链的方式调用所有的interceptor，只要其中某个Interceptor回调onInterupt，那么整个interceptor就会终止(counter.cancel)，如果所有的interceptor都调用了callback.onContinue，那么最终会调用_ARouter._navigation方法，回到4。