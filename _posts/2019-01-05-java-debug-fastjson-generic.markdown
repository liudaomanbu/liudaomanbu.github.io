---
layout:     post
title:      "一次线上内存泄漏问题的排查"
subtitle:   "与FastjsonHttpMessageConverter有关的内存泄漏问题"
date:       2019-01-05
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - Java
    - java
    - SSM
    - ssm
    - Bug
    - bug
    - Fastjson
    - fastjson
    - FastjsonHttpMessageConverter
    - fastjsonHttpMessageConverter

---
# 与FastjsonHttpMessageConverter有关的内存泄漏问题
## (一). 排查经过
1.首先确定有内存泄漏问题的是因为报了这个异常:java.lang.OutOfMemoryError:**GC overhead limit exceeded**
默认情况下，**当应用程序花费超过98%的时间用来做GC并且回收了不到2%的堆内存时**，会抛出java.lang.OutOfMemoryError:GC overhead limit exceeded错误。具体的表现就是你的应用几乎耗尽所有可用内存，并且GC多次均未能清理干净。

2.由于本地之前开代码跑过一个晚上都没内存泄漏,所以不方便在本地用更加方便的visualVm来排查内存泄漏.

所以直接上服务器使用jmap查看内存泄漏时内存情况
![image](/img/in-post/2019-01-05-java-debug-fastjson-generic/内存泄漏时内存情况.png)
可以明显看到上图中占内存最多的除了sun包的反射相关以外,最接近我们的使用的就是com.alibaba.fastjson的东西,所以肯定是怀疑fastjson内部有bug或者我们使用不当造成fastjson内部对于反射的东西内存泄漏了.

3.直接尝试去fastjson的github的issue中搜索内存泄漏的相关issue.
![image](/img/in-post/2019-01-05-java-debug-fastjson-generic/fastjson的内存泄漏issue.png)
https://github.com/alibaba/fastjson/issues/1418
这个issue提到了ParserConfig的IdentityHashMap和com.alibaba.fastjson.util.ParameterizedTypeImpl共同造成的内存泄漏.
![image](/img/in-post/2019-01-05-java-debug-fastjson-generic/IdentityHashMap和ParameterizedTypeImpl.png)
正好这个和目前从jmap中看到的情况非常相符,所以从这个方面入手排查.

4.查看ParserConfig的com.alibaba.fastjson.util.IdentityHashMap.
```java
public class IdentityHashMap<K, V> {
    private final Entry<K, V>[] buckets;
    private final int           indexMask;
    public final static int DEFAULT_SIZE = 8192;

    public IdentityHashMap(){
        this(DEFAULT_SIZE);
    }

    public IdentityHashMap(int tableSize){
        this.indexMask = tableSize - 1;
        this.buckets = new Entry[tableSize];
    }

    public final V get(K key) {
        final int hash = System.identityHashCode(key);
        final int bucket = hash & indexMask;

        for (Entry<K, V> entry = buckets[bucket]; entry != null; entry = entry.next) {
            if (key == entry.key) {
                return (V) entry.value;
            }
        }

        return null;
    }

    public Class findClass(String keyString) {
        for (int i = 0; i < buckets.length; i++) {
            Entry bucket = buckets[i];

            if (bucket == null) {
                continue;
            }

            for (Entry<K, V> entry = bucket; entry != null; entry = entry.next) {
                Object key = bucket.key;
                if (key instanceof Class) {
                    Class clazz = ((Class) key);
                    String className = clazz.getName();
                    if (className.equals(keyString)) {
                        return clazz;
                    }
                }
            }
        }

        return null;
    }

    public boolean put(K key, V value) {
        final int hash = System.identityHashCode(key);
        final int bucket = hash & indexMask;

        for (Entry<K, V> entry = buckets[bucket]; entry != null; entry = entry.next) {
            if (key == entry.key) {
                entry.value = value;
                return true;
            }
        }

        Entry<K, V> entry = new Entry<K, V>(key, value, hash, buckets[bucket]);
        buckets[bucket] = entry;  // 并发是处理时会可能导致缓存丢失，但不影响正确性

        return false;
    }

    public void clear() {
        Arrays.fill(this.buckets, null);
    }
}
```
发现其内部使用的是System.identityHashCode.

```java
/**
     * Returns the same hash code for the given object as
     * would be returned by the default method hashCode(),
     * whether or not the given object's class overrides
     * hashCode().
     * The hash code for the null reference is zero.
     *
     * @param x object for which the hashCode is to be calculated
     * @return  the hashCode
     * @since   JDK1.1
     */
    public static native int identityHashCode(Object x);
```
这个方法注释说明这是Object的hashCode方法的默认实现.
那也就意味着实际上这判断的是是否同一个对象,而不是我们通常意义上Map中的对象是否equals.
很有可能内存泄漏就发生在这点上.
经过debug发现果然,这个IdentityHashMap的元素无限增多,很显然发生了内存泄漏.
查看IdentityHashMap无限缓存的元素都是什么对象,最后定位到***.json.fastjson.codec.ResultDeserializer类中.

```java
public class ResultDeserializer implements ObjectDeserializer {

    public static final ResultDeserializer instance = new ResultDeserializer();

    @Override
    public <T> T deserialze(DefaultJSONParser parser, Type type, Object fieldName) {
        if (type instanceof ParameterizedType) {
            ParameterizedType pType = (ParameterizedType) type;
            return (T) parser.parseObject(
                    new ParameterizedTypeImpl(
                            pType.getActualTypeArguments(),
                            pType.getOwnerType(), DefaultResult.class),
                    fieldName);
        }
        return (T) parser.parseObject(DefaultResult.class, fieldName);
    }

    @Override
    public int getFastMatchToken() {
        return 0;
    }

}
```
显然每次new新的ParameterizedTypeImpl对象就是造成内存泄漏的原因.
## (二). 总结
1. 本次内存泄漏的原因由fastjson过于追求性能,在内部缓存时使用的map并没有遵照map接口规范来对待hashCode和convention框架中没有正确使用fastjson共同造成.

2. 本地之所以跑了一晚上都没有OOM是有两个原因.一是因为本地内存远比服务器大,光是空余内存就超过7G,而服务器总共才4G.二是因为服务器多核运算效率远比本地快,而且这个内存泄漏是会不断降低反序列化效率的,所以越到后面泄漏的速度越慢.

3. 之前一直没有暴露的原因是之前没有使用Result接口作为返回结果对象并且使用fastjson来进行反序列化.