---
layout:     post
title:      "debug记录:SpringBoot中jackson自动配置失效"
subtitle:   "项目中有多个MappingJackson2HttpMessageConverter对象导致jackson自动配置失效的一次debug"
date:       2018-10-31
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - Java
    - java
    - Debug
    - debug
    - Bug
    - bug
    - SpringBoot
    - springBoot
    - Spring Boot
    - spring boot
    - MappingJackson2HttpMessageConverter
    - mappingJackson2HttpMessageConverter
    - Configuration
    - configuration
---

[Spring序列化与反序列化设计探究](https://caotc.org//2018/10/27/java-spring-serialize-deserialize-design/)

在我如上这篇博客中,遇到了对SpringBootMvc环境的项目中,试图使用将Module对象托管给Spring容器托管来达成MappingJackson2HttpMessageConverter对象的自动全局配置.

结果失败,发现在返回的HttpMessageConverter列表中和容器中各自存在一个不同的MappingJackson2HttpMessageConverter对象,怀疑可能是这个原因导致了自动全局配置失效.

今天debug的目的就是搞清楚这个现象的原因,以及jackson自动全局配置失效的原因是否就是这个.

首先,采用断点的方式,搞清楚,这两个MappingJackson2HttpMessageConverter对象中是否有哪个对象中jackson自动全局配置生效了.
```java
@Configuration
public class FormatterConfig extends WebMvcConfigurerAdapter {
  @Resource
  private LocalDateTimeFormatter localDateTimeFormatter;
  @Resource
  private MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter;

  @Bean
  public SimpleModule simpleModule() {
    return new SimpleModule().addDeserializer(LocalDateTime.class,
        new JsonDeserializerAdapter<>(LocalDateTime.class, localDateTimeFormatter));
  }

  @Override
  public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    System.out.println(converters);
  }
}
```
定义一个自定义的LocalDateTimeFormatter,并使用适配器包装成JsonDeserializer,将其注册到SimpleModule对象中并将该对象交给Spring容器托管.
然后在extendMessageConverters断点查看情况:
![Spring容器中的MappingJackson2HttpMessageConverter](/img/in-post/2018-10-31-java-debug-spring-multiple-http-message-converter/spring-container-jackson-http-message-converter.png)
![extendMessageConverters方法中列表里的MappingJackson2HttpMessageConverter](/img/in-post/2018-10-31-java-debug-spring-multiple-http-message-converter/list-jackson-http-message-converter.png)
可见在Spring容器中的MappingJackson2HttpMessageConverter应用了自动全局配置而列表中的没有.jackson自动全局配置失效的原因确实就是因为这个.

接下来,将断点定在MappingJackson2HttpMessageConverter对象的构造函数中,查看这两个对象分别是什么时候被创建加载的.
最后定位结果为Spring容器中的对象,是首先产生的,是在org.springframework.boot.autoconfigure.web.JacksonHttpMessageConvertersConfiguration中的mappingJackson2HttpMessageConverter方法产生的:
```java
@Configuration
class JacksonHttpMessageConvertersConfiguration {

	@Configuration
	@ConditionalOnClass(ObjectMapper.class)
	@ConditionalOnBean(ObjectMapper.class)
	@ConditionalOnProperty(name = HttpMessageConvertersAutoConfiguration.PREFERRED_MAPPER_PROPERTY, havingValue = "jackson", matchIfMissing = true)
	protected static class MappingJackson2HttpMessageConverterConfiguration {

		@Bean
		@ConditionalOnMissingBean(value = MappingJackson2HttpMessageConverter.class, ignoredType = {
				"org.springframework.hateoas.mvc.TypeConstrainedMappingJackson2HttpMessageConverter",
				"org.springframework.data.rest.webmvc.alps.AlpsJsonHttpMessageConverter" })
		public MappingJackson2HttpMessageConverter mappingJackson2HttpMessageConverter(
				ObjectMapper objectMapper) {
			return new MappingJackson2HttpMessageConverter(objectMapper);
		}

	}
  //......
}
```
这应该是没有问题的,这也是官方的类.
而HttpMessageConverter列表中的对象是之后产生的,定位结果为公司封装框架的自动配置类,因此这部分代码做了一些处理:
```java
@Configuration
public class MyFastJsonHttpMessageConverterAutoConfiguration {
    @Bean
    @ConditionalOnMissingBean({MyFastJsonHttpMessageConverter.class})
    public HttpMessageConverters myFastJsonHttpMessageConverter() {
        //..省略不相关内容
      return new HttpMessageConverters(new HttpMessageConverter[]{myFastJsonHttpMessageConverter});
    }  
}
```
这个方法自己new了一个HttpMessageConverters对象,并且只加了一个自定义的FastJsonHttpMessageConverter进去.
查看HttpMessageConverters类的注释:
> Bean used to manage the {@link HttpMessageConverter}s used in a Spring Boot
> application. Provides a convenient way to add and merge additional
> {@link HttpMessageConverter}s to a web application.
> <p>
> An instance of this bean can be registered with specific
> {@link #HttpMessageConverters(HttpMessageConverter...) additional converters} if
> needed, otherwise default converters will be used.
> <p>
> NOTE: The default converters used are the same as standard Spring MVC (see
> {@link WebMvcConfigurationSupport#getMessageConverters} with some slight re-ordering to
> put XML converters at the back of the list.

注释说明这是一个管理所有HttpMessageConverter对象的类,是组合模式的体现.

我认为如果公司封装框架中没有创建,默认情况下,Spring一定会创建这个对象.果然定位到org.springframework.boot.autoconfigure.web.HttpMessageConvertersAutoConfiguration类中也有messageConverters方法创建该对象:
```java
@Configuration
@ConditionalOnClass(HttpMessageConverter.class)
@AutoConfigureAfter({ GsonAutoConfiguration.class, JacksonAutoConfiguration.class })
@Import({ JacksonHttpMessageConvertersConfiguration.class,
		GsonHttpMessageConvertersConfiguration.class })
public class HttpMessageConvertersAutoConfiguration {

	static final String PREFERRED_MAPPER_PROPERTY = "spring.http.converters.preferred-json-mapper";

	private final List<HttpMessageConverter<?>> converters;

	public HttpMessageConvertersAutoConfiguration(
			ObjectProvider<List<HttpMessageConverter<?>>> convertersProvider) {
		this.converters = convertersProvider.getIfAvailable();
	}

	@Bean
	@ConditionalOnMissingBean
	public HttpMessageConverters messageConverters() {
		return new HttpMessageConverters(this.converters != null ? this.converters
				: Collections.<HttpMessageConverter<?>>emptyList());
	}
  //......
}
```
但是两者的代码有明显的不同,在该方法中以成员变量this.converters作为参数传入了构造器中,这个变量看起来应该就会是其他地方所有配置产生的HttpMessageConverter对象.

经过debug,果然如果将公司框架的配置类禁用掉,这里将会把JacksonHttpMessageConvertersConfiguration中创建的MappingJackson2HttpMessageConverter和StringHttpMessageConverter放入this.converters中.
然后作为参数传入HttpMessageConverters的构造函数中,HttpMessageConverters将会保证自定义的同类型的HttpMessageConverter排在同类型的默认HttpMessageConverter前面.

而公司封装框架的配置类中,没有读取当前被Spring容器托管的HttpMessageConverter传入HttpMessageConverters的构造函数中,自然jackson自动全局配置就失效了.

试着禁用掉公司封装框架的配置类,按照Spring的官方文档,不自己创建HttpMessageConverters对象,而是只把MyFastJsonHttpMessageConverter对象交给Spring容器托管,果然jackson自动全局配置就生效了.
并且MyFastJsonHttpMessageConverter依旧注册到HttpMessageConverter列表中,并且排在首位.
以下为改动后的代码
```java
@Configuration
public class MyFastJsonHttpMessageConverterAutoConfiguration {
  @Bean
  @ConditionalOnMissingBean({MyFastJsonHttpMessageConverter.class})
  public MyFastJsonHttpMessageConverter myFastJsonHttpMessageConverter() {
        //..省略不相关内容
    return myFastJsonHttpMessageConverter;
  }
}
```

本次debug的目的到此完成.
附带说明一下,HttpMessageConverters对象中创建的默认MappingJackson2HttpMessageConverter对象是永远不会注册到Spring容器中的,默认对象是相当于一个兜底的存在.
凡是注册到Spring容器中HttpMessageConverters对象,都认为是自定义的存在,即使是其他Springboot的配置类加载的.
而且由于同类型的HttpMessageConverter对象对支持的mediaType设置不一定相同,所以Spring并不会删除同类型的默认HttpMessageConverter对象,只会把同类型中的自定义对象放在最前,保证用户自定义的优先级.
相关说明可参见[MappingJackson2HttpMessageConverter is not singleton?](https://github.com/spring-projects/spring-boot/issues/15027)