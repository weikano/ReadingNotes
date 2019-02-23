### Retrofit2解析

> 动态代理使用ServiceMethod(实现类为HttpServiceMethod).invoke->CallAdapter.adapt(OkHttpCall)->OkHttpCall.execute()
>
> Retrofit通过OkHttp来进行Http请求，所以要通过Request来进行，RequestFactory.create()方法生成对应的Request
>
> ParameterHandler用来给RequestBuilder设置参数，通过apply方法。而RequestFactory中的parameterhandler则是在RequestFactory.build()方法中通过解析调用方法中所有参数的注解来生成对应的handler。

##### returnType和responseType

- returnType在retrofit中用来获取对应的CallAdapter
- responseType用于获取对应的Converter.responseBodyConverter

```java
@Get("/user") Observable<User> getUser(@Query("userId") int id);
//returnType 为Observable<User>
//responseType 为User
```

#### RequestFactory.build()

1. 首先根据method.getAnnotations()方法获取到的注解调用parseMethodAnnotation方法

   > 根据DELETE,GET,HEAD,PATCH,POST,PUT,OPTIONS分别调用parseHttpMethodAndPath方法设置httpMethod值,为注解对应的小写,hasBody(PATCH,POST,PUT时为true,其他false)，relativeUrl的值为注解的value属性。然后解析relativeUrl，将其中的{name}类型的正则中的name保存至relativeUrlParamNames中

2. 然后根据method.getParameterAnnotations()方法调用parseParameter获取对应的parameterHandler，用来解析参数并设置RequestBuilder。实际上调用的parseParameterAnnotation。

   | 注解      | ParameterHandler             | 作用(value都是converter转换参数值成的string值。converter为ConverterFactory.stringConverter，一般使用BuiltInConverters.ToStringConverter.INSTANCE) | 说明                                                         |
   | --------- | ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | Url       | ParameterHandler.RelativeUrl | RequestBuilder.setRelativeUrl                                | 不能跟设置多个Url注解，不能跟Path通用，不能跟Query，QueryName，QueryMap同用 |
   | Path      | ParameterHandler.Path        | RequestBuilder.addPathParam，通过Converter将标注的参数的值转成对应的string值，然后调用addPathParam方法 | 不能跟Query，QueryName,QueryMap,Url同用<br>@GET("/users/{id}") User getUser(@Path("id") int id);<br>后面的url会变成/users/${id}，可以有多个Path注解的参数，对应的GET中的URL也要加入多个{name} |
   | Query     | ParameterHandler.Query       | RequestBuilder.addQueryParam(name, value)，value为Converter转化后的String | @Get('/friends/')getFriend(@Query(value="userId") String id)即链接为/friends?userId=id |
   | QueryName | ParameterHandler.QueryName   | RequestBuilder.addQueryParam(converter(value))               | 上例改为QueryName，链接为/friends?id，如果为contains(Bob)，链接为/friends?contains(Bob) |
   | QueryMap  | ParameterHandler.QueryMap    | RequestBuilder.addQueryParam(key, convert(value))，其中key,value为Map的key,value | 只能标注Map<String,T>类型参数                                |
   | Header    | ParameterHandler.Header      | RequestBuilder.addHeader(name, converter(value))             | 设置请求的Header<br/>void foo(@Header("Accept-Language") String lang) |
   | HeaderMap | ParameterHandler.HeaderMap   | 同上                                                         | void list(@HeaderMap Map<String, String> headers)            |
   | Field     | ParameterHandler.Field       | 用于设置requestBody,RequestBuilder.addFormField              | @POST @FormUrlEncoded Call<ResponseBody> example(@Field("name") String name,@Field("occupation") String occupation)<br/>foo.example("Bob Smith", "President")的requestbody为name=Bob+Smith&occupation=President |
   | FieldMap  | ParameterHandler.FieldMap    | 同上                                                         | @POST @FormUrlEncoded Call<ResponseBody> foo(@FieldMap Map<String,T> fields)<br/>foo({"1":"hi","2":"world"})的requestBody为1=hi&2=world |
   | Part      | ParameterHandler.RawPart     | RequestBuilder.addPart                                       | 跟Multipart一起使用                                          |
   | PartMap   | ParameterHandler.PartMap     | 同上                                                         | 同上                                                         |
   | Body      | ParameterHandler.Body        | 通过bodyConverter将参数转换成RequestBody。RequestBuilder.setBody | 不能有多个Body注解，不能跟FormUrlEncoded或者Multipart一起用  |


#### Converter.Factory

分为**responseBodyConverter**和**requestBodyConverter**

> - requestBodyConverter：用于上述的Body注解，将对应的参数转换成byte[]写入RequestBody
> - responseBodyConverter：将ResponseBody中的值解析为对应的类，用于OkHttpCall.parseResponse方法中，即call.execute方法会触发

##### HttpServiceMethod中的createCallAdapter和createResponseAdapter

- createCallAdapter调用Retrofit.nextCallAdapter->CallAdapter.Factory.get方法，遍历调用Retrofit.callAdapterFactories.get方法获取对应的callAdapter。<br>Retrofit.callAdapterFactories在build时加入DefaultCallAdapterFactory,ExecutorCallAdapterFactory和CompletableFutureCallAdapterFactory(>=android 24)。<br>比如**DefaultCallAdapterFactory如果returnType不是Call则会返回null**。**RxJava2CallAdapterFactory为例，根据传入的returnType，返回对应的RxJava2CallAdapter，如果returnType不是Observable/Flowable/Single/Maybe，则返回null，交给Retrofit.callAdapterFactories中的下一个Factory.get方法处理**。
- createResponseAdapter调用Retrofit.nextResponseBodyConverter，遍历调用Converter.Factory.responseBodyConverter方法获取对应responseType的Converter。**以GsonConverterFactory为例，每次调用responseBodyConverter，都会生一个新的GsonResponseBodyConverter**.

##### Retrofit中的nextRequestBodyConvter

在RequesFactory.Builder中的parseParameterAnnotation中调用，主要是解析方法中用上述注解标注的参数，将其转化为对应的RequestBody。通过调用Converter.Factory.requestBodyConverter方法获取对应的Converter. 以GsonConverterFactory为例，每次调用requestBodyConverter都会返回一个新的GsonRequestBodyConverter，在convert方法中将对应的value写入RequestBody中。

#### ServiceMethod解析

ServiceMethod具体实现类为HttpServiceMethod

通过HttpServiceMethod.parseAnnotations方法，根据对应的注解，实现invoke方法，其步骤如下：

1. 调用Retrofit.callAdapter方法，根据method的返回值类型和注解来获取对应的CallAdapter。
2. 调用Retrofit.responseBodyConverter方法，根据responseType(CallAdapter.responseType())和annotations获取Converter。
3. invoke中通过calladapter.adapt(OkHttpCall)返回对应的值

##### Retrofit中的OkHttp部分

OkHttpCall的execute方法会触发RealCall.execute，最终会交给OkHttpClient中的interceptors来处理请求并获取返回值(**getResponseWithInterceptorChain**方法)。

getResponseWithInterceptorChain方法会用链式的方法(**Interceptor.Chain.proceed(request)**)获取request的返回值。interceptors中的顺序为**client.interceptors**(通过OkHttpClient.Builder().addInterceptor()方法添加)，**retryAndFollowUpInterceptor**, **BridgeInterceptor**, **CacheInterceptor**, **ConnectInterceptor**, **client.networkInterceptors**, **CallServerInterceptor**。

#### RealInterceptorChain.proceed()解析

```kotlin
fun proceed(request:Request, streamAllocation:StreamAllocation, httpCodec:HttpCodec, connection RealConnection):Response {
    RealInterceptorChain next = new RealInterceptorChain(interceptors, streamAllocation, httpCodec,
        connection, index + 1, request, call, eventListener, connectTimeout, readTimeout,
        writeTimeout);
    //获取当前index对应的interceptor
    Interceptor interceptor = interceptors.get(index);
    //interceptors中靠前的interceptor通过intercept下一个interceptor来获取对应的response。即最后的CallServerInterceptor才是最后的生成Response的类。interceptor通过chain.proceed方法来调用下一个interceptor.intercept(chain)方法
    Response response = interceptor.intercept(next);
    //其他处理
    return response;
}
```

##### 内置Interceptor分析

> interceptor.intercept如果能获取到结果，则返回对应的response。如果不能就调用chain.proceed(request)方法交给下一个interceptor处理。

- retryAndFollowUpInterceptor

  > RetryAndFollowUpInterceptor：chain.proceed抛出异常时负责重试，requestCode出现重定向时，负责重新创建一个request再调用chain.proceed（通过while(true)循环）。**这就是为什么重定向时, client.networkInterceptors会多次调用的原因**。

- BridgeInterceptor

  > 负责给request中添加需要的header，比如Keep-Alive, Content-Type, Content-Length之类，以及通过CookieJar添加保存的Cookie；解析Response，添加必要的header以及通过CookieJar解析保存response的header中的cookie相关的信息

- CacheInterceptor

  > 通过**OkHttpClient.Builder.cache(Cache为InternalCache的默认实现，一般使用Cache)**或者internalCache(InternalCache，应用一般不自己实现InternalCache)方法设置Cache, 根据Header获取对应的缓存策略(CacheControl.parse(headers))。intercept方法通过cache获取缓存的response，如果不需要走网络请求那么就返回缓存的response；如果需要继续请求网络，比如缓存失效之类的，则通过cache将返回的response通过LruDiskCache保存。

- ConnectInterceptor

  > 可以理解为跟服务器建立socket连接，这里用到了ConnectionPool**(StreamAllocation.newStream**)。StreamAllocation在RetryAndFollowUpInterceptor.intercept中初始化。

- CallServerInterceptor

  > 真正的跟服务器打交道。通过connection写入request和读取response。