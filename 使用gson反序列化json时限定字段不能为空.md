##使用gson反序列化json时限定字段不能为空
设计Json Api时，总会遇到将Json字符串转化为Java对象，然后再处理业务逻辑。但对象中的某些字段我们不希望它们为空，这就需要我们对Java对象的字段进行校验。如果对象中包含多级对象，这样校验起来就非常麻烦。
我使用Google的gson进行json的序列化和反序列化操作，所以就想到gson是否具有这样的过滤机制，查看了好久的api，无奈也没有什么收获，所以就想到自己修改gson代码，增加这样一个功能。实现方式类似gson中的`@SerializedName`和`@Expose`等，采用注解实现。
首先声明一个注解
```java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface NotNull {
}
```
gson在解析json字符串时，会将字段信息存在BoundField这个类中，在这个字段增加NotNull字段，标识该字段是否不能为空。
```java
private Map<String, BoundField> getBoundFields(Gson context, TypeToken<?> type, Class<?> raw) {
    Map<String, BoundField> result = new LinkedHashMap<String, BoundField>();
    if (raw.isInterface()) {
      return result;
    }
    Type declaredType = type.getType();
    while (raw != Object.class) {
      Field[] fields = raw.getDeclaredFields();
      for (Field field : fields) {
        boolean serialize = excludeField(field, true);
        boolean deserialize = excludeField(field, false);
        // 判断字段是否有@NotNull注解
        boolean notNull = field.getAnnotation(NotNull.class) != null;
        if (!serialize && !deserialize) {
          continue;
        }
        field.setAccessible(true);
        Type fieldType = $Gson$Types.resolve(type.getType(), raw, field.getGenericType());
        BoundField boundField = createBoundField(context, field, getFieldName(field),
            TypeToken.get(fieldType), serialize, deserialize, notNull);
        // 后面代码省略
}
```
然后在给字段赋值的时候检查字段值是否为null或者是否不包含该字段，同时字段的notNull还是true，如果这样就抛出JsonSyntaxException异常
检查字段值是否为null：
```java
@Override 
void read(JsonReader reader, Object value)
      throws IOException, IllegalAccessException {
    Object fieldValue = typeAdapter.read(reader);
    if (fieldValue == null && notNull) {
        throw new JsonSyntaxException("field " + name + " should not be null");
    }
    if (fieldValue != null || !isPrimitive) {
        field.set(value, fieldValue);
    }
}
```
检查是否包含该字段：
```java
@Override public T read(JsonReader in) throws IOException {
    if (in.peek() == JsonToken.NULL) {
        in.nextNull();
        return null;
    }
    T instance = constructor.construct();
    try {
        in.beginObject();
        Set<String> fields = new HashSet<String>();
        while (in.hasNext()) {
          String name = in.nextName();
          BoundField field = boundFields.get(name);
          fields.add(name);
          if (field == null || !field.deserialized) {
              in.skipValue();
          } else {
              field.read(in, instance);
          }
      }
      for (BoundField field : boundFields.values()) {
          if (field != null && field.notNull && !fields.contains(field.name)) {
              throw new JsonSyntaxException("field " + field.name + " should not be null");
          }
      }
  } catch (IllegalStateException e) {
      throw new JsonSyntaxException(e);
  } catch (IllegalAccessException e) {
      throw new AssertionError(e);
  }
  in.endObject();
  return instance;
}
```
这样，你在解析json字符串到java 对象时，如果想限定某个字段的值不能为null时，只要加上`@NotNull`注解即可。
代码从这里下载：[https://github.com/Will-luffy/gson](https://github.com/Will-luffy/gson)
PS: *由于没有把gson的源码完全看过来，对增加的功能也没有完全充分的进行测试，如果有什么问题，还请指正，谢谢。*
