---
layout:     post
title:      "数据库null值替代值的优雅处理方案"
subtitle:   "数据库null值替代值的优雅处理方案"
date:       2021-01-12
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - DB Null Value
    - db null value
    - Database Null Value
    - database null value
    - DB Null Substitute Value
    - db null substitute value
    - Database Null Substitute Value
    - database null substitute value
---


# 数据库null值替代值

**在编程中,各个编程语言中,null值都是一个特殊含义的值.代表的是空值.**

**各个数据库也同样支持null值,但是由于null值的特殊性,sql条件的实际结果会和一般开发人员的预期结果完全不同,并且在包括索引在内的各方面性能上都会有较大影响.所以一般会要求在非特殊情况下,尽量使用特定值来替代null值.**

**这个数据库特定值我们就可以称之为null值替代值.**

**因此会出现同样是表示一个数据的某个字段值未填写状态,代码和数据库中的值不一致的情况.**

举例来说,[逻辑删除实现方案](https://caotc.org/2020/10/20/logic-delete/)中删除时间字段方案中强调过的一点:

>这里需要强调的是不能使用null作为逻辑删除字段的默认值,否则因为null值无法放入索引,会导致唯一索引加上逻辑删除字段后失效,可以插入多条该字段为null的数据,存在多条未被删除的有效数据.

这就是典型的表示一个数据的某个字段值未填写状态,代码和数据库中的值不一致的情况.

再比如说是同样手机号字段,手机号未填写的数据,在数据库手机号字段的值为空字符串.而在代码中对于没有填写手机号的数据对象的手机号字段的值一般使用null值来表示.

同样是表示一个数据的手机号未填写状态,代码和数据库中的值不一致.

**从概念上来说,这种现象会导致所有相关开发者强制关心自己所使用的对象的一切细节,包括该对象是否来自数据库,数据库的对应表的每个字段的null值替代值.**

**如果这种值没有被屏蔽地传递到DTO中,传递到其他服务,甚至开发者还需要关心其他服务的表的字段的null值替代值.**

**从实际编程来说,这种现象的字段的会导致代码中需要在每个地方使用字段is null或字段等于特定值来判断该字段是否为未填写,并且在很多意想不到的情况中引发问题.**

举例来说,某个接口需要校验手机号是否为正确格式,可以不填写手机号,但是在填写了手机号的时候需要校验格式正确性,使用了框架提供的参数校验功能.

这时,将数据库获取的数据传入接口时,就会因为空字符串并非null值而触发格式校验,但是空字符串又不是正确手机号格式.

为了避免这种情况,接口需要放弃框架的参数校验功能,而是手动进行校验,于是会造成代码腐烂.

或者要求接口调用方只能使用null值来表示手机号未填写状态,但是又会造成代码中空值标准不一,让开发者在调用各个接口时遇到自己意想不到的问题.

# 处理思路
要解决这个问题,首先需要理解问题的本质是什么.

**问题的本质是由于数据库的null值替代值的存在,代码中对于同一个对象的同一个字段的空值存在两种标准.并且各种三方库和框架的层级上只认为null值为空值的标准.**

**空值存在两种标准而不统一本身就是一种违反编程原则的问题,将内部的细节暴露给外部.**

**因此,处理方案的原则应该是统一标准,屏蔽差异.**

通常设计中,数据库和业务代码层之间都是统一通过orm中间层来进行操作的,显然,应该在orm中间层中把作为性能提升的特殊处理手段的数据库null值替代值进行屏蔽.

在进行新增和更新操作时,在orm层将需要新增和更新的字段的null值替换为对应的null值替代值,再进行实际的新增和更新操作.

在进行查询操作时,在orm层需要将查询出来的数据中的字段的null值替代值替换为null值,然后再将数据返回给业务代码层.

# 实现方案

显然,根据我们的处理思路,应该在orm中间层使用拦截器模式进行屏蔽和映射操作,统一标准.

由于我们使用的是mybatis,因此使用其interceptor机制进行操作.

# 设计缺陷

由于复杂sql的解析问题和根据column name寻找对应属性难以做到,对于where条件中的字段无法做到将使用了null替代值注解的字段is null的条件转换为=null替代值的操作

## null值替代值注解
```java
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target(value = {FIELD})
public @interface NullSubstituteValue {
    String value() default "";

    /**
     * 中转类,当该属性有值时,转换null替代值不是直接将String的value()值转为属性类型,而是按照顺序,先转为中转类型,最后转为属性类型.
     */
    Class<?>[] transferClasses() default {};
}
```
## null值替代值的配置类
```java
@Slf4j
@AllArgsConstructor
@FieldDefaults(makeFinal = true, level = AccessLevel.PRIVATE)
public class NullSubstituteValueConfiguration {

    @Data
    @AllArgsConstructor
    @FieldDefaults(makeFinal = true, level = AccessLevel.PRIVATE)
    public static class NullSubstituteValueEntry<T>{
        Class<T> key;
        T value;
    }

    Map<Class<?>, Object> classToNullSubstituteValues = new HashMap<>();

    /**
     * 注册类对应的默认值
     * @param classNullSubstituteValueEntry classNullSubstituteValueEntry
     * @return self
     *
     * @author caotc
     * @date 2021-01-12
     * @since 1.0.0
     */
    public <T> NullSubstituteValueConfiguration register(@NonNull NullSubstituteValueEntry<T> classNullSubstituteValueEntry){
        classToNullSubstituteValues.put(classNullSubstituteValueEntry.getKey(),classNullSubstituteValueEntry.getValue());
        return this;
    }

    /**
    * 注册类对应的默认值
    * @param type 类型
    * @param nullSubstituteValue 默认值
    * @return self
    *
    * @author caotc
    * @date 2021-01-12
    * @since 1.0.0
    */
    public <T> NullSubstituteValueConfiguration register(@NonNull Class<T> type,@NonNull T nullSubstituteValue){
        classToNullSubstituteValues.put(type,nullSubstituteValue);
        return this;
    }

    /**
    * 获取类型对应默认值
    * @param type 类型
    * @return 默认值
    *
    * @author caotc
    * @date 2021-01-12
    * @since 1.0.0
    */
    @SuppressWarnings("unchecked")
    public <T> T getNullSubstituteValue(@NonNull Class<T> type){
        return (T) classToNullSubstituteValues.get(type);
    }

    /**
    * 查询该类型是否有对应的默认值
    * @param type 类型
    * @return 是否有对应的默认值
    *
    * @author caotc
    * @date 2021-01-12
    * @since 1.0.0
    */
    public boolean containsNullSubstituteValue(@NonNull Class<?> type){
        return classToNullSubstituteValues.containsKey(type);
    }
}
```

## null值替代值处理器
```java
@Slf4j
@Validated
@AllArgsConstructor
@FieldDefaults(makeFinal = true, level = AccessLevel.PRIVATE)
public class NullSubstituteValueHandler {

    NullSubstituteValueConfiguration configuration;
    ConversionService mvcConversionService;

    public void setNullSubstituteValue(@NonNull Object object) {
        if (object instanceof Collection) {
            ((Collection<?>) object).forEach(this::setNullSubstituteValue);
            return;
        }
        if (object instanceof Map) {
            ((Map<?, ?>) object).values().forEach(this::setNullSubstituteValue);
            return;
        }
        ReflectionUtils.doWithFields(object.getClass(), field -> {
            ReflectionUtils.makeAccessible(field);
            Object value = ReflectionUtils.getField(field, object);
            if (Objects.isNull(value)) {
                Object nullSubstituteValue = getNullSubstituteValue(field);
                ReflectionUtils.setField(field, object, nullSubstituteValue);
            }
        }, field -> field.isAnnotationPresent(NullSubstituteValue.class));
    }

    public void clearNullSubstituteValue(@NonNull Object object) {
        if (object instanceof Collection) {
            ((Collection<?>) object).forEach(this::clearNullSubstituteValue);
            return;
        }
        if (object instanceof Map) {
            ((Map<?, ?>) object).values().forEach(this::clearNullSubstituteValue);
            return;
        }
        ReflectionUtils.doWithFields(object.getClass(), field -> {
            ReflectionUtils.makeAccessible(field);
            Object value = ReflectionUtils.getField(field, object);
            if (Objects.nonNull(value)) {
                Object nullSubstituteValue = getNullSubstituteValue(field);
                if (Objects.equals(value, nullSubstituteValue)) {
                    ReflectionUtils.setField(field, object, null);
                }
            }
        }, field -> field.isAnnotationPresent(NullSubstituteValue.class));
    }

    public <T extends Enum<T>> Object getNullSubstituteValue(@NonNull Field field) {
        ReflectionUtils.makeAccessible(field);
        NullSubstituteValue annotation = field.getAnnotation(NullSubstituteValue.class);
        Object nullSubstituteValue = null;
        //没有填写value时
        if (annotation.value().isEmpty()) {
            //类型在内置map中有定义的,以map中的默认值为准
            if (configuration.containsNullSubstituteValue(field.getType())) {
                nullSubstituteValue = configuration.getNullSubstituteValue(field.getType());
            }
        } else {
            //填写了value时将value以Spring内置转换api从String转换为属性类型
            nullSubstituteValue = annotation.value();
            //如果有中转类型,先转为中转类型
            for (Class<?> transferClass : annotation.transferClasses()) {
                nullSubstituteValue = mvcConversionService.convert(nullSubstituteValue, transferClass);
            }
            nullSubstituteValue = mvcConversionService.convert(nullSubstituteValue, field.getType());
        }
        return nullSubstituteValue;
    }
}
```

## null值替代值mybatis拦截器
```java
@Component
@Intercepts({
        @Signature(type = Executor.class, method = "update", args = {MappedStatement.class,
                Object.class})
})
@Slf4j
public class NullSubstituteValueInsertInterceptor implements Interceptor {
    @Resource
    NullSubstituteValueHandler nullSubstituteValueHandler;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        MappedStatement mappedStatement= (MappedStatement) invocation.getArgs()[0];
        if(SqlCommandType.INSERT.equals(mappedStatement.getSqlCommandType())){
            Object param= invocation.getArgs()[1];
            nullSubstituteValueHandler.setNullSubstituteValue(param);
            Object result = invocation.proceed();
            nullSubstituteValueHandler.clearNullSubstituteValue(param);
            return result;
        }
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}
```
```java
@Component
@Intercepts({
        @Signature(type = ResultSetHandler.class, method = "handleResultSets", args = {Statement.class})
})
@Slf4j
public class NullSubstituteValueSelectInterceptor implements Interceptor {
    @Resource
    NullSubstituteValueHandler nullSubstituteValueHandler;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        Object result = invocation.proceed();
        if (result instanceof Collection) {
            nullSubstituteValueHandler.clearNullSubstituteValue(result);
        }
        return result;
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}
```
```java
@Component
@Intercepts({
        @Signature(type = StatementHandler.class, method = "parameterize", args = {Statement.class})
})
@Slf4j
public class NullSubstituteValueUpdateInterceptor implements Interceptor {
    private static final String STATEMENT_HANDLER_MAPPED_STATEMENT_FIELD_NAME = "delegate.mappedStatement";
    @Resource
    NullSubstituteValueHandler nullSubstituteValueHandler;

    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        MappedStatement mappedStatement = (MappedStatement) SystemMetaObject.forObject(statementHandler)
                .getValue(STATEMENT_HANDLER_MAPPED_STATEMENT_FIELD_NAME);
        if (SqlCommandType.UPDATE.equals(mappedStatement.getSqlCommandType())) {
            Object param = statementHandler.getParameterHandler().getParameterObject();
            nullSubstituteValueHandler.setNullSubstituteValue(param);
            Object result = invocation.proceed();
            nullSubstituteValueHandler.clearNullSubstituteValue(param);
            return result;
        }
        return invocation.proceed();
    }

    @Override
    public Object plugin(Object target) {
        return Plugin.wrap(target, this);
    }

    @Override
    public void setProperties(Properties properties) {

    }
}
```