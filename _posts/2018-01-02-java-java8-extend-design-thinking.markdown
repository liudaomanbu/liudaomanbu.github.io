---
layout:     post
title:      "关于Java单继承机制与多接口机制设计的思考"
subtitle:   "由Java8对interface增加default方法引发的Java继承体系设计的思考"
date:       2018-1-2
author:     "caotc"
header-img: "img/home-bg.jpg"
tags:
    - Java
    - java
    - Java8
    - java8
    - extend
    - extends
    - implement
    - implements
    - interface
    - default method
    - Default Method
---
# Java的单继承(extends)机制与多接口(interface)机制
## 一、含义
众所周知,**Java中的继承体系是单继承的,即只能继承一个类(Class).**

也就是说一个类只会有一个父类(Parent Class),如果不断追溯一个类的父类,将该类和其所有超类(Super Class)用类关系图展示,那么将会是一个线型的类关系图.

然而无论是在现实世界中还是在代码世界中,都确实存在一个对象(Object)既是A类又是B类,且A类和B类没有父子关系的情况.

如果只有单继承的继承体系,那么遇到这种情况时会令人感觉束手无策.

以我们熟悉的JDK源码中的常见接口为例:

Comparable接口表示该类型能进行比较动作,是可比较的.

Serializable接口表示该类型能进行序列化和反序列化动作,是可存储的.

假如没有接口机制存在,Comparable和Serializable只能被定义为类,且只允许单继承,那么你会瞬间发现你陷入了巨大的困境.

显然,如Integer这样一个表示整数的类,既应该是可以被比较大小的,也应该是可以被序列化存入存储设备中的.

但是由于只能单继承,Integer类没法办到这个合理的需求,因为它不能同时拥有Comparable和Serializable两个超类.

也许你会认为,那将Comparable和Serializable两个类定义为父子关系,然后Integer继承其中的子类不就解决了吗?

然而这样做是不行的.**面向对象语言(OOP language)的类关系设计原则之一就是"只有当每个B确实也是A"的时候,才能将B类定义为A类的子类.**

之所以有这样的原则,自然是有原因的.

在上面的例子中,如果我们把Comparable类定义为Serializable类的子类,然后让Comparable类成为Integer类的超类,那么当我们需要增加一个新的可比较的类,但是却并不打算让其支持序列化操作时,你就立刻重新陷入了僵局.

如果你觉得上面的例子Comparable和Serializable的含义让你很难映射到现实世界中进行理解.

那么你可以想象一下,现实世界中一个人拥有太多的身份.

即使只是职业方面,一个音乐人可能既是歌手(Singer)也是作曲家(Songwriter),也有可能只是一个歌手或者只是一个作曲家.

从以上的例子中,我们可以看出,**在一个面向对象语言中,让一个类有多个类型(type)的机制是必要的.**

**因此,Java设计了接口这个机制,允许一个类实现(implements)多个接口,让一个类有了多个类型.**

## 二、多继承机制的问题

上面我们说过,在一个面向对象语言中,让一个类有多个类型的机制是必要的.

如果考虑到这一点的话,将继承机制设计为多继承是顺理成章的第一反应,例如C++就是这样设计的.

那么为什么Java要设计为单继承+接口机制的模式呢?

网上许多博客文章在介绍Java的单继承机制时,总是用现实世界中人也只有一个父亲来做比喻.

事实上,这样的比喻不但没有增加理解的用处,反而产生了严重的误导.

因为面向对象语言中继承一词与现实世界中的父子关系相差甚远.

面向对象语言中继承父类的子类对象能直接使用父类的方法和属性.

现实世界中父亲拥有年龄属性,孩子难道继承了父亲的年龄属性吗?博尔特的孩子能直接像父亲一样跑出世界记录吗?

而且既然说父类是子类的父亲,那么母亲在哪里呢?

事实上,真正的多继承机制要处理的问题有很多.

### 1.声明多继承与实现多继承

假如现在允许多继承,让我们看以下的范例代码:
```java
class FlyAnimal{
  public void fly(){
    System.out.println("this is FlyAnimal fly");
  }
}

class Aircraft{
  public void fly(){
    System.out.println("this is Aircraft fly");
  }
}

public class ExtendTest extends FlyAnimal,Aircraft{

}
```
那么此时,我实例化了一个ExtendTest类的对象,然后让该对象调用fly方法,此时到底应该输出"this is FlyAnimal fly"还是"this is Aircraft fly"?

这就是**实现多继承,一个类拥有从多个父类继承的多个已经实现的相同签名方法(超类如果有同名方法会直接被父类覆盖,所以只需要考虑父类).**

**实现多继承设计中有着多个同名可调用方法的歧义性的问题需要解决.**

C++对此问题的解决办法是允许当前子类对象调用任何一个父类的任何方法,但是遇到歧义时编译报错,要求前面加上类名和域解析符::来明确指定调用的目标方法,消除歧义.

```java
interface FlyAnimal{
  void fly();
}

interface Aircraft{
  void fly();
}

public class ExtendTest implements FlyAnimal,Aircraft{

  @Override
  public void fly() {
    System.out.println("fly");
  }
}
```
而像上面代码这样的**声明多继承只允许有多个相同签名的方法声明,实现却只有一个,不会存在歧义.**

**Java8之前对于接口只能拥有方法声明和常量的设计,保证了只能存在声明多继承,避免了实现多继承情况的出现.**

### 2.成员命名冲突

**与实现多继承类似,多继承时,一个类会拥有从多个父类继承的多个同名成员变量,同样存在着歧义性的问题需要解决.**

C++对此问题的解决办法与实现多继承的解决办法相同,仍然是要求在有歧义时消除歧义.

### 3.构造函数执行顺序

构造函数是很重要很特殊的函数,意义不言而喻.

多继承机制下,**复杂的继承关系下,一个类实例化时的构造函数执行顺序难以处理.**

出于自由性和可用性考虑,一个类对于父类构造函数的执行顺序应该拥有指定的机制(C++就是用继承声明的顺序来指定构造函数执行顺序),然而复杂情况下,程序员仍旧难以理解整个实例化过程中的构造函数执行顺序,容易出错.

## 三、Java设计取舍的原因
Java设计取舍的原因不必猜测,看看Java之父James Gosling是怎么说的:[James Gosling on Java, May 1999](https://link.zhihu.com/?target=http%3A//www.artima.com/intv/gosling13.html)

>**Bill Venners:** In Java you included multiple inheritance of interface, but left out multiple inheritance of implementation. Were you trying to say anything to designers by making the interface a separate construct? Were you trying to say anything by leaving out multiple inheritance of implementation? How did that come about?
>
>**James Gosling:** It listened to people from the C++ and Objective-C camps, and I tried to sort out who had the most happy experiences. The Objective-C notion of a pure interface with no implementation seemed to have worked out really well for people. It avoids a lot of the sticky issues such as disambiguation that people get into in C++. It's still kind of messy. It's an area that I don't feel totally happy with.
>
>**Bill Venners:** In what way?
>
>**James Gosling:**  Another way for doing these things is a technique called delegation. Some ideas in the delegation camp felt good, but I never came up with anything that really worked. I ended up with the interface construct, which felt simple enough to be comprehensible, sophisticated enough to be useful. It also avoided any of the tar pits that the other folks got into.


上面提到的多继承机制时**实现多继承、成员命名冲突、构造函数执行顺序等问题在菱形继承等环境下更为复杂.**

虽然这些问题都能够解决(不然C++也没法存在),但是个人品味考虑,James Gosling还是认为能避免这些问题的单继承机制与多接口机制更好.

## 四、单继承机制与多接口机制的限制

上面说了那么多多继承机制下的问题,Java最后也选择了单继承机制与多接口机制,那么难道单继承机制与多接口机制没有任何问题吗?

当然不可能.

### 1.接口实现类的代码重复性

以下面的代码为例
```java
interface FlyAnimal{
  void fly();
}

class Dove implements FlyAnimal{

  @Override
  public void fly() {
    System.out.println("扇动翅膀飞行");
  }
}

class Parrot implements FlyAnimal{

  @Override
  public void fly() {
    System.out.println("扇动翅膀飞行");
  }
}
```
很显然,Dove(鸽子)类和Parrot(鹦鹉)类都能飞,所以都实现了FlyAnimal接口,但是实际上,两者的fly方法的实现内容是完全相同的重复代码.

也许有人会说,你应该定义一个Bird(鸟)类来实现FlyAnimal接口,Dove Class和Parrot Class直接继承Bird类就行了.

事实上,这的确是一个不错的解决方案.

**这种解决方案被称之为骨架实现现(skeletal implementation),JDK源码中就大量使用了这种方式.**

**骨架实现中还存在特例是简单实现(simple implementation),简单实现就像个骨架实现,这是因为它实现了接口,并且是为了继承而设计的,但是区别在于它不是抽象的：它是最简单的可能的有效实现.你可以原封不动地使用,也可以看情况将它子类化.**

比如AbstractCollection 、 AbstractSet 、 AbstractList等都是骨架实现类.

然而**这种方案并不是适用于任何情况.**

比如说以上面的代码为例,鸵鸟这个不会飞的鸟类,继承了Bird类之后就会拥有不应该拥有的fly方法.

而且蝙蝠是属于哺乳类的,显然不能继承Bird类,但是其fly方法的实现也是扇动翅膀飞行.

可以说,**在Java8之前,这种冗余性是不可避免的.**

**其根本原因就是fly方法的共同实现的代码就是应该属于FlyAnimal接口的,但是由于设计限制,FlyAnimal接口却不能放下本应该属于自己的代码,所以只能存在多份重复的冗余代码.**

### 2.接口的不可扩展性

如果说接口实现类的代码重复性问题只是不够优雅,麻烦点也能解决的话,另一个接口的不可扩展性问题就非常严重了.

**面向对象语言中的继承(这里的继承是广义的,包括Java的implements)特性本身就是就是破坏封装性的,继承关系的类之间会互相影响.**

**由于接口中不能含有方法的实现,只能含有方法的声明,这一限制使得一个接口对外发布后,完全失去了扩展性.**

一旦你给一个已经发布的接口增加了新的方法,那么所有的实现类都会遭到破坏.

骨架实现类并不能对这个问题有多大的帮助,因为你没法保证所有接口实现类都继承了骨架实现类,而且正如上面所说,骨架实现类并不适用所有情况.

Java向来是以兼容性著称的,也向来是将兼容性的优先级放的很高的.

为了兼容性,Java没有像C#一样对于泛型特性进行了深度改造,而是选择用擦除式泛型实现泛型特性.

现在反而任何一个接口都能造成不兼容,这自然是不能被容忍的,因此接口发布后实质上就等于不可进行任何更改了.

**这实质上就是要求你必须在初次设计的时候就保证接口是没有任何问题的,且预见了以后所有的需求.**

**这已经不是难度的问题了,而是完全不可能完成的任务.**

**即使是Java委员会这样全是优秀的团队,依然没法完成这点,更别提其他程序员了.**

## 五、默认方法(default method)

**Java8为了兼顾兼容性和接口的增加方法,被迫加入了默认方法(default method)的特性,允许接口定义默认方法的方法实现.**

还记得我们上面说过的多继承机制问题之一的实现多继承吗?

**Java8加入了默认方法机制后不可避免地需要解决曾经千方百计避免的实现多继承方法歧义性问题.**

>1)类中的方法优先级最高.类或父类中声明的方法的优先级高于任何声明为默认方法的优先级.
>
>2)如果无法依据第一条进行判断,那么子接口的优先级最高：函数签名相同时,优先选择拥有最具体实现的默认方法的接口,即如果B继承了A,那么B就比A更具体.
>
>3)最后,如果还是无法判断,继承了多个接口的类必须显式覆盖和调用期望的方法,显式地选择使用哪一个默认方法的实现

**Java8采用如上的规则来处理实现菱形继承下多继承方法歧义性问题.**

**Java8对此的设计是符合一直以来自身应对菱形继承和歧义性的设计思路的.**

也许有的人不知道Java什么时候需要解决菱形继承和歧义性的问题.

事实上这个特性你天天都在使用,那就是**重载方法参数匹配优先级**.

```java
public class Overload {
	
	public static void say(Object arg) {
		System.out.println("Object");
	}
	
	public static void say(String arg) {
		System.out.println("String");
	}
	
	public static void main(String[] args) {
        say("a");
	}

}
```
上面代码就涉及到了Java中重载方法参数匹配优先级的问题.

Object是String的父类,当调用say方法的传参为一个字符串时,那么参数条件必定是同时符合两个方法的参数需求的,那么此时该调用哪个方法就涉及到重载方法参数匹配优先级.

**Java中对于重载方法参数匹配优先级中的引用类型规则总结如下:**

**(1)菱形继承中重复implements的接口以所有实现类中最上级的实现类为准,评级是最上级的实现类的上一级**

**(2)Object类为例外,不与任何接口或者类同级,默认为单独的最上级.**

**(3)从继承树由下往上进行匹配,如果当前调用方法的传参的最高优先级中有平级的多个方法存在,提示编译错误.**

其他关于重载方法参数匹配优先级的内容在下篇博客中讲解.

怎么样,上面关于引用类型的重载方法参数匹配优先级规则是不是和多继承方法优先级规则很像?

**同样都是先以规则确定菱形继承情况下的继承树,然后从继承树由下往上,最高优先级不唯一时报错.**

## 六、接口与抽象类(abstract class)的区别

截止到Java8为止,接口与抽象类仍然有不少区别.

(1)抽象类有构造函数,接口没有.

(2)抽象类的方法可以定义为任何权限的,接口方法权限只能为public(Java9中可以为private).

(3)抽象类的方法可以定义为final,防止被重写,接口不行.

(4)抽象类中可以有状态(成员变量),接口只能有静态常量.

**目前来说接口最大的限制,也是和抽象类最大的区别,是不能有状态.**

## 七、接口未来演进的思考

Java引入默认方法机制证明了人类的预见性是有限的.

Java8才妥协性地引入默认方法机制进一步证明了**人类预见性的局限性**,按理想来说早在使用了不需要状态支撑的骨架实现类时,就应该意识到这个问题了.

事实上,**《Effective Java》早就在第18条：接口优于抽象类中提出过这方面的担忧,并且建议:**

>简而言之，接口通常是定义允许多个实现的类型的最佳途径。这条规则有个例外，即当演变 的容易性比灵活性和功能更为重要的时候。在这种情况下，应该使用抽象类来定义类型，但 前提是必须理解并且可以接受这些局限性。

从这一点来看,**Java为了避免多继承机制而选择的设计思路是否可能本身就是不现实的?**

假如未来某些接口需要增加的方法需要状态的支撑,是否要么只能放弃,要么就进一步妥协,为接口增加状态?

如果为接口增加了状态,那么实际上就已经成为多继承机制了,上面说的多继承机制的问题还是一个不少的需要解决.

毕竟另一门JVM语言Scala就是如此.

# 参考列表
[《Effective Java》中文版 第2版 Joshua Bloch(美)]()

[Java 为什么不支持多继承？ RednaxelaFX](https://www.zhihu.com/question/24317891/answer/65097560)
