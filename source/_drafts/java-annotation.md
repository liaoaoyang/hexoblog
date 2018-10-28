title: Java自定义注解（使用篇）
date: 2018-10-24 23:53:01
categories: Java
tags: [Java]
---

# TL;DR

Java 注解广泛运用在开发之中，用于增强变量/方法/类等。

尝试说明 Java 自定义注解的使用，以及通过开源项目中的使用进行说明。

<!-- java-aonnotaion -->
<!-- more -->

本文主要记录个人的理解，全文基于Java SE8。

# 自定义注解

自定义注解分为两个部分：`注解声明`和`注解处理逻辑`。

每个注解可以有多个属性值，同名注解通过声明后可以在对象上使用多个。

## 注解结构

### 定义注解

用以下实例说明：

```
@Repeatable(LearnRepeatableAnnotation.class)
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.LOCAL_VARIABLE, ElementType.METHOD})
public @interface LearnAnnotation {
    String value() default "";

    String filedAnnotationValue() default "";

    Class<?> className() default Void.class;
}
```

逐行分析一下。

`@Repeatable(LearnRepeatableAnnotation.class)` 表示本注解可以在一个对象上使用多次，具体内容下文会具体说明。

`@Retention(RetentionPolicy.RUNTIME)` 是一个元注解，表示注解可以在运行时通过反射使用，元注解下文会具体说明。

`@Target({ElementType.FIELD, ElementType.LOCAL_VARIABLE, ElementType.METHOD})` 也是一个元注解，表示注解可以在属性、本地变量、方法上。

`public @interface LearnAnnotation` 表示这是一个注解声明，注解需要以`@interface`声明。

`String value() default "";` 表示注解的值域是字符串类型，默认为空字符串。注解使用时，可以通过`属性名=值`的形式进行赋值，如果不声明属性名，说明会赋值到`value`属性上。注解中的属性名就是声明中的方法名。

`String filedAnnotationValue() default "";` 表示自定义注解`@LearnAnnotation`有一个名为`filedAnnotationValue`的字符串属性，使用时可以通过`@LearnAnnotation(filedAnnotationValue = "NAME")`这一形式使用。

`Class<?> className() default Void.class;` 表示自定义注解`@LearnAnnotation`有一个名为`className`的Class对象，此处需要注意，自定义注解的属性值只能是基本类型(`int`/`char`/`String`/`class`等)以及他们这些类型的数组。

注解如果没有default声明的，需要指定属性值后才能使用。

### 同一对象使用多个相同的注解声明

还是使用上述案例，第一行`@Repeatable(LearnRepeatableAnnotation.class)`通过声明利用`@LearnRepeatableAnnotation`这一注解表示可以在一个对象上使用多个`LearnAnnotation`注解。

`@LearnRepeatableAnnotation` 的实现如下：

```
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.LOCAL_VARIABLE, ElementType.METHOD})
public @interface LearnRepeatableAnnotation {
    LearnAnnotation[] value() default {};
}
```

即需要声明一个新类型的注解，且这一注解的值，是计划使用多个注解的数组。

实际使用中可以像如下形式使用：

```
@LearnAnnotation(filedAnnotationValue = "v1")
@LearnAnnotation(value = "v2")
private int testRepeatInt = 0;
```

使用多个同名注解，例如作为配置规则，可以让当前对象获取多个规则。

## 注解声明

注解声明又主要分为两个部分：元注解注解名称及字段定义。

### 元注解

Java 中提供了4种元注解：

+ @Documented - 在JavaDoc中提供注解信息
+ @Retention - 注解的生效范围
+ @Target - 注解允许使用的对象
+ @Inherited - 注解是否可以被子类继承

元注解是实现自定义注解的重要工具，最重要的是`@Retention`与`@Target`。

#### 元注解@Retention

元注解 `@Retention` 可以有如下3个属性值：

+ `RetentionPolicy.SOURCE` – 注解保留在源码中，编译阶段会被丢弃
+ `RetentionPolicy.CLASS` – 注解保留在`.class`文件中，但不会在运行时存在
+ `RetentionPolicy.RUNTIME` – 注解可以在运行时读取、使用反射可以获得

默认是`RetentionPolicy.CLASS`。

#### 元注解@Target

元注解 `@Target` 可以有如下8个属性值：

+ `ElementType.ANNOTATION_TYPE`
+ `ElementType.CONSTRUCTOR`
+ `ElementType.FIELD`
+ `ElementType.LOCAL_VARIABLE`
+ `ElementType.METHOD`
+ `ElementType.PACKAGE`
+ `ElementType.PARAMETER`
+ `ElementType.TYPE`

作用范围看名称基本都能对应上。默认是全部。

# 参考

+ [Lesson: Annotations (Oracle Java Documentation)](https://docs.oracle.com/javase/tutorial/java/annotations/index.html)

