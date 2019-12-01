[JSON](http://json.org/json-zh.html) 是一种文本形式的数据交换格式，它比XML更轻量、比二进制容易阅读和编写，调式也更加方便。其重要性不言而喻。解析和生成的方式很多，Java中最常用的类库有：JSON-Java、Gson、Jackson、FastJson等。

#### Gson的基本用法

Gson提供了`fromJson()` 和`toJson()` 两个直接用于解析和生成的方法，前者实现反序列化，后者实现了序列化。同时每个方法都提供了重载方法，常用的有5个。

###### **基本数据类型的解析**

```java
Gson gson = new Gson();
int i = gson.fromJson("100", int.class);              //100
double d = gson.fromJson("\"99.99\"", double.class);  //99.99
boolean b = gson.fromJson("true", boolean.class);     // true
String str = gson.fromJson("String", String.class);   // String
```

###### **POJO类的生成与解析**

```java
public class User {
    //省略其它
    public String name;
    public int age;
    public String emailAddress;
}
```

###### 生成JSON：

```java
Gson gson = new Gson();
User user = new User("怪盗kidou",24);
String jsonObject = gson.toJson(user); // {"name":"怪盗kidou","age":24}
```

###### 解析JSON：

```java
Gson gson = new Gson();
String jsonString = "{\"name\":\"怪盗kidou\",\"age\":24}";
User user = gson.fromJson(jsonString, User.class);
```

#### 属性重命名 @SerializedName 注解的使用

上面POJO的生成与解析可以看出json的字段和值是的名称和类型是一一对应的，但也有一定容错机制，解析对应字符串。

```java
@SerializedName("email_address")
public String emailAddress;
```

#### Gson中使用泛型

数组比较简单

```java
Gson gson = new Gson();
String jsonArray = "[\"Android\",\"Java\",\"PHP\"]";
String[] strings = gson.fromJson(jsonArray, String[].class);
```

集合解析使用Gson提供的TypeToken

```java
Gson gson = new Gson();
String jsonArray = "[\"Android\",\"Java\",\"PHP\"]";
String[] strings = gson.fromJson(jsonArray, String[].class);
List<String> stringList = gson.fromJson(jsonArray, new TypeToken<List<String>>() {}.getType());
```

注：`TypeToken`的构造方法是`protected`修饰的,所以上面才会写成`new TypeToken<List<String>>() {}.getType()` 而不是  `new TypeToken<List<String>>().getType()`

#### 使用GsonBuilder导出null值、格式化输出、日期时间

```java
Gson gson = new GsonBuilder()
        //序列化null,默认不导出null值
        .serializeNulls()
        // 设置日期时间格式，另有2个重载方法
        // 在序列化和反序化时均生效
        .setDateFormat("yyyy-MM-dd")
        // 禁此序列化内部类
        .disableInnerClassSerialization()
        //生成不可执行的Json（多了 )]}' 这4个字符）
        .generateNonExecutableJson()
        //禁止转义html标签
        .disableHtmlEscaping()
        //格式化输出
        .setPrettyPrinting()
        .create();
```

#### 字段过滤的几种方法

###### 基于@Expose注解

**@Expose**提供了两个属性（序列化和反序列化），默认值都为true。

```java
Gson gson = new GsonBuilder()
        .excludeFieldsWithoutExposeAnnotation()
        .create();
gson.toJson(category);
```

###### 基于版本

Gson在对基于版本的字段导出提供了两个注解 `@Since` 和 `@Until`,和`GsonBuilder.setVersion(Double)`配合使用。`@Since` 和 `@Until`都接收一个`Double`值。当前版本(GsonBuilder中设置的版本) **大于等于Since**的值时该字段导出，**小于Until**的值时该该字段导出。

```java
class SinceUntilSample {
    @Since(4)
    public String since;
    @Until(5)
    public String until;
}

public void sineUtilTest(double version){
        SinceUntilSample sinceUntilSample = new SinceUntilSample();
        sinceUntilSample.since = "since";
        sinceUntilSample.until = "until";
        Gson gson = new GsonBuilder().setVersion(version).create();
        System.out.println(gson.toJson(sinceUntilSample));
}
//当version <4时，结果：{"until":"until"}
//当version >=4 && version <5时，结果：{"since":"since","until":"until"}
//当version >=5时，结果：{"since":"since"}
```

###### 基于访问修饰符

```java
ModifierSample modifierSample = new ModifierSample();
Gson gson = new GsonBuilder()
        .excludeFieldsWithModifiers(Modifier.FINAL, Modifier.STATIC, Modifier.PRIVATE)
        .create();
System.out.println(gson.toJson(modifierSample));
// 结果：{"publicField":"public","protectedField":"protected","defaultField":"default"}
```

###### 基于策略（自定义规则）

基于策略是利用Gson提供的`ExclusionStrategy`接口，同样需要使用`GsonBuilder`,相关API 2个，分别是`addSerializationExclusionStrategy` 和`addDeserializationExclusionStrategy` 分别针对序列化和反序化时。

```java
Gson gson = new GsonBuilder()
        .addSerializationExclusionStrategy(new ExclusionStrategy() {
            @Override
            public boolean shouldSkipField(FieldAttributes f) {
                // 这里作判断，决定要不要排除该字段,return true为排除
                if ("finalField".equals(f.getName())) return true; //按字段名排除
                Expose expose = f.getAnnotation(Expose.class); 
                if (expose != null && expose.deserialize() == false) return true; //按注解排除
                return false;
            }
            @Override
            public boolean shouldSkipClass(Class<?> clazz) {
                // 直接排除某个类 ，return true为排除
                return (clazz == int.class || clazz == Integer.class);
            }
        })
        .create();
```

#### POJO与JSON的字段映射规则

`GsonBuilder`提供了`FieldNamingStrategy`接口和`setFieldNamingPolicy`和`setFieldNamingStrategy` 两个方法。
`GsonBuilder.setFieldNamingStrategy` 方法需要与Gson提供的`FieldNamingStrategy`接口配合使用，用于实现将POJO的字段与JSON的字段相对应。上面的`FieldNamingPolicy`实际上也实现了`FieldNamingStrategy`接口，也就是说`FieldNamingPolicy`也可以使用`setFieldNamingStrategy`方法。

```java
Gson gson = new GsonBuilder()
        .setFieldNamingStrategy(new FieldNamingStrategy() {
            @Override
            public String translateName(Field f) {
                //实现自己的规则
                return null;
            }
        })
        .create();
```

**注意：** `@SerializedName`注解拥有最高优先级，在加有`@SerializedName`注解的字段上`FieldNamingStrategy`不生效！

#### TypeAdapter

`TypeAdapter` 是Gson自2.0（源码注释上说的是2.1）开始版本提供的一个抽象类，用于**接管某种类型的序列化和反序列化过程**，包含两个注要方法 `write(JsonWriter,T)` 和 `read(JsonReader)` 其它的方法都是`final`方法并最终调用这两个抽象方法。

```java
public abstract class TypeAdapter<T> {
    public abstract void write(JsonWriter out, T value) throws IOException;
    public abstract T read(JsonReader in) throws IOException;
    //其它final 方法就不贴出来了，包括`toJson`、`toJsonTree`、`toJson`和`nullSafe`方法。
}
```

**注意：**TypeAdapter 以及 JsonSerializer 和 JsonDeserializer 都需要与 `GsonBuilder.registerTypeAdapter` 示或`GsonBuilder.registerTypeHierarchyAdapter`配合使用。

使用示例：

```java
User user = new User("怪盗kidou", 24);
user.emailAddress = "ikidou@example.com";
Gson gson = new GsonBuilder()
        //为User注册TypeAdapter
        .registerTypeAdapter(User.class, new UserTypeAdapter())
        .create();
System.out.println(gson.toJson(user));
```

UserTypeAdapter的定义：

```java
public class UserTypeAdapter extends TypeAdapter<User> {

    @Override
    public void write(JsonWriter out, User value) throws IOException {
        out.beginObject();
        out.name("name").value(value.name);
        out.name("age").value(value.age);
        out.name("email").value(value.email);
        out.endObject();
    }

    @Override
    public User read(JsonReader in) throws IOException {
        User user = new User();
        in.beginObject();
        while (in.hasNext()) {
            switch (in.nextName()) {
                case "name":
                    user.name = in.nextString();
                    break;
                case "age":
                    user.age = in.nextInt();
                    break;
                case "email":
                case "email_address":
                case "emailAddress":
                    user.email = in.nextString();
                    break;
            }
        }
        in.endObject();
        return user;
    }
}
```

当`User.class` 注册了 `TypeAdapter`之后，只要是操作`User.class` 那@SerializedName` 、`FieldNamingStrategy`、`Since`、`Until`、`Expos`都失去了效果，只会调用`UserTypeAdapter.write(JsonWriter, User)` 方法。

#### JsonSerializer与JsonDeserializer

JsonSerializer 和JsonDeserializer 不用像TypeAdapter一样，必须要实现序列化和反序列化的过程，可以据需要选择，如只接管序列化的过程就用 JsonSerializer ，只接管反序列化的过程就用 JsonDeserializer 。

```java
Gson gson = new GsonBuilder()
        .registerTypeAdapter(Integer.class, new JsonDeserializer<Integer>() {
            @Override
            public Integer deserialize(JsonElement json, Type typeOfT, JsonDeserializationContext context) throws JsonParseException {
                try {
                    return json.getAsInt();
                } catch (NumberFormatException e) {
                    return -1;
                }
            }
        })
        .create();
System.out.println(gson.toJson(100)); //结果：100
System.out.println(gson.fromJson("\"\"", Integer.class)); //结果-1
```

注：`registerTypeAdapter`必须使用包装类型，所以`int.class`,`long.class`,`float.class`和`double.class`是行不通的。同时不能使用父类来替上面的子类型。

**registerTypeAdapter与registerTypeHierarchyAdapter的区别：**

|          | registerTypeAdapter | registerTypeHierarchyAdapter |
| -------- | ------------------- | ---------------------------- |
| 支持泛型 | 是                  | 否                           |
| 支持继承 | 否                  | 是                           |

注：如果一个被序列化的对象本身就带有泛型，且注册了相应的`TypeAdapter`，那么必须调用`Gson.toJson(Object,Type)`，明确告诉Gson对象的类型。

#### TypeAdapterFactory

TypeAdapterFactory,见名知意，用于创建TypeAdapter的工厂类，通过对比`Type`，确定有没有对应的`TypeAdapter`，没有就返回null，与`GsonBuilder.registerTypeAdapterFactory`配合使用。

```java
Gson gson = new GsonBuilder()
    .registerTypeAdapterFactory(new TypeAdapterFactory() {
        @Override
        public <T> TypeAdapter<T> create(Gson gson, TypeToken<T> type) {
            return null;
        }
    })
    .create();
```

#### @JsonAdapter注解

`JsonAdapter`相较之前介绍的`SerializedName` 、`FieldNamingStrategy`、`Since`、`Until`、`Expos`这几个注解都是比较特殊的，其它的几个都是用在POJO的字段上，而这一个是用在POJO类上的，接收一个参数，且必须是`TypeAdpater`，`JsonSerializer`或`JsonDeserializer`这三个其中之一。

```java
@JsonAdapter(UserTypeAdapter.class) //加在类上
public class User {
    public User() {
    }
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    public User(String name, int age, String email) {
        this.name = name;
        this.age = age;
        this.email = email;
    }
    public String name;
    public int age;
    @SerializedName(value = "emailAddress")
    public String email;
}
```

使用时不用再使用 `GsonBuilder`去注册`UserTypeAdapter`了。
 **注：** `@JsonAdapter` 仅支持 `TypeAdapter`或`TypeAdapterFactory`( 2.7开始已经支持 JsonSerializer/JsonDeserializer)

```java
Gson gson = new Gson();
User user = new User("怪盗kidou", 24, "ikidou@example.com");
System.out.println(gson.toJson(user));
//结果：{"name":"怪盗kidou","age":24,"email":"ikidou@example.com"}
//为区别结果，特意把email字段与@SerializedName注解中设置的不一样
```

**注意：**`JsonAdapter`的优先级比`GsonBuilder.registerTypeAdapter`的优先级更高。

#### TypeAdapter与 JsonSerializer、JsonDeserializer对比

|            | TypeAdapter            | JsonSerializer、JsonDeserializer   |
| ---------- | ---------------------- | ---------------------------------- |
| 引入版本   | 2.0                    | 1.x                                |
| Stream API | 支持                   | 不支持*，需要提前生成`JsonElement` |
| 内存占用   | 小                     | 比`TypeAdapter`大                  |
| 效率       | 高                     | 比`TypeAdapter`低                  |
| 作用范围   | 序列化 **和** 反序列化 | 序列化 **或** 反序列化             |

