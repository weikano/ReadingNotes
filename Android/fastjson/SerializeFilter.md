## SerializeFilter

### 简介

定制序列化。fastjson支持6种SerializeFilter。

1. PropertyPreFilter： 根据PropertyName判断是否序列化。
2. PropertyFilter：根据PropertyName和PropertyValue判断是否序列化。
3. NameFilter：修改key。如果需要修改key， process返回值即可。
4. ValueFilter：修改value。
5. BeforeFilter：序列化时在最前面添加内容。
6. AfterFilter：序列化时在最后添加内容。