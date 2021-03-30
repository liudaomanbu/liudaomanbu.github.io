---
layout:     post
title:      "debug记录:eclipse违反java8方法引用规范的bug"
subtitle:   "由一个eclipse bug学习java8方法引用规范"
date:       2018-3-6
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - Java
    - java
    - Java8
    - java8
    - Debug
    - debug
    - Bug
    - bug
    - eclipse
    - Eclipse
    - 方法引用
    - 函数引用
    - Method References
    - method references
---


今天,在公司项目中修复一个Bug时遇到了百思不得其解的事情.

事情的起源是测试上报了一个Bug,这个Bug的操作很简单,在测试环境下确实存在,但是在**本地的开发环境中无法复现**.

于是我首先怀疑开发环境和测试环境的代码版本不同导致了这种事情.但是一系列操作之后,我确认了并**不是测试环境与本地开发环境的代码版本区别**.

由于服务器上的日志无法准确定位出错位置,显然Bug出在我没有预料的地方.我只好通过逐渐**增加日志的方式定位了具体报错的代码位置和发生情况**.

报错的代码逻辑非常简单.
```java
Optional.ofNullable(a).orElseGet(b::getName)
```
日志显示,在服务器上**当a变量不为null,而b变量为null**的情况出现时,就会发生NullPointException,而**本地的开发环境在这种情况下代码正常运行.**

似乎,服务器似乎在a变量不为null时,依然试图执行b.getName(),导致出现了NullPointException.

然而查看jdk中有关Optional的源码
```java
public T orElseGet(Supplier<? extends T> other) {
        return value != null ? value : other.get();
    }
```
orElseGet函数内部是使用三元表达式实现的,按照Java规范来说,**此时的other.get()代码根本不应该被执行**.

**我为了确认在服务器上产生Bug的原因是否如我所想,我试图在整个功能逻辑代码前加入一段类似的测试代码,保证a变量值不为null,而b变量值为null.**

**最令我难以理解的事情发生了,本地环境下出现了诡异的情况:
测试代码正常执行通过,但是后面的功能逻辑代码中的那行Optional代码出现了和服务器同样的Bug.
然后我删掉了测试代码,然而后面的功能逻辑代码中的那行Optional代码依然和服务器的表现一样,并没有回到原来能正常运行的状态.**

至此,也算是以意想不到的方式完成了Bug在开发环境的复现.
我试图断点进入orElseGet方法内部跟踪执行,然而在这行代码中**debug过程出现异常,无法断点进入orElseGet方法内部**.

**最后我只好将orElseGet写成orElse(Optional.ofNullable(b).map(B::getName).orElse(null))这种形式,然后修复了Bug.**

但是在**本地开发环境,这行代码正常运行好一段时间**了,使用中都是有遇到a不为null,b为null的情况的,一直没发生NullPointException.
直到我为了debug,试图加入了相似的测试代码,本地的这行代码也出现了NullPointException,并且怎么还原版本都无法正常运行.

我简直要相信玄学了.

好吧,事已至此,我们只好去**看看[Java规范](https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.13.3)对于这种情况怎么说**的.

>15.13.3. Run-Time Evaluation of Method References
>
>At run time, evaluation of a method reference expression is similar to evaluation of a class instance creation expression, insofar as normal completion produces a reference to an object. Evaluation of a method reference expression is distinct from invocation of the method itself.
>
>First, if the method reference expression begins with an ExpressionName or a Primary, this subexpression is evaluated. **If the subexpression evaluates to null, a NullPointerException is raised, and the method reference expression completes abruptly. If the subexpression completes abruptly, the method reference expression completes abruptly for the same reason.**

那么上面的**ExpressionName and Primary**具体指什么呢?

>15.13. Method Reference Expressions
>
>MethodReference:
>
>ExpressionName :: [TypeArguments] Identifier 
>
>ReferenceType :: [TypeArguments] Identifier 
>
>Primary :: [TypeArguments] Identifier 
>
>super :: [TypeArguments] Identifier 
>
>TypeName . super :: [TypeArguments] Identifier 
>
>ClassType :: [TypeArguments] new 
>
>ArrayType :: new

也就是说**根据Java规范,当方法引用的左边的表达式解析结果为null时,就应该直接抛出NullPointerException,不管最后这个方法引用创建的匿名对象(即orElseGet方法中的other对象)有没有被使用执行**.

个人认为这**可能是出于简单的fail fast的考虑,认为反正方法引用创建的匿名对象被使用执行代码时,内部也会抛出NullPointerException,还不如直接提早抛了**.

但是不知道设计者是否没考虑到本文这种情况,如果执行时才抛出NullPointerException写法将简单的多.
现在只能被迫将**写法改成我上面所写的给b变量再包一层Optional或者将方法引用改写为()->b.getName()的形式才能避免NullPointerException**.

那么从Java规范的规定来回顾上面的debug过程.

**服务器的编译结果一直符合Java规范,而本地的开发环境之前的正常运行是不符合Java规范的.**

重新增加测试代码.
```java
	String a="sssss";
	File b=null;
	String c=Optional.ofNullable(a).orElseGet(b::getName);
```
代码执行通过,没有按Java规范抛出NullPointerException.会不会是里面有dead code所以编译的时候优化了什么导致不符合Java规范呢?

修改测试代码.
```java
    String a="sssss";
    File b=Objects.isNull(fileId)?file:null;
    String c=Optional.ofNullable(a).orElseGet(b::getName);
```
fileId由外部传入,消除dead code的警告,再次测试.结果依然是代码执行通过.

我开始猜想这也许是ide的bug,顺带一说我使用的开发IDE是eclipse.

google搜索"Run-Time Evaluation of Method References".[第一条stackoverflow的结果](https://stackoverflow.com/questions/47816807/runtime-evaluation-of-expressions-in-java-method-references)的提问者就问了和我类似的问题.

下面有人回复
>Oh darn! That's an eclipse bug (ECJ), it fails with javac/java (I've tried 9 here in the example, but same thing happens in 8):

对上了,应该没错,我就是遇到了eclipse的bug.

**总结,eclipse存在Method References没有按Java规范处理的bug,所以产生了开发环境和服务器环境代码行为不一致的问题.**

**eclipse开发环境中,orElseGet(b::getName)代码的一直正常运行是eclipse的bug,debug过程中在整个功能逻辑代码前加入一段类似的测试代码后,开发环境的orElseGet(b::getName)代码符合Java规范地抛出了NullPointerException应该是eclipse又发生了bug,误打误撞地符合了Java规范.**

注:询问[冰冰大佬](http://ice1000.org/)后得知,Eclipse直接用自己的编译器代替Javac,IDEA在编译的时候直接调用编译器的API编译而不是命令行,然后再给你注入一堆方便你调试用的额外字节码,所以感性地推断可以知道IDEA没有这个bug.