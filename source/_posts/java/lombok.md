title: Lombok介绍
author: Figthing
tags:
  - java
  - lombok
categories:
  - java
date: 2018-02-12 11:25:00
---
### 背景
我们在开发过程中，通常都会定义大量的JavaBean，然后通过IDE去生成其属性的构造器、getter、setter、equals、hashcode、toString方法，当要对某个属性进行改变时，比如命名、类型等，都需要重新去生成上面提到的这些方法，那Java中有没有一种方式能够避免这种重复的劳动呢？答案是有，我们来看一下下面这张图，右面是一个简单的JavaBean，只定义了两个属性，在类上加上了@Data，从左面的结构图上可以看到，已经自动生成了上面提到的方法。 

![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/java/2.jpg)

### Lombok简介

ombok是一个可以通过简单的注解形式来帮助我们简化消除一些必须有但显得很臃肿的Java代码的工具，通过使用对应的注解，可以在编译源码的时候生成对应的方法。
- 官方地址：[https://projectlombok.org/](https://projectlombok.org/)
- github地址：[https://github.com/rzwitserloot/lombok](https://projectlombok.org/)

<!--more-->

### Lombok使用

#### 注解介绍

1. **@Getter / @Setter**
> 可以作用在类上和属性上，放在类上，会对所有的非静态(non-static)属性生成Getter/Setter方法，放在属性上，会对该属性生成Getter/Setter方法。并可以指定Getter/Setter方法的访问级别。

2. **@EqualsAndHashCode**
> 默认情况下，会使用所有非瞬态(non-transient)和非静态(non-static)字段来生成equals和hascode方法，也可以指定具体使用哪些属性。

3. **@ToString**
> 生成toString方法，默认情况下，会输出类名、所有属性，属性会按照顺序输出，以逗号分割。

4. **@NoArgsConstructor, @RequiredArgsConstructor and @AllArgsConstructor**
> 无参构造器、部分参数构造器、全参构造器，当我们需要重载多个构造器的时候，Lombok就无能为力了。

5. **@Data**
> @ToString, @EqualsAndHashCode, 所有属性的@Getter, 所有non-final属性的@Setter和@RequiredArgsConstructor的组合，通常情况下，我们使用这个注解就足够了。

### Lombok原理

了解了简单的使用之后，现在应该比较好奇它是如何实现的。整个使用的过程中，只需要使用注解而已，不需要做其它额外的工作，那玄妙之处应该是在注解的解析上。JDK5引入了注解的同时，也提供了两种解析方式。

### 运行时解析
运行时能够解析的注解，必须将@Retention设置为RUNTIME，这样可以通过反射拿到该注解。java.lang.reflect反射包中提供了一个接口AnnotatedElement，该接口定义了获取注解信息的几个方法，Class、Constructor、Field、Method、Package等都实现了该接口，大部分开发者应该都很熟悉这种解析方式。

```java
boolean isAnnotationPresent(Class<? extends Annotation> annotationClass);
<T extends Annotation> T getAnnotation(Class<T> annotationClass);
Annotation[] getAnnotations();
Annotation[] getDeclaredAnnotations();
```

### 编译时解析
编译时解析有两种机制，网上很多文章都把它俩搞混了，分别简单描述一下。

### Annotation Processing Tool

apt自JDK5产生，JDK7已标记为过期，不推荐使用，JDK8中已彻底删除，自JDK6开始，可以使用Pluggable Annotation Processing API来替换它，apt被替换主要有2点原因：

- api都在com.sun.mirror非标准包下
- 没有集成到javac中，需要额外运行

### Pluggable Annotation Processing API

JSR 269，自JDK6加入，作为apt的替代方案，它解决了apt的两个问题，javac在执行的时候会调用实现了该API的程序，这样我们就可以对编译器做一些增强，这时javac执行的过程如下： 

![](http://zhouqi-blog.oss-cn-shenzhen.aliyuncs.com/img/java/3.jpg)

### Lombok问题

- 无法支持多种参数构造器的重载
- 奇淫巧技，使用会有争议