---
layout:     post
title:      "Spring序列化与反序列化设计探究"
subtitle:   "从自定义数据序列化与反序列化格式探究Spring相关设计"
date:       2018-10-27
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - Java
    - java
    - Spring
    - spring
    - SpringMvc
    - springMvc
    - HttpMessageConverter
    - httpMessageConverter
    - Serialize
    - serialize
    - 序列化
    - Deserialize
    - deserialize
    - 反序列化
    - custom format
    - 自定义格式
    - type convert
    - 类型转换
    - Converter
    - converter
    - Formatter
    - formatter
---
# 一.探究原因和需求概述
最近在项目中遇到了需要在**SpringMvc中对枚举类型和java8新时间类型的自定义序列化与反序列化格式**的问题.

简单说就是希望作为返回对象的DTO中,对于枚举类型和java8新时间类型的属性,定义为对应的枚举类型和java8新时间类型.

但是由于对接和展示需要或者规范规定,很多时候需要序列化和反序列化时自定义格式.比如说很多人习惯把时间格式定为yyyy-MM-dd HH:mm:ss的字符串,把枚举格式定义为int型的code.

对于这种需求,最容易想到的解决办法是在接收的RequestParam对象和输出的DTO对象中,定义属性类型为自定义格式.比如原本应该为LocalDateTime型的属性定义为String类型,原本为枚举类型的属性定义为Integer类型,然后在转换器中手动实现转换自定义格式的转换逻辑.

对于这种做法,个人不太喜欢.

首先是代码量增加了不少,每个有需要自定义格式的属性都需要手动处理.而统一处理方式又是必要的,否则如果哪天要更改自定义格式那就麻烦了,只能靠规范规定和code review来约束保证统一处理.而且很明显这些都是冗余代码.

更重要的是其实这并不符合SpringMvc帮开发者屏蔽网络报文到业务对象的转换逻辑,让开发者专心业务逻辑的原则.对于开发者来说,自己拿到的业务对象应该就是自己需要的类型,而不应该自己再手动进行转换逻辑.

# 二.源码探究过程
所以下面就来带着这样的需求和想法来探究一下SpringMvc框架的序列化与反序列化设计.

既然要探究SpringMvc框架的设计,那么自然要从众所周知的DispatcherServlet,HandlerMapping,HandlerAdapter入手.

显然HandlerMapping是处理请求映射的，处理@RequestMapping跟请求地址之间的关系,与本次探究关系不大.

HandlerAdapter则是请求处理的适配器，也就是请求之后处理具体逻辑的执行，关系到哪个类的哪个方法以及转换器等工作，这是本次探究的入手点,其源码如下:
```java
public interface HandlerAdapter {
	boolean supports(Object handler);
	
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
	
	long getLastModified(HttpServletRequest request, Object handler);
}
```
显然,三个方法即使不看注释也能直接猜测出用途.

supports:判断是否支持传入的Handler

handle:使用Handler对象处理请求

getLastModified:获取最后修改时间

但是具体我们涉及到的是哪些HandlerAdapter接口的实现类,则需要去看所有HandlerAdapter的优先级以及其实现类的注释说明和support方法的具体逻辑.
HandlerAdapter接口的相关继承与实现关系类图如下
![HandlerAdapter diagram](/img/in-post/2018-10-27-java-spring-serialize-deserialize-design/request-mapping-handler-adapter.png)

首先在org.springframework.web.servlet.DispatcherServlet中,initHandlerAdapters方法中对其进行了加载.
```java
public class DispatcherServlet extends FrameworkServlet {
  private void initHandlerAdapters(ApplicationContext context) {
  		this.handlerAdapters = null;
  
  		if (this.detectAllHandlerAdapters) {
  			// Find all HandlerAdapters in the ApplicationContext, including ancestor contexts.
  			Map<String, HandlerAdapter> matchingBeans =
  					BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerAdapter.class, true, false);
  			if (!matchingBeans.isEmpty()) {
  				this.handlerAdapters = new ArrayList<HandlerAdapter>(matchingBeans.values());
  				// We keep HandlerAdapters in sorted order.
  				AnnotationAwareOrderComparator.sort(this.handlerAdapters);
  			}
  		}
  		else {
  			try {
  				HandlerAdapter ha = context.getBean(HANDLER_ADAPTER_BEAN_NAME, HandlerAdapter.class);
  				this.handlerAdapters = Collections.singletonList(ha);
  			}
  			catch (NoSuchBeanDefinitionException ex) {
  				// Ignore, we'll add a default HandlerAdapter later.
  			}
  		}
  
  		// Ensure we have at least some HandlerAdapters, by registering
  		// default HandlerAdapters if no other adapters are found.
  		if (this.handlerAdapters == null) {
  			this.handlerAdapters = getDefaultStrategies(context, HandlerAdapter.class);
  			if (logger.isDebugEnabled()) {
  				logger.debug("No HandlerAdapters found in servlet '" + getServletName() + "': using default");
  			}
  		}
  	}
}
```

显然从代码中可以看出,具体HandlerAdapter的加载种类和顺序与配置有关.

这并不是本次探究重点.所以直接使用debug的办法来探究当前项目加载的HandlerAdapter种类和顺序.

可以看到this.handlerAdapters属性的使用是在getHandlerAdapter方法中.
```java
public class DispatcherServlet extends FrameworkServlet {
	protected HandlerAdapter getHandlerAdapter(Object handler) throws ServletException {
		for (HandlerAdapter ha : this.handlerAdapters) {
			if (logger.isTraceEnabled()) {
				logger.trace("Testing handler adapter [" + ha + "]");
			}
			if (ha.supports(handler)) {
				return ha;
			}
		}
		throw new ServletException("No adapter for handler [" + handler +
				"]: The DispatcherServlet configuration needs to include a HandlerAdapter that supports this handler");
	}
}
```

方法内代码非常简单,就是遍历所有加载的HandlerAdapter,调用其supports方法判断该对象是否支持本次请求的处理,返回第一个支持处理的HandlerAdapter对象.

![handlerAdapters](/img/in-post/2018-10-27-java-spring-serialize-deserialize-design/handlerAdapters.png)

debug中发现本项目加载了RequestMappingHandlerAdapter,HttpRequestHandlerAdapter,SimpleControllerHandlerAdapter三种.

首先查看RequestMappingHandlerAdapter源码和注释,该类注释中并没有说明该适配器的处理范围,其supports方法判断逻辑为handler对象是否为HandlerMethod类型,也让人摸不着头脑.

但是从其名字来看,似乎与常用的注解@RequestMapping有关,于是查看@RequestMapping类的源码,搜索RequestMappingHandlerAdapter,果然在其中找到了如下注释.
> <p><b>NOTE:</b> Spring 3.1 introduced a new set of support classes for
> {@code @RequestMapping} methods in Servlet environments called
> {@code RequestMappingHandlerMapping} and
> {@code RequestMappingHandlerAdapter}. They are recommended for use and
> even required to take advantage of new features in Spring MVC 3.1 (search
> {@literal "@MVC 3.1-only"} in this source file) and going forward.

显然,这段注释告诉我们,只要注解了@RequestMapping,就会被RequestMappingHandlerAdapter匹配处理.

也就是说RequestMappingHandlerAdapter就是平时我们常用的HandlerAdapter了,继续查看该类中handle方法的具体逻辑,结合其属性变量名和类型名称的含义来推测,与本次探究相关的重点如下:
```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
	private List<HandlerMethodArgumentResolver> customArgumentResolvers;

	private HandlerMethodArgumentResolverComposite argumentResolvers;

	private List<HandlerMethodReturnValueHandler> customReturnValueHandlers;

	private HandlerMethodReturnValueHandlerComposite returnValueHandlers;
}
```
显然customArgumentResolvers是自定义参数解析器,argumentResolvers是默认参数解析器,其HandlerMethodArgumentResolverComposite为HandlerMethodArgumentResolver接口的实现类.

同样的,customReturnValueHandlers是自定义返回值处理器,returnValueHandlers是默认返回值处理器,其HandlerMethodReturnValueHandlerComposite为HandlerMethodReturnValueHandler接口的实现类.

查看HandlerMethodArgumentResolver的源码
```java
public interface HandlerMethodArgumentResolver {
  	boolean supportsParameter(MethodParameter parameter);
  	
  	Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
    			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception;
}
```
非常简单,显然supportsParameter方法是判断是否支持该参数类型的转换,而resolveArgument方法则是具体实现参数转换.

再查看HandlerMethodArgumentResolver的实现类,非常多,其中我们能看到很熟悉的类名,如PathVariableMethodArgumentResolver显然是@PathVariable注解的参数解析器,RequestParamMethodArgumentResolver显然是@RequestParam注解的参数解析器,RequestResponseBodyMethodProcessor显然是@RequestBody注解的参数解析器同时是@ResponseBody注解的返回值处理器.

查看RequestMappingHandlerAdapter中的默认参数解析器HandlerMethodArgumentResolverComposite,其注释如下:
> Resolves method parameters by delegating to a list of registered {@link HandlerMethodArgumentResolver}s.
> Previously resolved method parameters are cached for faster lookups.

显然这个类只是一个代理类,提供缓存之类的包装功能,实际逻辑是委托给内部注册的HandlerMethodArgumentResolver集合的.

而注册行为在RequestMappingHandlerAdapter的如下源码中:
```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
  	@Override
  	public void afterPropertiesSet() {
  		// Do this first, it may add ResponseBody advice beans
  		initControllerAdviceCache();
  
  		if (this.argumentResolvers == null) {
  			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
  			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
  		}
  		if (this.initBinderArgumentResolvers == null) {
  			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
  			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
  		}
  		if (this.returnValueHandlers == null) {
  			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
  			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
  		}
  	}
}
```
getDefaultArgumentResolvers方法中初始化了SpringMvc默认支持的参数解析器如上面说的PathVariableMethodArgumentResolver等类,然后加入了自定义参数解析器,返回后一起注册到HandlerMethodArgumentResolverComposite中.

查看argumentResolvers的使用处,主要在invokeHandlerMethod方法中.
```java
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter
		implements BeanFactoryAware, InitializingBean {
  protected ModelAndView invokeHandlerMethod(HttpServletRequest request,
  			HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {
  
  		ServletWebRequest webRequest = new ServletWebRequest(request, response);
  		try {
        //......
  
  			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
  			invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
  			invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
  			invocableMethod.setDataBinderFactory(binderFactory);
  			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);
  			
  			//......
  
  			invocableMethod.invokeAndHandle(webRequest, mavContainer);
        //......
  		}
  		finally {
  			webRequest.requestCompleted();
  		}
  	}
}
```
显然,HandlerMethodArgumentResolver和HandlerMethodReturnValueHandler都是被设置到反射方法执行对象invocableMethod中,然后在invokeAndHandle方法中被使用.

继续查看ServletInvocableHandlerMethod类中invokeAndHandle方法的源码:
```java
public class ServletInvocableHandlerMethod extends InvocableHandlerMethod {
  public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer,
  			Object... providedArgs) throws Exception {
  
  		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
  		setResponseStatus(webRequest);
  
  		if (returnValue == null) {
  			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
  				mavContainer.setRequestHandled(true);
  				return;
  			}
  		}
  		else if (StringUtils.hasText(getResponseStatusReason())) {
  			mavContainer.setRequestHandled(true);
  			return;
  		}
  
  		mavContainer.setRequestHandled(false);
  		try {
  			this.returnValueHandlers.handleReturnValue(
  					returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
  		}
  		catch (Exception ex) {
  			if (logger.isTraceEnabled()) {
  				logger.trace(getReturnValueHandlingErrorMessage("Error handling return value", returnValue), ex);
  			}
  			throw ex;
  		}
  	}
}
```
显然,里面就是调用HandlerMethodArgumentResolver解析参数,然后接收参数,反射调用方法获得返回值,最后用HandlerMethodReturnValueHandler进行返回值处理.

我们先从解析参数开始探究,继续查看invokeForRequest方法的源码:
```java
public class ServletInvocableHandlerMethod extends InvocableHandlerMethod {
  public Object invokeForRequest(NativeWebRequest request, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) {
			logger.trace("Invoking '" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
					"' with arguments " + Arrays.toString(args));
		}
		Object returnValue = doInvoke(args);
		if (logger.isTraceEnabled()) {
			logger.trace("Method [" + ClassUtils.getQualifiedMethodName(getMethod(), getBeanType()) +
					"] returned [" + returnValue + "]");
		}
		return returnValue;
	}
}
```
这个方法内部也非常简单,就是获取参数,然后反射执行方法获得返回值.继续查看getMethodArgumentValues方法源码:
```java
public class ServletInvocableHandlerMethod extends InvocableHandlerMethod {
  private Object[] getMethodArgumentValues(NativeWebRequest request, ModelAndViewContainer mavContainer,
			Object... providedArgs) throws Exception {

		MethodParameter[] parameters = getMethodParameters();
		Object[] args = new Object[parameters.length];
		for (int i = 0; i < parameters.length; i++) {
			MethodParameter parameter = parameters[i];
			parameter.initParameterNameDiscovery(this.parameterNameDiscoverer);
			args[i] = resolveProvidedArgument(parameter, providedArgs);
			if (args[i] != null) {
				continue;
			}
			if (this.argumentResolvers.supportsParameter(parameter)) {
				try {
					args[i] = this.argumentResolvers.resolveArgument(
							parameter, mavContainer, request, this.dataBinderFactory);
					continue;
				}
				catch (Exception ex) {
					if (logger.isDebugEnabled()) {
						logger.debug(getArgumentResolutionErrorMessage("Failed to resolve", i), ex);
					}
					throw ex;
				}
			}
			if (args[i] == null) {
				throw new IllegalStateException("Could not resolve method parameter at index " +
						parameter.getParameterIndex() + " in " + parameter.getMethod().toGenericString() +
						": " + getArgumentResolutionErrorMessage("No suitable resolver for", i));
			}
		}
		return args;
	}
}
```
里面实际的逻辑还是很简单,循环调用argumentResolvers的resolveArgument方法解析出反射方法的参数对象.具体的解析逻辑还是取决于HandlerMethodArgumentResolverComposite变量argumentResolvers委托的具体HandlerMethodArgumentResolver.

绕了一圈,重新进入HandlerMethodArgumentResolverComposite源码查看resolveArgument方法:
```java
public class HandlerMethodArgumentResolverComposite implements HandlerMethodArgumentResolver {
	@Override
	public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
		HandlerMethodArgumentResolver resolver = getArgumentResolver(parameter);
		if (resolver == null) {
			throw new IllegalArgumentException("Unknown parameter type [" + parameter.getParameterType().getName() + "]");
		}
		return resolver.resolveArgument(parameter, mavContainer, webRequest, binderFactory);
	}
	
		private HandlerMethodArgumentResolver getArgumentResolver(MethodParameter parameter) {
  		HandlerMethodArgumentResolver result = this.argumentResolverCache.get(parameter);
  		if (result == null) {
  			for (HandlerMethodArgumentResolver methodArgumentResolver : this.argumentResolvers) {
  				if (logger.isTraceEnabled()) {
  					logger.trace("Testing if argument resolver [" + methodArgumentResolver + "] supports [" +
  							parameter.getGenericParameterType() + "]");
  				}
  				if (methodArgumentResolver.supportsParameter(parameter)) {
  					result = methodArgumentResolver;
  					this.argumentResolverCache.put(parameter, result);
  					break;
  				}
  			}
  		}
  		return result;
  	}
}
```
可以看到委托HandlerMethodArgumentResolver的逻辑也很简单,就是将所有已经注册的HandlerMethodArgumentResolver循环调用supportsParameter方法,
找到第一个支持该参数解析的对象,然后调用其resolveArgument方法.

接着先选择一个最简单的HandlerMethodArgumentResolver实现类查看具体源码,就选择常用注解中最简单的的@PathVariable的参数解析器PathVariableMethodArgumentResolver查看,
实际上PathVariableMethodArgumentResolver的resolveArgument方法实现在其父类AbstractNamedValueMethodArgumentResolver中:
```java
public abstract class AbstractNamedValueMethodArgumentResolver implements HandlerMethodArgumentResolver {
	@Override
	public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

		NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
		MethodParameter nestedParameter = parameter.nestedIfOptional();

		Object resolvedName = resolveStringValue(namedValueInfo.name);
		if (resolvedName == null) {
			throw new IllegalArgumentException(
					"Specified name must not resolve to null: [" + namedValueInfo.name + "]");
		}

		Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
		if (arg == null) {
			if (namedValueInfo.defaultValue != null) {
				arg = resolveStringValue(namedValueInfo.defaultValue);
			}
			else if (namedValueInfo.required && !nestedParameter.isOptional()) {
				handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
			}
			arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
		}
		else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
			arg = resolveStringValue(namedValueInfo.defaultValue);
		}

		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
			try {
				arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
			}
			catch (ConversionNotSupportedException ex) {
				throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());
			}
			catch (TypeMismatchException ex) {
				throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());

			}
		}

		handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);

		return arg;
	}
}
```
经过debug可以发现,Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest)这一行代码中是简单的直接取出原始格式的数据.
由于是url中的参数,所以PathVariableMethodArgumentResolver中此处永远是获取对应的原始字符串.

实际上跟本次探究目的有关的代码主要还是arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter)这一句,
显然convertIfNecessary方法中有一些数据格式的转换操作,查看WebDataBinder父类DataBinder的注释:
> Binder that allows for setting property values onto a target object,
> including support for validation and binding result analysis.
> The binding process can be customized through specifying allowed fields,
> required fields, custom editors, etc.

注释说明该类的主要用途是绑定属性的值到目标对象,其中也包括了验证过程,且绑定过程可以被自定义.看来已经比较接近我们需求了,继续查看convertIfNecessary方法的源码:
```java
public class DataBinder implements PropertyEditorRegistry, TypeConverter {
  @Override
	public <T> T convertIfNecessary(Object value, Class<T> requiredType, MethodParameter methodParam)
			throws TypeMismatchException {

		return getTypeConverter().convertIfNecessary(value, requiredType, methodParam);
	}
	
	protected TypeConverter getTypeConverter() {
  		if (getTarget() != null) {
  			return getInternalBindingResult().getPropertyAccessor();
  		}
  		else {
  			return getSimpleTypeConverter();
  		}
  	}
}
```
显然里面只是简单的调用类型转换器TypeConverter接口的convertIfNecessary方法,继续debug,进入了TypeConverterSupport的convertIfNecessary方法:
```java
public abstract class TypeConverterSupport extends PropertyEditorRegistrySupport implements TypeConverter {
	@Override
	public <T> T convertIfNecessary(Object value, Class<T> requiredType, MethodParameter methodParam)
			throws TypeMismatchException {

		return doConvert(value, requiredType, methodParam, null);
	}
	
	private <T> T doConvert(Object value, Class<T> requiredType, MethodParameter methodParam, Field field)
  			throws TypeMismatchException {
  		try {
  			if (field != null) {
  				return this.typeConverterDelegate.convertIfNecessary(value, requiredType, field);
  			}
  			else {
  				return this.typeConverterDelegate.convertIfNecessary(value, requiredType, methodParam);
  			}
  		}
  		catch (ConverterNotFoundException ex) {
  			throw new ConversionNotSupportedException(value, requiredType, ex);
  		}
  		catch (ConversionException ex) {
  			throw new TypeMismatchException(value, requiredType, ex);
  		}
  		catch (IllegalStateException ex) {
  			throw new ConversionNotSupportedException(value, requiredType, ex);
  		}
  		catch (IllegalArgumentException ex) {
  			throw new TypeMismatchException(value, requiredType, ex);
  		}
  	}
}
```
显然,里面具体的类型转换逻辑依旧是交给this.typeConverterDelegate,继续查看该变量源码:
```java
class TypeConverterDelegate {
  public <T> T convertIfNecessary(String propertyName, Object oldValue, Object newValue,
        Class<T> requiredType, TypeDescriptor typeDescriptor) throws IllegalArgumentException {
  
      // Custom editor for this type?
      PropertyEditor editor = this.propertyEditorRegistry.findCustomEditor(requiredType, propertyName);
      //...
      // No custom editor but custom ConversionService specified?
      ConversionService conversionService = this.propertyEditorRegistry.getConversionService();
      if (editor == null && conversionService != null && newValue != null && typeDescriptor != null) {
        TypeDescriptor sourceTypeDesc = TypeDescriptor.forObject(newValue);
        if (conversionService.canConvert(sourceTypeDesc, typeDescriptor)) {
          try {
            return (T) conversionService.convert(newValue, sourceTypeDesc, typeDescriptor);
          }
          catch (ConversionFailedException ex) {
            // fallback to default conversion logic below
            conversionAttemptEx = ex;
          }
        }
      }
  
      Object convertedValue = newValue;
  
      // Value not of required type?
      if (editor != null || (requiredType != null && !ClassUtils.isAssignableValue(requiredType, convertedValue))) {
        if (typeDescriptor != null && requiredType != null && Collection.class.isAssignableFrom(requiredType) &&
            convertedValue instanceof String) {
          TypeDescriptor elementTypeDesc = typeDescriptor.getElementTypeDescriptor();
          if (elementTypeDesc != null) {
            Class<?> elementType = elementTypeDesc.getType();
            if (Class.class == elementType || Enum.class.isAssignableFrom(elementType)) {
              convertedValue = StringUtils.commaDelimitedListToStringArray((String) convertedValue);
            }
          }
        }
        if (editor == null) {
          editor = findDefaultEditor(requiredType);
        }
        convertedValue = doConvertValue(oldValue, convertedValue, requiredType, editor);
      }
  
      boolean standardConversion = false;
  
      if (requiredType != null) {
        // Try to apply some standard type conversion rules if appropriate.
  
        if (convertedValue != null) {
          if (Object.class == requiredType) {
            return (T) convertedValue;
          }
          //......
        }
        //......
    }
  }
}
```
显然源码中涉及到了两种类型转换器的接口,即PropertyEditor和ConversionService.

这里决定对比一下ConversionService和PropertyEditor,看看为什么要有两种类型转换器.直接进入ConversionService源码查看注释:
> A service interface for type conversion. This is the entry point into the convert system.
> Call {@link #convert(Object, Class)} to perform a thread-safe type conversion using this system.

看到"thread-safe type conversion",我就猜测很有可能PropertyEditor是非线程安全的原来的类型转换器,而ConversionService则是新的类型转换器.

查看官方文档后,大致整理了情况.**PropertyEditor接口的确是最早的类型转换器接口**,最初全部都是用这个接口来完成类型转换的.大致的逻辑如下图:
![基于PropertyEditor的流程图](/img/in-post/2018-10-27-java-spring-serialize-deserialize-design/property-editor-flow.jpg)
而这个接口的设计问题主要有如下几点:
1. **从setAsText和getAsText两个方法名就能看出,其只支持String<-->Object之间的互相转换.如果要进行其他类型之间的转换就比较麻烦,甚至没法做到.**
**比如Long时间戳转换为可用的时间类型时还能通过把Long转成字符串来间接实现.如果是反过来的话,时间类型的字符串显然是没法转换为Long时间戳的.**
2. **PropertyEditor是线程不安全的(这里说的是Spring的默认实现类,具体的自定义实现类取决于自定义代码)，也就是有状态的，因此每次使用时都需要创建一个,比较影响性能.**
**这是一个类似SimpleDateFormat一样的设计失误.**
3. **没有泛型带来的类型约束,需要自己代码判断,容易出错.**
4. **粒度太粗,只能到类型为止,同一类型的不同格式化不能满足.**

于是在Spring3中引入了新的接口设计来取代PropertyEditor接口.

网络大部分博客说是Converter接口取代了PropertyEditor接口,实际上不太正确,应该说是Formatter接口取代了PropertyEditor接口,
或者说Formatter接口和Converter接口各自取代了PropertyEditor接口的一部分职责,将类型转换与序列化和反序列化为字符串的职责分开了.

**Converter及相关接口被设计为通用的类型转换器,取代了PropertyEditor接口的类型转换职责,与PropertyEditor的比较如下:**
1. **改进为能够实现任何两种类型之间的转换.**
2. **要求实现线程安全,自然Spring的默认实现类全部都实现了线程安全.**
3. **使用了泛型约束.**
4. **同样只能到类型为止.**

**Formatter及相关接口被设计为取代了PropertyEditor接口的序列化和反序列化为字符串,或者说格式化和解析功能的职责,与PropertyEditor的比较如下:**
1. **同样只能实现String<-->Object之间的互相转换.**
2. **要求实现线程安全,自然Spring的默认实现类全部都实现了线程安全.**
3. **使用了泛型约束.**
4. **更细粒度的控制,提供了注解方式方便属性或者说字段级别的粒度,如果通过编程方式控制的话,还能有Local对象提供地区相关信息,能过做到按地区实现多种格式化.**

而上面源码中的**ConversionService接口则是管理代理所有Converter及相关接口,统一提供服务,内部当然也是按顺序委托给首个支持这种转换的Converter.**

那么按照这个说法,为什么上面源码中没有看见Formatter接口被使用呢?不是说其代替了PropertyEditor接口了吗?

事实上,**Formatter接口**是有被使用的,但是是**被适配器包装成PropertyEditor后,当做PropertyEditor调用的**.
其实这种做法比较奇怪,一般来说都是把老的接口使用适配器包装成新的接口使用的,比如Jdk中的Enumeration和Collection.

而**TypeConverterDelegate类内部则是一起管理PropertyEditor和Converter,并且在优先级上是PropertyEditor优先.**

也就是说从PathVariableMethodArgumentResolver来看,对于本次探究的目的,只要实现自定义的Converter或者Formatter相关接口,并且注册到Spring中就可以做到.

但是由于不同的MethodArgumentResolver可能有不同的解析逻辑,这种做法是否都有效,还需要继续查看其它常用的MethodArgumentResolver.

查看RequestParamMethodArgumentResolver源码,内部调用依然是AbstractNamedValueMethodArgumentResolver类的方法,逻辑自然与PathVariableMethodArgumentResolver保持一致.

经过debug发现如下常用写法,匹配的是ServletModelAttributeMethodProcessor类.
```java
@RestController
@RequestMapping("/test")
public class TestController{
  @GetMapping("")
  public void get(RequestParam requestParam) {
      System.out.println(requestParam);
    }
}
```
ServletModelAttributeMethodProcessor的源码如下:
```java
public class ModelAttributeMethodProcessor implements HandlerMethodArgumentResolver, HandlerMethodReturnValueHandler {
  @Override
  	public final Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
  			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {
  
  		String name = ModelFactory.getNameForParameter(parameter);
  		ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
  		if (ann != null) {
  			mavContainer.setBinding(name, ann.binding());
  		}
  
  		Object attribute = (mavContainer.containsAttribute(name) ? mavContainer.getModel().get(name) :
  				createAttribute(name, parameter, binderFactory, webRequest));
  
  		WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
  		if (binder.getTarget() != null) {
  			if (!mavContainer.isBindingDisabled(name)) {
  				bindRequestParameters(binder, webRequest);
  			}
  			validateIfApplicable(binder, parameter);
  			if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
  				throw new BindException(binder.getBindingResult());
  			}
  		}
  
  		// Add resolved attribute and BindingResult at the end of the model
  		Map<String, Object> bindingResultModel = binder.getBindingResult().getModel();
  		mavContainer.removeAttributes(bindingResultModel);
  		mavContainer.addAllAttributes(bindingResultModel);
  
  		return binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
  	}
}
```
其中的重点在于bindRequestParameters(binder, webRequest)和binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter)两句代码.
由于一般来说实例化的attribute变量类型就会是要求的参数类型,所以binder.convertIfNecessary此时一般不会做任何转换操作,只需继续看bindRequestParameters方法源码即可:
```java
public class ServletModelAttributeMethodProcessor extends ModelAttributeMethodProcessor {
  @Override
	protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
		ServletRequest servletRequest = request.getNativeRequest(ServletRequest.class);
		ServletRequestDataBinder servletBinder = (ServletRequestDataBinder) binder;
		servletBinder.bind(servletRequest);
	}
}
```
显然实际逻辑在bind方法中,继续查看其源码:
```java
public class ServletRequestDataBinder extends WebDataBinder {
	public void bind(ServletRequest request) {
		MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
		MultipartRequest multipartRequest = WebUtils.getNativeRequest(request, MultipartRequest.class);
		if (multipartRequest != null) {
			bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
		}
		addBindValues(mpvs, request);
		doBind(mpvs);
	}
}
```
其中mpvs是获取的原始参数键值对,其基于ServletRequest产生,所以此处的值类型必定为String或者String数组.
所以doBind方法中才会有转换的逻辑,继续查看doBind方法源码:
```java
public class WebDataBinder extends DataBinder {
	@Override
	protected void doBind(MutablePropertyValues mpvs) {
		checkFieldDefaults(mpvs);
		checkFieldMarkers(mpvs);
		super.doBind(mpvs);
	}
}
```
显然需要继续查看super.doBind方法源码:
```java
public class DataBinder implements PropertyEditorRegistry, TypeConverter {
	protected void doBind(MutablePropertyValues mpvs) {
		checkAllowedFields(mpvs);
		checkRequiredFields(mpvs);
		applyPropertyValues(mpvs);
	}
	
	protected void applyPropertyValues(MutablePropertyValues mpvs) {
		try {
			// Bind request parameters onto target object.
			getPropertyAccessor().setPropertyValues(mpvs, isIgnoreUnknownFields(), isIgnoreInvalidFields());
		}
		catch (PropertyBatchUpdateException ex) {
			// Use bind error processor to create FieldErrors.
			for (PropertyAccessException pae : ex.getPropertyAccessExceptions()) {
				getBindingErrorProcessor().processPropertyAccessException(pae, getInternalBindingResult());
			}
		}
	}
	
	protected ConfigurablePropertyAccessor getPropertyAccessor() {
		return getInternalBindingResult().getPropertyAccessor();
	}
}
```
继续查看setPropertyValues方法源码:
```java
public abstract class AbstractPropertyAccessor extends TypeConverterSupport implements ConfigurablePropertyAccessor {
  @Override
  	public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid)
  			throws BeansException {
  
  		List<PropertyAccessException> propertyAccessExceptions = null;
  		List<PropertyValue> propertyValues = (pvs instanceof MutablePropertyValues ?
  				((MutablePropertyValues) pvs).getPropertyValueList() : Arrays.asList(pvs.getPropertyValues()));
  		for (PropertyValue pv : propertyValues) {
  			try {
  				// This method may throw any BeansException, which won't be caught
  				// here, if there is a critical failure such as no matching field.
  				// We can attempt to deal only with less serious exceptions.
  				setPropertyValue(pv);
  			}
  			catch (NotWritablePropertyException ex) {
  				if (!ignoreUnknown) {
  					throw ex;
  				}
  				// Otherwise, just ignore it and continue...
  			}
          //......
  		}
  		//......
  	}
}
```
继续查看setPropertyValues方法源码:
```java
public abstract class AbstractNestablePropertyAccessor extends AbstractPropertyAccessor {
	@Override
	public void setPropertyValue(PropertyValue pv) throws BeansException {
		PropertyTokenHolder tokens = (PropertyTokenHolder) pv.resolvedTokens;
		if (tokens == null) {
			String propertyName = pv.getName();
			AbstractNestablePropertyAccessor nestedPa;
			try {
				nestedPa = getPropertyAccessorForPropertyPath(propertyName);
			}
			catch (NotReadablePropertyException ex) {
				throw new NotWritablePropertyException(getRootClass(), this.nestedPath + propertyName,
						"Nested property in path '" + propertyName + "' does not exist", ex);
			}
			tokens = getPropertyNameTokens(getFinalPath(nestedPa, propertyName));
			if (nestedPa == this) {
				pv.getOriginalPropertyValue().resolvedTokens = tokens;
			}
			nestedPa.setPropertyValue(tokens, pv);
		}
		else {
			setPropertyValue(tokens, pv);
		}
	}
	
	protected void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
		if (tokens.keys != null) {
			processKeyedProperty(tokens, pv);
		}
		else {
			processLocalProperty(tokens, pv);
		}
	}
	
	private void processLocalProperty(PropertyTokenHolder tokens, PropertyValue pv) {
  //......
		Object oldValue = null;
		try {
			Object originalValue = pv.getValue();
			Object valueToApply = originalValue;
			if (!Boolean.FALSE.equals(pv.conversionNecessary)) {
				if (pv.isConverted()) {
					valueToApply = pv.getConvertedValue();
				}
				else {
					if (isExtractOldValueForEditor() && ph.isReadable()) {
						try {
							oldValue = ph.getValue();
						}
						catch (Exception ex) {
							//......
						}
					}
					valueToApply = convertForProperty(
							tokens.canonicalName, oldValue, originalValue, ph.toTypeDescriptor());
				}
				pv.getOriginalPropertyValue().conversionNecessary = (valueToApply != originalValue);
			}
			ph.setValue(this.wrappedObject, valueToApply);
		}
		catch (TypeMismatchException ex) {
			throw ex;
		}
    //......
	}
	
	protected Object convertForProperty(String propertyName, Object oldValue, Object newValue, TypeDescriptor td)
			throws TypeMismatchException {

		return convertIfNecessary(propertyName, oldValue, newValue, td.getType(), td);
	}
	
	private Object convertIfNecessary(String propertyName, Object oldValue, Object newValue, Class<?> requiredType,
			TypeDescriptor td) throws TypeMismatchException {
		try {
			return this.typeConverterDelegate.convertIfNecessary(propertyName, oldValue, newValue, requiredType, td);
		}
		catch (ConverterNotFoundException ex) {
			//......
		}
		//......
	}
}
```
到了这里,我们又看到了熟悉的this.typeConverterDelegate.convertIfNecessary,显然,后面的逻辑肯定与PathVariableMethodArgumentResolver一样委托给ConversionService处理了.

最后我们查看@RequestBody对应的RequestResponseBodyMethodProcessor源码:
```java
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {
	@Override
	public Object resolveArgument(MethodParameter parameter, ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, WebDataBinderFactory binderFactory) throws Exception {

		parameter = parameter.nestedIfOptional();
		Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
		String name = Conventions.getVariableNameForParameter(parameter);

		WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
		if (arg != null) {
			validateIfApplicable(binder, parameter);
			if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
				throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
			}
		}
		mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());

		return adaptArgumentIfNecessary(arg, parameter);
	}
	
	@Override
	protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter parameter,
			Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

		HttpServletRequest servletRequest = webRequest.getNativeRequest(HttpServletRequest.class);
		ServletServerHttpRequest inputMessage = new ServletServerHttpRequest(servletRequest);

		Object arg = readWithMessageConverters(inputMessage, parameter, paramType);
		if (arg == null) {
			if (checkRequired(parameter)) {
				throw new HttpMessageNotReadableException("Required request body is missing: " +
						parameter.getMethod().toGenericString());
			}
		}
		return arg;
	}
}
```
显然其转换逻辑在readWithMessageConverters方法中,继续查看源码:
```java
public abstract class AbstractMessageConverterMethodArgumentResolver implements HandlerMethodArgumentResolver {
	protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter,
			Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

		MediaType contentType;
		boolean noContentType = false;
		try {
			contentType = inputMessage.getHeaders().getContentType();
		}
		catch (InvalidMediaTypeException ex) {
			throw new HttpMediaTypeNotSupportedException(ex.getMessage());
		}
		if (contentType == null) {
			noContentType = true;
			contentType = MediaType.APPLICATION_OCTET_STREAM;
		}
		//......
		try {
			inputMessage = new EmptyBodyCheckingHttpInputMessage(inputMessage);

			for (HttpMessageConverter<?> converter : this.messageConverters) {
				Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
				if (converter instanceof GenericHttpMessageConverter) {
					GenericHttpMessageConverter<?> genericConverter = (GenericHttpMessageConverter<?>) converter;
					if (genericConverter.canRead(targetType, contextClass, contentType)) {
						if (logger.isDebugEnabled()) {
							logger.debug("Read [" + targetType + "] as \"" + contentType + "\" with [" + converter + "]");
						}
						if (inputMessage.getBody() != null) {
							inputMessage = getAdvice().beforeBodyRead(inputMessage, parameter, targetType, converterType);
							body = genericConverter.read(targetType, contextClass, inputMessage);
							body = getAdvice().afterBodyRead(body, inputMessage, parameter, targetType, converterType);
						}
						else {
							body = getAdvice().handleEmptyBody(null, inputMessage, parameter, targetType, converterType);
						}
						break;
					}
				}
				else if (targetClass != null) {
					if (converter.canRead(targetClass, contentType)) {
						if (logger.isDebugEnabled()) {
							logger.debug("Read [" + targetType + "] as \"" + contentType + "\" with [" + converter + "]");
						}
						if (inputMessage.getBody() != null) {
							inputMessage = getAdvice().beforeBodyRead(inputMessage, parameter, targetType, converterType);
							body = ((HttpMessageConverter<T>) converter).read(targetClass, inputMessage);
							body = getAdvice().afterBodyRead(body, inputMessage, parameter, targetType, converterType);
						}
						else {
							body = getAdvice().handleEmptyBody(null, inputMessage, parameter, targetType, converterType);
						}
						break;
					}
				}
			}
		}
		catch (IOException ex) {
			throw new HttpMessageNotReadableException("I/O error while reading input message", ex);
		}
		//......
	}
}
```
显然,body中的信息最后由首个支持的HttpMessageConverter来处理.查看HttpMessageConverter接口的注释:
> Strategy interface that specifies a converter that can convert from and to HTTP requests and responses.

说明**该接口是用来转换http请求到目标类型的对象,以及目标类型对象到Http响应的策略模式接口**,其源码如下:
```java
public interface HttpMessageConverter<T> {
	boolean canRead(Class<?> clazz, MediaType mediaType);

	boolean canWrite(Class<?> clazz, MediaType mediaType);

	List<MediaType> getSupportedMediaTypes();

	T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;

	void write(T t, MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;
}
public interface HttpInputMessage extends HttpMessage {
	InputStream getBody() throws IOException;
}
public interface HttpOutputMessage extends HttpMessage {
	OutputStream getBody() throws IOException;
}
```
显然**该接口实现的是从二进制到指定Java类型的转换.我们最常用的其实现为MappingJackson2HttpMessageConverter和FastJsonHttpMessageConverter,
就是将该序列化和反序列化过程交给了jackson和fastJson来处理.**

其内部逻辑就属于实现类的自定义,与上面说的Formatter以及Converter接口毫无关系了,毕竟很有可能**在某些二进制序列化协议中,
是直接将IOStream的二进制数据直接转换到目标Java类型的,期间可能根本没有任何Java类型之间的转换,自然Formatter以及Converter在这里就不生效了.**

这就意味着你**通过Formatter以及Converter接口实现的自定义格式化与反格式化是不能在通过jackson或fastJson进行转换的Http的body中生效的,
虽然json格式在http协议层面本质是字符串,因此期间会发生Java类型之间的转换.**

**因此想要在类似json,xml等以字符串格式的序列化协议中进行自定义的序列化与反序列化格式,需要去修改具体HttpMessageConverter实现类的逻辑,
需要委托的json库合xml库等支持相关的配置,修改注册在Spring容器中的配置,否则只能自己实现HttpMessageConverter并替代原实现类才能做到.**

# 三.总结
本次探究到这里也就差不多达到目的了,下面给出本次需求的方案总结.首先来看SpringMvc中整体的序列化与反序列化设计图.
![SpringMvc序列化与反序列化设计](/img/in-post/2018-10-27-java-spring-serialize-deserialize-design/springmvc-serialize-deserialize.png)
从上图我们可以看出,想要实现对一个类型的全局统一自定义序列化与反序列化,需要做到以下两点:
1. **实现Spring-core的PropertyEditor\Formatter\Converter接口并注册到Spring中,实现对Url和Header的自定义解析.**
2. **对于所有项目支持的序列化标准为字符串的HttpMessageConverter中修改其配置,加入全局的自定义序列化与反序列化.**

之所以这么说是因为,从上面HttpMessageConverter接口的设计和使用来看,**SpringMvc是天然支持同一个接口代码同时支持多套序列化与反序列化协议的,
其会根据Http请求头部来自动匹配对应的HttpMessageConverter.**

而SpringMvc在自动帮开发者完成了序列化与反序列化操作时,屏蔽了序列化与反序列化协议中的不同解析逻辑后,**开发者只需要实现一套业务逻辑,即可自动支持多种序列化标准的请求和响应.**

对于第一点Spring-core的自定义,有以下方法可选:
1. **实现Formatter,并通过一个@Configuration的配置类继承WebMvcConfigurerAdapter,重写addFormatters方法来注册.这是SpringBoot官方推荐的注册方式.**
2. **实现Converter,其他步骤同上.**
3. 实现PropertyEditor并注册.(不推荐,上面说过该接口其实已经不被推荐,应该使用Formatter来代替).

对于到底实现Formatter还是Converter,**个人的理解是这样的,Converter是一种Java类型之间的转换,理论上任何两种类型之间只应该有一种转换规则.**

而Formatter则不同,**Formatter是一种格式化,限制只能为String和其他Java类型之间的转换的原因就是,转换出来的String是一种给人阅读用的,要考虑阅读者的理解和习惯,
所以Formatter接口的参数中会有Locale参数存在,很有可能有多种格式化的标准,甚至某些情况下,同样类型的属性格式化标准是不同的.所以Formatter支持属性级别的注解**

当然很多时候,String类型和其他类型之间的转换总是很模糊,分不清到底是类型转换还是格式化,甚至Spring官方也是把两者都统一注入到一个类中进行管理的.

那么对于我们本次的需求枚举和Java8时间类型到字符串的转换,个人习惯认为应该是一种格式化,应该实现Formatter.或者换个说法,
个人习惯认为String类型的Converter大多数时候是一种默认情况下的Formatter,如果需要的话,应该使用适配器Adapter把相应的Formatter包装成Converter.

因此,这里本人选择1方案来注册.以下是简单的LocalDateTime的Formatter实现:
```java
@Configuration
@Data
public class DateTimeFormatterConfig {
  private final String defaultDateTimeFormatterString;

  private final DateTimeFormatter defaultDateTimeFormatter;

  private final ImmutableList<DateTimeFormatter> parseDateTimeFormatters;

  @Autowired
  public DateTimeFormatterConfig(@Value("${spring.default.date-format:yyyy-MM-dd HH:mm:ss}") String defaultDateTimeFormatterString){
    this.defaultDateTimeFormatterString = defaultDateTimeFormatterString;
    defaultDateTimeFormatter=DateTimeFormatter.ofPattern(defaultDateTimeFormatterString);
    parseDateTimeFormatters=ImmutableList
        .of(defaultDateTimeFormatter,DateTimeFormatter.ISO_LOCAL_DATE_TIME,
            DateTimeFormatter.ISO_DATE_TIME, DateTimeFormatter.ISO_LOCAL_DATE,
            DateTimeFormatter.ISO_DATE, DateTimeFormatter.BASIC_ISO_DATE,
            DateTimeFormatter.ISO_ORDINAL_DATE, DateTimeFormatter.ISO_WEEK_DATE,
            DateTimeFormatter.ISO_LOCAL_TIME, DateTimeFormatter.ISO_TIME,
            DateTimeFormatter.ISO_ZONED_DATE_TIME, DateTimeFormatter.ISO_OFFSET_DATE_TIME,
            DateTimeFormatter.RFC_1123_DATE_TIME, DateTimeFormatter.ISO_OFFSET_DATE,
            DateTimeFormatter.ISO_OFFSET_TIME, DateTimeFormatter.ISO_INSTANT);
  }

  @Bean
  public DateTimeFormatter defaultDateTimeFormatter() {
    return defaultDateTimeFormatter;
  }
}

@Component
public class LocalDateTimeFormatter implements Formatter<LocalDateTime> {

  private final DateTimeFormatterConfig dateTimeFormatterConfig;

  @Autowired
  public LocalDateTimeFormatter(DateTimeFormatterConfig dateTimeFormatterConfig) {
    this.dateTimeFormatterConfig = dateTimeFormatterConfig;
  }

  @Override
  public LocalDateTime parse(String text, Locale locale) throws ParseException {
    for (DateTimeFormatter dateTimeFormatter : dateTimeFormatterConfig.getParseDateTimeFormatters()) {
      try {
        return LocalDateTime.parse(text, dateTimeFormatter);
      } catch (DateTimeException e) {
      }
    }
    throw new ParseException("all DateTimeFormatter can't parse " + text+" to LocalDateTime", -1);
  }

  @Override
  public String print(LocalDateTime object, Locale locale) {
    return object.format(dateTimeFormatterConfig.getDefaultDateTimeFormatter());
  }
}

@Configuration
public class FormatterConfig extends WebMvcConfigurerAdapter {

  @Resource
  private DateTimeFormatterConfig dateTimeFormatterConfig;

  @Resource
  private LocalDateTimeFormatter localDateTimeFormatter;

  @Override
  public void addFormatters(FormatterRegistry registry) {
    registry.addFormatter(localDateTimeFormatter);
  }
}
```
**由于希望反序列化时能够最好能够接受所有可以理解的格式,所以解析时调用了所有Java内置的时间类型.其中使用了配置读取来实现通过配置自定义默认输出时间格式,**
其他Java8时间类型类似.

对于第二点序列化标准为字符串的HttpMessageConverter的配置修改,由于项目内主要以jackson和fastJson进行序列化和反序列化,所以就以这两个为示例.

首先看Spring默认的jackson,从上面的图中,我们可以看到**jackson的自定义序列化与反序列化配置,需要实现JsonSerializer和JsonDeserializer接口.**
至于将实现的自定义类注册使用,根据文档说明有如下方法:
1. Module Interface模式全局注册
(1) 所有类型为com.fasterxml.jackson.databind.Module的beans都会自动注册到自动配置的Jackson2ObjectMapperBuilder，并应用到它创建的任何ObjectMapper实例
(2) 如果想完全替换默认的ObjectMapper，你既可以定义该类型的@Bean并注解@Primary，也可以定义Jackson2ObjectMapperBuilder @Bean，通过builder构建。
(3) 如果你提供MappingJackson2HttpMessageConverter类型的@Bean，它们将替换MVC配置中的默认值。
2. 通过注解使用(由于本次需求是全局统一格式,所以不考虑)

对于1.2和1.3的方案,考虑到和需求不符,需求并不想要使默认的MappingJackson2HttpMessageConverter和其默认配置ObjectMapper失效,所以不采用.

然而发现1.1方案却并没有生效.经过debug发现原因是最后容器内存在两个MappingJackson2HttpMessageConverter对象.

一个注册为Spring容器的Bean,但是却没注册到MessageConverter列表中,另一个注册到了MessageConverter列表中,却没注册到Spring容器中.

因此**这里采取了迂回方案,即使用SpringBoot提供的注册修改MessageConverters的接口来获取当前真正使用的MappingJackson2HttpMessageConverter,再修改其ObjectMapper,
注册自定义module.** WebMvcConfig类修改源码如下:
```java
public class JsonDeserializerAdapter<T> extends JsonDeserializer<T> {

  private final Formatter<T> formatter;

  public JsonDeserializerAdapter(Formatter<T> formatter) {
    this.formatter = formatter;
  }

  @Override
  public T deserialize(JsonParser p, DeserializationContext ctxt)
      throws IOException, JsonProcessingException {
    try {
      return formatter.parse(p.getValueAsString(), Locale.getDefault());
    } catch (ParseException e) {
      throw new JsonParseException(p, e.getMessage());
    }
  }
}
@Configuration
public class FormatterConfig extends WebMvcConfigurerAdapter {

  @Resource
  private DateTimeFormatterConfig dateTimeFormatterConfig;

  @Resource
  private LocalDateTimeFormatter localDateTimeFormatter;

  @Bean
  public LocalDateTimeFormatter localDateTimeFormatter() {
    return new LocalDateTimeFormatter();
  }

  @Override
  public void addFormatters(FormatterRegistry registry) {
    registry.addFormatter(localDateTimeFormatter);
  }

  @Override
  public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    Optional<MappingJackson2HttpMessageConverter> mappingJackson2HttpMessageConverterOptional = converters
        .stream()
        .filter(converter -> converter instanceof MappingJackson2HttpMessageConverter)
        .map(MappingJackson2HttpMessageConverter.class::cast).findAny();
    mappingJackson2HttpMessageConverterOptional
        .ifPresent(mappingJackson2HttpMessageConverter -> mappingJackson2HttpMessageConverter
            .getObjectMapper().registerModule(new SimpleModule().addDeserializer(LocalDateTime.class,
                new JsonDeserializerAdapter<>(localDateTimeFormatter)))
        );

  }
}
```
接下来是fastJson的自定义序列化和反序列化,同样从上图可以得知其通过实现SerializeFilter或ParseProcess定制反序列化.

但是由于fastJson针对于包含Java8新时间类型的全局时间格式序列化有特殊处理,可以直接通过配置实现,所以这里可以不通过自己实现接口来完成.

具体实现是修改com.alibaba.fastjson.JSON.DEFFAULT_DATE_FORMAT属性,并且开启SerializerFeature.WriteDateUseDateFormat即可(此项默认开启).
另外由于项目中的FastJsonHttpMessageConverter对象没有注册到Spring容器中,所以如果有其他自定义序列化(如本次需求中的枚举)需要注册到FastJsonHttpMessageConverter,
依然只能采用上面的迂回方案来完成保留默认配置的情况下,对原有配置的修改.

WebMvcConfig类修改源码如下:
```java
@Configuration
public class FormatterConfig extends WebMvcConfigurerAdapter {

  @Resource
  private DateTimeFormatterConfig dateTimeFormatterConfig;

  @Resource
  private LocalDateTimeFormatter localDateTimeFormatter;

  @Override
  public void addFormatters(FormatterRegistry registry) {
    registry.addFormatter(localDateTimeFormatter);
  }

  @Override
  public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    JSON.DEFFAULT_DATE_FORMAT=dateTimeFormatterConfig.getJsonDateTimeFormatter();

    Optional<MappingJackson2HttpMessageConverter> mappingJackson2HttpMessageConverterOptional = converters
        .stream()
        .filter(converter -> converter instanceof MappingJackson2HttpMessageConverter)
        .map(MappingJackson2HttpMessageConverter.class::cast).findAny();
    mappingJackson2HttpMessageConverterOptional
        .ifPresent(mappingJackson2HttpMessageConverter -> mappingJackson2HttpMessageConverter
            .getObjectMapper().registerModule(new SimpleModule().addDeserializer(LocalDateTime.class,
                new JsonDeserializerAdapter<>(localDateTimeFormatter)))
        );

  }
}
```
到此为止就已经完成了所有地方的自定义时间格式.