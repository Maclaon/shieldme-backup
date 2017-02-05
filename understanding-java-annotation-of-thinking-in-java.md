---
title: Java编程思想之注解
date: 2013-04-22 10:00:00
tags: Java编程思想，注解，解耦
author: maclaon
comments: false
---
## 注解
> 也被称作: 元数据，**将任何的信息或元数据（metadata）与程序元素（类、方法、成员变量等）进行关联，存储有关程序的额外信息**，并通过对这些额外的信息的捕捉（注解解析器）来对执行的代码做一些额外的操作。

### 原理
+ Annotation其实是一种接口，通过Java的反射机制相关的API来访问annotation信息。相关类（框架或工具中的类）根据这些信息来决定如何使用该程序元素或改变它们的行为。
+ Annotation是不会影响程序代码的执行，无论annotation怎么变化，加入注解的代码都始终如一地执行。

### 优点
+ 干净易读的代码
+ **编译器类型检查**

<!--more-->

### 定义
``` Java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @inteface Cache {
	public String name();
	public String description() default "local cache";
}
```
其中@Target、@Retention是定义注解的一些**元注解**，注解中会包含一些元素以表示某些值，当分析处理注解时，程序或工具可以利用这些值，根据特定的值，来生成不同的行为或者动作，比如`SpringMVC`中的`RequestMapping(vaule = "/saveContentTag/")`等的设置，约束了该请求链接是如何到达`Spring`中的。

> 通常情况下，需要根据业务场景进行抽象，定义自己的注解和注解处理器来进行处理。

注解的元素要么具有默认提供的值，要么在使用注解的时提供元素的值，不允许存在不确定的值，同时默认值和注解提供的时候都不能以`null`作为其值，一般可用`-1`或者空的("")字符串来标识。如果注解中的元素是以字符串$$value$$进行定义的，那么久可以像使用构造函数那样进行对注解进行初始化。

### 元注解

|元注解|说明|值域|
|:---:|:---:|:---|
|@Target|表示该注解用于什么地方|构造器声明:CONSTRUCTOR、域声明:FIELD、局部变量声明:LOCAL_VARIABLE、方法声明:METHOD、包声明:PACKAGE、参数说明:PARAMETER、类，接口或`enum`声明:TYPE|
|@Retention|表示需要在什么级别保存该注解信息|SOURCE、CLASS、RUNTIME|
|@Documented|将此注解包含在Javadoc中||
|@Inherited|允许子类继承父类中的注解|这里指的是基于类继承关系下的注解，注解本身是不能被继承，所以也没有多态的性质|

### 注解解析器
注解必须与注解解析器同时存在，否则注解和注释没有啥区别，也不会起到任何的作用。注解解析器主要用到**反射**的原理，通过`getDeclaredMethods()`和`getAnnotation()`以及`getDeclaredFiedls()`方法来获取方法，注解和属性值，以此来获取相关值，并根据相关值进行额外的业务处理。

### 场景
> + 是供指定的**工具或者框架来使用的**，在对外提供接口的时候，可以考虑用注解来简化代码。
> + 每当你创建描述符性质的类或接口时，一旦其中包含了重复性的工作，就可以考虑使用注解来简化与自动化该过程(**这里的描述符性质，还没理解透!**)。比如说缓存通用的机制，数据库字段范围的注解。