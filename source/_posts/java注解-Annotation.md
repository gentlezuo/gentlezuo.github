---
title: java注解-Annotation
date: 2019-04-13 10:43:33
tags: java
category: java
---

# java注解

概念：注解用于给java代码添加元数据，在编译或者运行时解析处理这些元数据。注解可用于包，类，字段，方法，参数等。可以将理解为给这些包或类等等添加一个标签。

## java内置注解

1. @Override：表示当前的方法定义将覆盖父类中的方法
2. @Deprecated：表示代码被弃用，如果使用了被@Deprecated注解的代码则编译器将发出警告
3. @SuppressWarnings：表示关闭编译器警告信息


## 自定义注解
<!--more-->
注解也属于一种类型,就像类,接口一样，可以自定义，格式如下

~~~java
[@Target]
[@Retention]
[@Documented]
[@Inherited]
public @interface [名称] {
    // 元素
}
~~~

使用注解有几个条件：注解声明、使用注解的元素、操作注解使其起作用(注解处理器)。  

一个例子:   
创建一个@Bean的注解
~~~java

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Bean {
    String value();
    //也可以设置默认值
    //String value() default "entity";
}
~~~
测试注解
~~~java

@Bean("abc")
public class AnnotationDemo {

    public static void main(String[] args) {

        AnnotationDemo a=new AnnotationDemo();
        Class cls=AnnotationDemo.class;
        //使用反射获取注解
        Bean annotation=(Bean) cls.getAnnotation(Bean.class);
        System.out.println(annotation.value());
    }
}
~~~
输出
~~~
abc
~~~

当然这只是一个例子,没有任何实际意义,注解处理器远不止这么简单,与实际的逻辑相关,就如同Spring中的各种注解一样,Spring在处理这些带有注解的类时非常复杂   

### 元注解

元注解就是注解其他注解的注解   
在java中有4个元注解


| 注解       | 作用                                           | 取值   |
| ---------- | ---------------------------------------------- | ------ |
| @Target    | 设置注解可以应用的位置，比如在类上，还是方法上 | 下文中 |
| @Retention | 设置注解的生存时间                             | 下文中 |
| @Document  | 说明注解是否能被文档化                         | 无     |
| @Inhrited  | 说明注解能否被继承                             | 无     |


- @Target取值
  - ElementType.CONSTRUCTOR 可以给构造方法进行注解
  - ElementType.FIELD 可以给属性进行注解
  - ElementType.LOCAL_VARIABLE 可以给局部变量进行注解
  - ElementType.METHOD 可以给方法进行注解
  - ElementType.PACKAGE 可以给一个包进行注解
  - ElementType.PARAMETER 可以给一个方法内的参数进行注解
  - ElementType.ANNOTATION_TYPE 可以给一个注解进行注解
  - ElementType.TYPE 可以给一个类型进行注解，比如类、接口、枚举
- Retention:取值
  - RetentionPolicy.SOURCE阶段(java源文件阶段)
  - RetentionPolicy.CLASS阶段(class文件阶段)
  - RetentionPolicy.RUNTIME阶段(内存中的字节码运行时阶段)
  - 默认是在RetentionPolicy.CLASS阶段

此时可以明白上述Bean注解应用于类，周期是运行时期

### 注解支持的属性
1. 所有基本数据类型（int,float,boolean,byte,double,char,long,short) 
2. String类型 
3. Class类型 
4. enum类型 
5. Annotation类型 
6. 以上所有类型的数组

代码示例：
~~~java
//如果没有@Retention(RetentionPolicy.RUNTIME)，在反射时拿不到相应的值，会抛空指针异常，因为默认注解生存时间是class文件
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Bean2 {

    String a();
    int b();
    boolean c();
    long[] d();

}
~~~

测试
~~~java

@Bean2(a="aaaa",b=99,c=false,d={1,3,5,7,9})
public class AnnotationDemo2 {

    public static void main(String[] args) {
        Class cls=AnnotationDemo2.class;
        Bean2 bean2=(Bean2) cls.getAnnotation(Bean2.class);
        System.out.println(bean2.a());
        System.out.println(bean2.b());
        System.out.println(bean2.c());
        System.out.println(Arrays.toString(bean2.d()));
    }
}
~~~
控制台打印
~~~
aaaa
99
false
[1, 3, 5, 7, 9]
~~~

