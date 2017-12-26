## JsonField基础

1. 介绍：若属性是私有的，必须有set方法，否则无法反序列化。

   ```java
   public @interface JSONField {
       // 配置序列化和反序列化的顺序，1.1.42版本之后才支持
       int ordinal() default 0;

        // 指定字段的名称
       String name() default "";

       // 指定字段的格式，对日期格式有用
       String format() default "";

       // 是否序列化
       boolean serialize() default true;

       // 是否反序列化
       boolean deserialize() default true;
   }
   ```

2. 配置方式：可以配置在setter和getter方法或者字段上。

3. 使用format配置日期格式化

   ```java
    public class A {
         // 配置date序列化和反序列使用yyyyMMdd日期格式
         @JSONField(format="yyyyMMdd")
         public Date date;
    }
   ```

4. 使用serialize/deserialize指定字段不序列化

   ```java
   public class A {
         @JSONField(serialize=false)
         public Date date;
    }

    public class A {
         @JSONField(deserialize=false)
         public Date date;
    }
   ```

5. 使用ordinal指定字段顺序：缺省顺序根据fieldName的字母序进行序列化，可以通过ordinal指定字段顺序。

6. 使用serializeUsing制定属性的序列化表

   ```java
   public static class Model {
       @JSONField(serializeUsing = ModelValueSerializer.class)
       public int value;
   }

   public static class ModelValueSerializer implements ObjectSerializer {
       @Override
       public void write(JSONSerializer serializer, Object object, Object fieldName, Type fieldType,
                         int features) throws IOException {
           Integer value = (Integer) object;
           String text = value + "元";
           serializer.write(text);
       }
   }

   Model model = new Model();
   model.value = 100;
   String json = JSON.toJSONString(model);
   Assert.assertEquals("{\"value\":\"100元\"}", json);
   ```

