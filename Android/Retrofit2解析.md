### Retrofit2解析

> 动态代理使用ServiceMethod(实现类为HttpServiceMethod).invoke->CallAdapter.adapt(OkHttpCall)->OkHttpCall.execute()
>
> Retrofit通过OkHttp来进行Http请求，所以要通过Request来进行，RequestFactory.create()方法生成对应的Request
>
> ParameterHandler用来给RequestBuilder设置参数，通过apply方法。而RequestFactory中的parameterhandler则是在RequestFactory.build()方法中通过解析调用方法中所有参数的注解来生成对应的handler。

#### RequestFactory.build()

1. 首先根据method.getAnnotations()方法获取到的注解调用parseMethodAnnotation方法

   > 根据DELETE,GET,HEAD,PATCH,POST,PUT,OPTIONS分别调用parseHttpMethodAndPath方法设置httpMethod值,为注解对应的小写,hasBody(PATCH,POST,PUT时为true,其他false)，relativeUrl的值为注解的value属性。然后解析relativeUrl，将其中的{name}类型的正则中的name保存至relativeUrlParamNames中

2. 然后根据method.getParameterAnnotations()方法调用parseParameter获取对应的parameterHandler，用来解析参数并设置RequestBuilder。实际上调用的parseParameterAnnotation。

   | 注解      | ParameterHandler             | 作用(value都是converter转换参数值成的string值。converter为ConverterFactory.stringConverter，一般使用BuiltInConverters.ToStringConverter.INSTANCE) | 说明                                                         |
   | --------- | ---------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | Url       | ParameterHandler.RelativeUrl | RequestBuilder.setRelativeUrl                                | 不能跟设置多个Url注解，不能跟Path通用，不能跟Query，QueryName，QueryMap同用 |
   | Path      | ParameterHandler.Path        | RequestBuilder.addPathParam，通过Converter将标注的参数的值转成对应的string值，然后调用addPathParam方法 | 不能跟Query，QueryName,QueryMap,Url同用                      |
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

分为responseBodyConverter和requestBodyConverter

> - requestBodyConverter：用于上述的Body注解，将对应的参数转换成byte[]写入RequestBody
> - responseBodyConverter：将ResponseBody中的值解析为对应的类，用于OkHttpCall.parseResponse方法中，即call.execute方法会触发

##### HttpServiceMethod中的createCallAdapter和createResponseAdapter

//todo

##### Retrofit中的nextRequestBodyConvter

//todo

#### 一 ServiceMethod解析

ServiceMethod具体实现类为HttpServiceMethod

通过HttpServiceMethod.parseAnnotations方法，根据对应的注解，实现invoke方法，其步骤如下：

1. 调用Retrofit.callAdapter方法，根据method的返回值类型和注解来获取对应的CallAdapter。
2. 调用Retrofit.responseBodyConverter方法，根据responseType(CallAdapter.responseType())和annotations获取Converter。
3. invoke中通过calladapter.adapt(OkHttpCall)返回对应的值

#### 二 CallAdapter解析

#### 三 注解

