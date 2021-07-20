---
layout:     post
title:      "SSM中的类型转换、格式化、序列化与反序列化"
subtitle:   "SSM中的类型转换、格式化、序列化与反序列化"
date:       2018-12-27
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - Java
    - java
    - SSM
    - ssm
    - Bug
    - bug
    - Convert
    - convert
    - Formatter
    - formatter
    - Serializer
    - serializer
    - Deserializer
    - deserializer
---

# 定义与区别
类型转换、格式化、序列化与反序列化三个概念是相似而且经常相关,联系在一起的.
- **类型转换(convert)：将A类型的源对象转换为目标B类型的对象.**
- **格式化(format):将一个对象输出为显示的格式或者说容易让人理解的格式.以及把从显示的格式解析为对象.**
- **序列化(serializer)与反序列化(deserializer):序列化是指把对象转换为二进制化数据.反序列化是指把二进制化数据转换为对象.通常涉及到使用流进行IO传输.**

# 一.Spring中的类型转换与格式化
Spring中其实存在两套类型转换与格式化的设计.
## (一). Spring中的PropertyEditor
Spring中的第一套类型转换与格式化的设计依赖的接口是java.beans.PropertyEditor.从所在的包名可以看出,这个接口实际上是sun制定的awt规范的一部分.

```java
public interface PropertyEditor {
    void setValue(Object value);
    Object getValue();
    boolean isPaintable();
    void paintValue(java.awt.Graphics gfx, java.awt.Rectangle box);
    String getJavaInitializationString();
    String getAsText();
    void setAsText(String text) throws java.lang.IllegalArgumentException;
    String[] getTags();
    java.awt.Component getCustomEditor();
    boolean supportsCustomEditor();
    void addPropertyChangeListener(PropertyChangeListener listener);
    void removePropertyChangeListener(PropertyChangeListener listener);
}
```
- Object getValue()：返回属性的当前值。基本类型被封装成对应的包装类实例；
- void setValue(Object newValue)：设置属性的值，基本类型以包装类传入（自动装箱）；
- String getAsText()：将属性对象用一个字符串表示，以便外部的属性编辑器能以可视化的方式显示。缺省返回null，表示该属性不能以字符串表示；
- void setAsText(String text)：用一个字符串去更新属性的内部值，这个字符串一般从外部属性编辑器传入；

PropertyEditor实际上是设计为一个javaBean对象的属性的编辑器,所以才叫PropertyEditor
而Spring则使用getAsText和setAsText两个方法将其作为各类型与字符串类型的转换器使用.

在Spring中使用的大致的逻辑如下图:
![image](/img/in-post/2018-12-27-java-ssm-convert-formatter-serializer-deserializer/property-editor-flow.jpg)
而这个接口的设计问题主要有如下几点:

1.  **从setAsText和getAsText两个方法名就能看出,其只支持String<–>Object之间的互相转换.**如果要进行其他类型之间的转换就比较麻烦,甚至没法做到. 比如Long时间戳转换为可用的时间类型时还能通过把Long转成字符串来间接实现.如果是反过来的话,时间类型的字符串显然是没法转换为Long时间戳的.
2. **PropertyEditor是线程不安全的(这里说的是Spring的默认实现类,具体的自定义实现类取决于自定义代码)**，也就是有状态的，因此每次使用时都需要创建一个,比较影响性能. 这是一个类似SimpleDateFormat一样的设计失误.
3. **没有泛型带来的类型约束,需要自己代码判断,容易出错.**
4. **粒度太粗,只能到类型为止,同一类型的不同格式化不能满足.**


## (二). Spring中的Converter SPI和Formatter SPI
为了弥补这些问题,Spring3中引入了新的接口设计来取代PropertyEditor接口.

网络大部分博客说是Converter接口取代了PropertyEditor接口,实际上不太正确,应该说是Formatter接口取代了PropertyEditor接口, 或者说Formatter接口和Converter接口各自取代了PropertyEditor接口的一部分职责,将类型转换与格式化的职责分开了.

**Converter及相关接口被设计为通用的类型转换器,取代了PropertyEditor接口的类型转换职责.**
```java
/**
 * A converter converts a source object of type {@code S} to a target of type {@code T}.
 *
 * <p>Implementations of this interface are thread-safe and can be shared.
 *
 * <p>Implementations may additionally implement {@link ConditionalConverter}.
 *
 * @author Keith Donald
 * @since 3.0
 * @param <S> the source type
 * @param <T> the target type
 */
public interface Converter<S, T> {

	/**
	 * Convert the source object of type {@code S} to target type {@code T}.
	 * @param source the source object to convert, which must be an instance of {@code S} (never {@code null})
	 * @return the converted object, which must be an instance of {@code T} (potentially {@code null})
	 * @throws IllegalArgumentException if the source cannot be converted to the desired target type
	 */
	T convert(S source);

}
```
与PropertyEditor的比较如下:

1. **改进为能够实现任何两种类型之间的转换.**
2. **要求实现线程安全,自然Spring的默认实现类全部都实现了线程安全.**
3. **使用了泛型约束.**
4. **同样只能到类型为止.**

**Formatter及相关接口被设计为取代了PropertyEditor接口的序列化和反序列化为字符串,或者说格式化和解析功能的职责.**

```java
/**
 * Formats objects of type T.
 * A Formatter is both a Printer <i>and</i> a Parser for an object type.
 *
 * @author Keith Donald
 * @since 3.0
 * @param <T> the type of object this Formatter formats
 */
public interface Formatter<T> extends Printer<T>, Parser<T> {

}
/**
 * Prints objects of type T for display.
 *
 * @author Keith Donald
 * @since 3.0
 * @param <T> the type of object this Printer prints
 */
public interface Printer<T> {

	/**
	 * Print the object of type T for display.
	 * @param object the instance to print
	 * @param locale the current user locale
	 * @return the printed text string
	 */
	String print(T object, Locale locale);

}
/**
 * Parses text strings to produce instances of T.
 *
 * @author Keith Donald
 * @since 3.0
 * @param <T> the type of object this Parser produces
 */
public interface Parser<T> {

	/**
	 * Parse a text String to produce a T.
	 * @param text the text string
	 * @param locale the current user locale
	 * @return an instance of T
	 * @throws ParseException when a parse exception occurs in a java.text parsing library
	 * @throws IllegalArgumentException when a parse exception occurs
	 */
	T parse(String text, Locale locale) throws ParseException;

}
```

与PropertyEditor的比较如下:
1. **同样只能实现String<–>Object之间的互相转换.**
2. **要求实现线程安全,自然Spring的默认实现类全部都实现了线程安全.**
3. **使用了泛型约束.**
4. **更细粒度的控制,提供了注解方式方便属性或者说字段级别的粒度,如果通过编程方式控制的话,还能有Local对象提供地区相关信息,能过做到按地区实现多种格式化.**

## (三). 使用范围
Spring内部几乎所有需要用到类型转换和格式化的地方用的都是上面说的这两套设计,**包括从配置文件读取配置、get方法时读取url中的参数、读取http请求的header属性值等**,所以才放在Spring.core这个核心包中,当然使用的时候肯定是统一到一个代理对象管理使用的.

# 二.SpringMvc中的序列化与反序列化
其实Spring中有org.springframework.core.serializer.Serializer和org.springframework.core.serializer.Deserializer两个序列化与反序列化接口,不过这两个接口实际上就是它自己内部用的东西和使用者关系不大,所以就略过不提.

这里主要说和web开发者关系比较密切的Http请求中的序列化与反序列化.


```java
/**
 * Represents the base interface for HTTP request and response messages.
 * Consists of {@link HttpHeaders}, retrievable via {@link #getHeaders()}.
 *
 * @author Arjen Poutsma
 * @since 3.0
 */
public interface HttpMessage {

	/**
	 * Return the headers of this message.
	 * @return a corresponding HttpHeaders object (never {@code null})
	 */
	HttpHeaders getHeaders();

}
/**
 * Represents an HTTP input message, consisting of {@linkplain #getHeaders() headers}
 * and a readable {@linkplain #getBody() body}.
 *
 * <p>Typically implemented by an HTTP request handle on the server side,
 * or an HTTP response handle on the client side.
 *
 * @author Arjen Poutsma
 * @since 3.0
 */
public interface HttpInputMessage extends HttpMessage {

	/**
	 * Return the body of the message as an input stream.
	 * @return the input stream body (never {@code null})
	 * @throws IOException in case of I/O errors
	 */
	InputStream getBody() throws IOException;

}
/**
 * Represents an HTTP output message, consisting of {@linkplain #getHeaders() headers}
 * and a writable {@linkplain #getBody() body}.
 *
 * <p>Typically implemented by an HTTP request handle on the client side,
 * or an HTTP response handle on the server side.
 *
 * @author Arjen Poutsma
 * @since 3.0
 */
public interface HttpOutputMessage extends HttpMessage {

	/**
	 * Return the body of the message as an output stream.
	 * @return the output stream body (never {@code null})
	 * @throws IOException in case of I/O errors
	 */
	OutputStream getBody() throws IOException;

}
```
上面是Spring定义的Http信息的接口,HttpInputMessage和HttpOutputMessage的具体Body信息都是二进制化在IO流中的.

而**Spring对于HttpMessage序列化和反序列化处理的接口是org.springframework.http.converter.HttpMessageConverter.**

```java
/**
 * Strategy interface that specifies a converter that can convert from and to HTTP requests and responses.
 *
 * @author Arjen Poutsma
 * @author Juergen Hoeller
 * @since 3.0
 */
public interface HttpMessageConverter<T> {

	/**
	 * Indicates whether the given class can be read by this converter.
	 * @param clazz the class to test for readability
	 * @param mediaType the media type to read (can be {@code null} if not specified);
	 * typically the value of a {@code Content-Type} header.
	 * @return {@code true} if readable; {@code false} otherwise
	 */
	boolean canRead(Class<?> clazz, MediaType mediaType);

	/**
	 * Indicates whether the given class can be written by this converter.
	 * @param clazz the class to test for writability
	 * @param mediaType the media type to write (can be {@code null} if not specified);
	 * typically the value of an {@code Accept} header.
	 * @return {@code true} if writable; {@code false} otherwise
	 */
	boolean canWrite(Class<?> clazz, MediaType mediaType);

	/**
	 * Return the list of {@link MediaType} objects supported by this converter.
	 * @return the list of supported media types
	 */
	List<MediaType> getSupportedMediaTypes();

	/**
	 * Read an object of the given type from the given input message, and returns it.
	 * @param clazz the type of object to return. This type must have previously been passed to the
	 * {@link #canRead canRead} method of this interface, which must have returned {@code true}.
	 * @param inputMessage the HTTP input message to read from
	 * @return the converted object
	 * @throws IOException in case of I/O errors
	 * @throws HttpMessageNotReadableException in case of conversion errors
	 */
	T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;

	/**
	 * Write an given object to the given output message.
	 * @param t the object to write to the output message. The type of this object must have previously been
	 * passed to the {@link #canWrite canWrite} method of this interface, which must have returned {@code true}.
	 * @param contentType the content type to use when writing. May be {@code null} to indicate that the
	 * default content type of the converter must be used. If not {@code null}, this media type must have
	 * previously been passed to the {@link #canWrite canWrite} method of this interface, which must have
	 * returned {@code true}.
	 * @param outputMessage the message to write to
	 * @throws IOException in case of I/O errors
	 * @throws HttpMessageNotWritableException in case of conversion errors
	 */
	void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;

}
```
**该接口实现的是从二进制到指定Java类型的转换.我们最常用的其实现为MappingJackson2HttpMessageConverter和FastJsonHttpMessageConverter, 就是将该序列化和反序列化过程交给了jackson和fastJson来处理.**

对于这个接口的实现类其实可以分为两类.

**1.在二进制序列化协议中,是直接将IOStream的二进制数据直接转换到目标Java类型的,期间根本没有任何Java类型之间的转换.比如jdk自带的基于Serializable的序列化与反序列化.**

**2.另一类序列化协议则并不基于二进制,而是基于字符串.如json和xml协议.这一类的协议在序列化和反序列化期间是会发生字符串与目标java类型的相互转换的.**

但是对于SpringMvc来说,它只知道序列化和反序列化是二进制到java对象之间的转换,并不知道基于字符串的序列化协议经过了字符串这样的中间态.

**所以即使基于字符串的序列化协议在序列化和反序列化过程中发生了字符串与目标java类型的相互转换也是和Spring.core的那套Converter SPI和Formatter SPI是完全无关的.**

**因此想要在类似json,xml等以字符串格式的序列化协议中进行自定义的序列化与反序列化格式,需要去修改具体HttpMessageConverter实现类的逻辑, 需要委托的json库合xml库等支持相关的配置,修改注册在Spring容器中的配置,否则只能自己实现HttpMessageConverter并替代原实现类才能做到.**

# 三.Mybatis的类型转换
在Web开发中还有一块涉及到类型转换,那就是ORM框架中java其他类型与java.sql包中定义的数据库类型转换.

**以mybatis为例,内部使用的是org.apache.ibatis.type.TypeHandler接口来处理这部分的转换关系.**

当然平时可能会觉得这跟我们无关,所有的类型转换都已经被mybatis封装好了.

其实这个机制可以让我做到一些更便利的开发.
比如说,身份证号我们通常都是定义为String类型在DO中的.
然后有时候我们要从身份证号中获取一些别的信息,比如说身份证中存了人的性别信息,存了人的省市区信息.
我们通常会使用一个工具类去从身份证号中获取.

```java
@Data
public class TestDO{
    Long id;
    String idcard;
}

public class Test {

    @Test
    public void test(){
        TestDO testDO=new TestDO();
        String provinceCode = IdcardUtils.getProvinceCode(testDO.getIdcard());
    }
}
```
然而实际上我们可以换一种办法.直接把Idcrad封装成一个类,然后把IdcardUtils的逻辑内化到身份证类中去.
示例代码如下:

```java
@Value
public class Idcrad {
    String number;

    public String getProvinceCode(){
        return number.substring(0,2);
    }
}

@Data
public class TestDO{
    Long id;
    Idcrad idcard;
    
    public String getProvinceCode(){
        return idcard.getProvinceCode();
    }
}

public final class IdcradTypeHandler extends
    BaseTypeHandler<Idcrad> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Idcrad parameter,
      JdbcType jdbcType)
      throws SQLException {
    ps.setString(i, parameter.getNumber());
  }

  @Override
  public Idcrad getNullableResult(ResultSet rs, String columnName)
      throws SQLException {
    return new Idcrad(rs.getString(columnName));
  }

  @Override
  public Idcrad getNullableResult(ResultSet rs, int columnIndex)
      throws SQLException {
    return new Idcrad(rs.getString(columnIndex));
  }

  @Override
  public Idcrad getNullableResult(CallableStatement cs, int columnIndex)
      throws SQLException {
    return new Idcrad(cs.getString(columnIndex));
  }
}
```
然后在配置文件中加上这样一行配置:
mybatis.typeHandlersPackage=xxxxxxxxxx
配置mybatis扫描handler就行了.

甚至如果使用这种方法的话,某些数据库中没有被筛选需求的,纯粹为了展示用的字段,就可以直接删掉.

比如说我们经常会记录某个对象的ProvinceCode和ProvinceName.
实际上用这种思路考虑,如果没有对ProvinceName有筛选需求,仅仅是考虑展示用的话.
我们可以这样设计:
```java
@Value
public class Area {
    String code;
    String name;
    String fullName;
    ......
}

@Data
public class TestDO{
    Long id;
    Area province;
}

public final class IdcradTypeHandler extends
    BaseTypeHandler<Idcrad> {

  @Override
  public void setNonNullParameter(PreparedStatement ps, int i, Idcrad parameter,
      JdbcType jdbcType)
      throws SQLException {
    ps.setString(i, parameter.getCode());
  }

  @Override
  public Idcrad getNullableResult(ResultSet rs, String columnName)
      throws SQLException {
    return cache.getAreaByCode(rs.getString(columnName));
  }

  @Override
  public Idcrad getNullableResult(ResultSet rs, int columnIndex)
      throws SQLException {
    return cache.getAreaByCode(rs.getString(columnIndex));
  }

  @Override
  public Idcrad getNullableResult(CallableStatement cs, int columnIndex)
      throws SQLException {
    return cache.getAreaByCode(cs.getString(columnIndex));
  }
}
```

当然对于某些比较特殊的类型,比如枚举,就不是直接定义了TypeHandler的扫描就可以了,仅仅这样无法覆盖内置的枚举转换器的优先级.
而是需要用mybatis.configuration.defaultEnumTypeHandler=xxxx
来配置使用的全局枚举转换器.
但是这个配置是mybatis3.5版本才开始有的.
如果是低版本的mybaits,一般做法是自定义一个接口,让所有枚举实现这个接口,然后拦截和定义的是这个接口的TypeHandler.


# 四.全局自定义类型转换、格式化、序列化与反序列化
![image](/img/in-post/2018-12-27-java-ssm-convert-formatter-serializer-deserializer/springmvc-serialize-deserialize.png)

从上图我们可以看出,想要实现对一个类型的全局统一自定义序列化与反序列化,需要做到以下两点:

1. **实现Spring-core的PropertyEditor\Formatter\Converter接口并注册到Spring中,实现对Url和Header的自定义解析.**
2. **对于所有项目支持的序列化标准为字符串的HttpMessageConverter中修改其配置,加入全局的自定义序列化与反序列化.**
3. **如果涉及到的话,还要修改ORM框架的类型转换**

之所以说要修改所有基于字符串的HttpMessageConverter的配置,**SpringMvc是天然支持同一个接口代码同时支持多套序列化与反序列化协议的, 其会根据一定的规则（主要是Content-Type、Accept、controller方法的consumes/produces、Converter.mediaType以及Converter的排列顺序这四个属性）来自动匹配对应的HttpMessageConverter.**

而SpringMvc在自动帮开发者完成了序列化与反序列化操作时,屏蔽了序列化与反序列化协议中的不同解析逻辑后,**开发者只需要实现一套业务逻辑,即可自动支持多种序列化标准的请求和响应.**

这样说可能不好理解,看下面几张图就能明白了.
![image](/img/in-post/2018-12-27-java-ssm-convert-formatter-serializer-deserializer/请求和返回均json.png)
![image](/img/in-post/2018-12-27-java-ssm-convert-formatter-serializer-deserializer/请求和返回均java序列化.png)
甚至请求和返回可以使用不同的序列化协议,当然一般不会这样做,太奇葩了.
![image](/img/in-post/2018-12-27-java-ssm-convert-formatter-serializer-deserializer/请求java序列化返回json.png)
![image](/img/in-post/2018-12-27-java-ssm-convert-formatter-serializer-deserializer/请求json返回java序列化.png)

所以要做到真正的全局自定义,必须修改所有基于字符串的HttpMessageConverter的配置.