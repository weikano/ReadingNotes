## ParseProcess

### 1. 用于扩展定制反序列化的接口。fastjson支持如下ParseProcess：

- ExtraProcessor 用于处理多余的字段。
- ExtraTypeProcessor 用于处理多余字段时提供类型信息。

### 2. 使用ExtraProcessor和ExtraTypeProcessor

```java
//id为non-public字段，但是有setter方法，所以fastjson支持反序列话
// attributes同为non-public字段，但并没有提供setter方法
public static class VO {
    private int id;
    private Map<String, Object> attributes = new HashMap<String, Object>();
    public int getId() { return id; }
    public void setId(int id) { this.id = id;}
    public Map<String, Object> getAttributes() { return attributes;}
}

class MyExtraProcessor implements ExtraProcessor, ExtraTypeProvider {
    public void processExtra(Object object, String key, Object value) {
        VO vo = (VO) object;
        vo.getAttributes().put(key, value);
    }
    
    public Type getExtraType(Object object, String key) {
        if ("value".equals(key)) {
            return int.class;
        }
        return null;
    }
};


ExtraProcessor processor = new MyExtraProcessor();
//解析name时会触发processor.processExtra，将value,123456分别作为key,value保存至attributes中
//其中又会出发getExtraType，将"123456"转为int
VO vo = JSON.parseObject("{\"id\":123,\"value\":\"123456\"}", VO.class, processor);
Assert.assertEquals(123, vo.getId());
Assert.assertEquals(123456, vo.getAttributes().get("value")); 

```



 

