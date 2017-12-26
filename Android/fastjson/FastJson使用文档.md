## FastJson使用说明

- Android第一次序列化和反序列化的优化

  > ```java
  > public class SerializeConfig {
  >    ObjectSerializer registerIfNotExists(Class<?> clazz, // 类型
  >                                              int classModifers, // 如果类型为public，使用Modifier.PUBLIC
  >                                          boolean fieldOnly, // 是否只有field，没有getter/setter
  >                                          boolean jsonTypeSupport, // 是否有@JSONType配置
  >                                          boolean jsonFieldSupport, // 是否有@JSONField配置
  >                                          boolean fieldGenericSupport // 是否有泛型信息
  >                                                 );
  > }
  >
  > public class ParserConfig {
  >     ObjectDeserializer registerIfNotExists(Class<?> clazz, // 类
  >                                                 int classModifiers, // 如果类型为public，使用Modifier.PUBLIC
  >                                             boolean fieldOnly, // 是否只有field，没有getter/setter
  >                                             boolean jsonTypeSupport, // 是否有@JSONType配置
  >                                             boolean jsonFieldSupport, // 是否有@JSONField配置
  >                                             boolean fieldGenericSupport // 是否有泛型信息
  >                                                 );
  > }
  >
  > ParserConfig.getGlobalInstance()
  >        .registerIfNotExists(MediaContent.class, Modifier.PUBLIC, true, false, false, false);
  >
  > SerializeConfig.getGlobalInstance()
  >        .registerIfNotExists(MediaContent.class, Modifier.PUBLIC, true, false, false, false);
  > ```

- BeanToArray

  > 普通模式下，JavaBean映射为json object，BeanToArray模式映射为json array。
  >
  > BeanToArray模式下少了key的输出
  >
  > ```java
  > class Model {
  >    public int id;
  >    public String name;
  > }
  >
  > Model model = new Model();
  > model.id = 1001;
  > model.name = "gaotie";
  >
  > // {"id":1001,"name":"gaotie"}
  > String text_normal = JSON.toJSONString(model); 
  >
  > // [1001,"gaotie"]
  > String text_beanToArray = JSON.toJSONString(model, SerializerFeature.BeanToArray); 
  >
  > // support beanToArray & normal mode
  > JSON.parseObject(text_beanToArray, Feature.SupportArrayToBean); 
  > ```

  > BeanToArray也可以局部使用
  >
  > ```java
  > class Company {
  >      public int code;
  >       @JSONField(serialzeFeatures=SerializerFeature.BeanToArray, parseFeatures=Feature.SupportArrayToBean)
  >      public List<Department> departments = new ArrayList<Department>();
  > }
  > class Department {
  >      public int id;
  >      public Stirng name;
  >      public Department() {}
  >      public Department(int id, String name) {this.id = id; this.name = name;}
  > }
  > ```

- [支持类似Jackson的builder模式]: https://github.com/alibaba/fastjson/wiki/BuilderSupport	"支持类似Jackson的builder模式"

- 类级别的SerializeFilter

  > ```java
  > public class ClassNameFilterTest extends TestCase {
  >     public void test_filter() throws Exception {
  >         NameFilter upcaseNameFilter = new NameFilter() {
  >             public String process(Object object, String name, Object value) {
  >                 return name.toUpperCase();
  >             }
  >         };
  >         SerializeConfig.getGlobalInstance() //
  >                        .addFilter(A.class, upcaseNameFilter);
  >         
  >         Assert.assertEquals("{\"ID\":0}", JSON.toJSONString(new A()));
  >         Assert.assertEquals("{\"id\":0}", JSON.toJSONString(new B()));
  >     }
  >
  >     public static class A {
  >         public int id;
  >     }
  >
  >     public static class B {
  >         public int id;
  >     }
  > }
  > ```

- 关闭默认反序列化为BigDecimal

  > fastjson缺省反序列化带小数点的数值类型为BigDecimal，但还是存在使用float/double的场景
  >
  > ```java
  > //全局关闭
  > JSON.DEFAULT_PARSER_FEATURE &= ~ Feature.UseBigDecimal.getMask();
  > //局部关闭
  > int disable = JSON.DEFAULT_PARSER_FEATURE &= ~ Feature.UseBigDecimal.getMask();
  > String json = "...";
  > Class type = JavaBean.class;
  > JSON.parseObject(json, type, disable);
  > ```

- 将枚举类型序列化输出

  > ```java
  > public static enum OrderType {
  >     PayOrder(1, "支付订单"), //
  >     SettleBill(2, "结算单");
  >
  >     public final int value;
  >     public final String remark;
  >
  >     private OrderType(int value, String remark) {
  >         this.value = value;
  >         this.remark = remark;
  >     }
  > }
  > SerializeConfig.globalInstance.configEnumAsJavaBean(OrderType.class);
  > ```


- ExtraProcessable

  > 用于定制对象的反序列化功能。如果对象没有对应的public setter和public field，就会调用processExtra方法。

- 属性智能匹配（DisableFieldSmartMatch）

  > 如果json文本和JavaBean的属性不是精确匹配，fastjson会通过多种方式匹配查找，消耗较多资源。
  >
  > ```java
  > //用JSONType标注
  > //@JSONType(parseFeature = Feature.DisableFieldSmartMatch)
  > public class Test {
  >   public int personId;
  > }
  > assertEquals(0, JSON.parseObject("{\"person_id\":123}", Model_for_disableFieldSmartMatchMask2.class).personId);
  >
  > //或者在toJSONString时使用
  > assertEquals(123
  >           , JSON.parseObject("{\"person_id\":123}"
  >                            , Model_for_disableFieldSmartMatchMask.class).personId);
  > assertEquals(0
  >            , JSON.parseObject("{\"person_id\":123}"
  >                             , Model_for_disableFieldSmartMatchMask.class, Feature.DisableFieldSmartMatch).personId);
  > ```

- 支持非public字段（SupportNonPublicField）

  > ```java
  > public class Model {
  >   private int id;
  > }
  > Model model = JSON.parseObject("{\"id\":123}", Model.class, Feature.SupportNonPublicField);
  > ```

- FieldBased TODO

- FieldTypeResolver

  > 用于解析嵌套对象时，自动识别子对象的的类型信息。FieldTypeResolver只会作用于value类型为json object的字段。自动识别类型，能避免二次解析。
  >
  > ```java
  > public static class Item {
  >     public int value;
  > }
  >
  > FieldTypeResolver fieldResolver = new FieldTypeResolver() {
  >     public Type resolve(Object object, String fieldName) {
  >         if (fieldName.startsWith("item_")) { // 字段名称为item_开始的对象，识别类型为Item
  >             return Item.class;
  >         }
  >         
  >         return null;
  >     }
  > };
  >
  > String text = "{\"item_0\":{},\"item_1\":{},\"item_2\":1001}";
  > JSONObject o = JSON.parseObject(text, JSONObject.class, fieldResolver);
  > Assert.assertTrue(o.get("item_0") instanceof Item);
  > Assert.assertTrue(o.get("item_1") instanceof Item);
  > Assert.assertTrue(o.get("item_2") instanceof Integer);//还是Integer，因为value不是Object。
  > ```

- 反序列话时多字段名称支持

  ```java
  public static class Model {
      public int id;

      @JSONField(alternateNames = {"user", "person"})
      public String name;
  }
  ```

- 支持gzip

  当json传输较大的byte[]时可以启用减少网络传输

  ```java
  public static class Model {
      @JSONField(format = "gzip")
      public byte[] value;
  }

  Model model = new Model();

  StringBuffer buf = new StringBuffer();
  for (int i = 0; i < 1000; ++i) {
      buf.append("0123456890");
      buf.append("ABCDEFGHIJ");
  }

  model.value = buf.toString().getBytes();

  String json = JSON.toJSONString(model);

  assertEquals("{\"value\":\"H4sIAAAAAAAAAO3IsRGAIBAAsJVeUE5LBBXcfyC3sErKxJLyupX9iHq2ft3PmG8455xzzjnnnHPOOeecc84555xzzjnnnHPOOeecc84555xzzjnnnHPOOeecc84555z7/T6powiAIE4AAA==\"}", json);

  Model model1 = JSON.parseObject(json, Model.class);
  Assert.assertArrayEquals(model.value, model1.value);
  ```

- jsonDirect TODO

- unwrapped

  ```java
  public static class VO {
      public int id;

      @JSONField(unwrapped = true)
      public Localtion localtion;
  }

  public static class Localtion {
      public int longitude;
      public int latitude;

      public Localtion() {}

      public Localtion(int longitude, int latitude) {
          this.longitude = longitude;
          this.latitude = latitude;
      }
  }

  VO vo = new VO();
  vo.id = 123;
  vo.localtion = new Localtion(127, 37);
  //直接将location中的字段flattern到VO中，不需要再嵌套一层location:{latitude:, longtitude:}
  String text = JSON.toJSONString(vo);
  Assert.assertEquals("{\"id\":123,\"latitude\":37,\"longitude\":127}", text);

  VO vo2 = JSON.parseObject(text, VO.class);
  assertNotNull(vo2.localtion);
  assertEquals(vo.localtion.latitude, vo2.localtion.latitude);
  assertEquals(vo.localtion.longitude, vo2.localtion.longitude);
  ```

- JSONPath：可以当作OQL语言来使用。

- typeKey TODO

- LabelFilter

  > 使用LabelFilter，根据不同场景定制序列化
  >
  > ```java
  > public static class VO {
  >     private int    id;
  >     private String name;
  >     private String password;
  >     private String info;
  >     @JSONField(label = "normal")
  >     public int getId() {
  >         return id;
  >     }
  >     public void setId(int id) {
  >         this.id = id;
  >     }
  >     @JSONField(label = "normal")
  >     public String getName() {
  >         return name;
  >     }
  >     public void setName(String name) {
  >         this.name = name;
  >     }
  >     @JSONField(label = "secret")
  >     public String getPassword() {
  >         return password;
  >     }
  >     public void setPassword(String password) {
  >         this.password = password;
  >     }
  >     public String getInfo() {
  >         return info;
  >     }   
  >     public void setInfo(String info) {
  >         this.info = info;
  >     }
  > }
  >
  > VO vo = new VO();
  > vo.setId(123);
  > vo.setName("wenshao");
  > vo.setPassword("ooxxx");
  > vo.setPassword("ooxxx");
  > //只包含label="normal"的字段
  > String text1 = JSON.toJSONString(vo, Labels.includes("normal"));
  > Assert.assertEquals("{\"id\":123,\"name\":\"wenshao\"}", text1);
  > //去掉label="secrete"的字段
  > String text2 = JSON.toJSONString(vo, Labels.excludes("secret"));
  > Assert.assertEquals("{\"id\":123,\"info\":\"fofo\",\"name\":\"wenshao\"}", text2);
  > ```

- 自定义ObjectDeserializer

  1. 实现一个ObjectDeserializer。
  2. 注册的ParserConfig中。

  ```java
  public static enum OrderActionEnum {
      FAIL(1), SUCC(0);
      private int code;
      OrderActionEnum(int code){
          this.code = code;
      }
  }

  public static class Msg {
      public OrderActionEnum actionEnum;
      public String          body;
  }
  //实现
  public static class OrderActionEnumDeser implements ObjectDeserializer {

      @SuppressWarnings("unchecked")
      @Override
      public <T> T deserialze(DefaultJSONParser parser, Type type, Object fieldName) {
          Integer intValue = parser.parseObject(int.class);
          if (intValue == 1) {
              return (T) OrderActionEnum.FAIL;
          } else if (intValue == 0) {
              return (T) OrderActionEnum.SUCC;
          }
          throw new IllegalStateException();
      }

      @Override
      public int getFastMatchToken() {
          return JSONToken.LITERAL_INT;
      }

  }
  //注册
  ParserConfig.getGlobalInstance().putDeserializer(OrderActionEnum.class, new OrderActionEnumDeser());

  Msg msg = JSON.parseObject("{\"actionEnum\":1,\"body\":\"A\"}", Msg.class);
  Assert.assertEquals(msg.body, "A");
  Assert.assertEquals(msg.actionEnum, OrderActionEnum.FAIL);
  ```

- 将非字符串类型输出为字符串类型（WriteNonStringValueAsString）

  ```java
  public class VO {
    public boolean id;
  }
  VO vo = new VO();
  vo.id = true;

  Assert.assertEquals("{\"id\":\"true\"}"
          , JSON.toJSONString(vo, SerializerFeature.WriteNonStringValueAsString));
  Assert.assertEquals("{\"id\":true}", JSON.toJSONString(vo));
  ```

- 泛型

  ```java
  import com.alibaba.fastjson.TypeReference;
  //类似gson
  List<VO> list = JSON.parseObject("...", new TypeReference<List<VO>>() {});
  //如下性能更好
  final static Type type = new TypeReference<List<VO>>() {}.getType();
  List<VO> list = JSON.parseObject("...", type);
  ```

  ​