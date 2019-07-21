---
title: java反射
date: 2019-04-11 21:30:05
tags:
- java
- JDK源码
category: java
---

# java反射

定义：JAVA反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法；对于任意一个对象，都能够调用它的任意一个方法和属性；这种动态获取的信息以及动态调用对象的方法的功能称为java语言的反射机制。

<!--more-->

反射多使用在框架中，控制反转，动态代理都需要使用到反射。

使用反射有一个前提：**得到对应字节码的Class对象**。因为要获取一个类的信息，就得通过Class对象。

反射就是把java类中的各种成分映射成一个个的Java对象   
例如：一个类有：成员变量、方法、构造方法、包等等信息，利用反射技术可以对一个类进行解剖，把各个组成部分映射成一个个对象。

## Class类
Class对象表示正在运行的程序中已经加载的类或者接口。jvm中有很多个Class对象，每一个类（包括八种基本类型）都有一个Class对象。Class对象是由jvm在类加载的时候创建的，所以没有构造方法，也不需要我们去构造。

通过Class对象可以获取某个类中的：构造方法、成员变量、成员方法；并访问成员；

### 获取Class对象的方法

三种方式：   
- 类名.class
- Object.getClass()
- Class的静态方法：Class.forName(String className)
通常使用第三种方式，可以和配置文件配合使用，更加方便    

以String类举例
~~~java
        //第一种方法
        Class clz=String.class;
        String s="adfgag";
        //第二种方法
        Class clz1=s.getClass();
        //第三种方法
        Class clz2=Class.forName("java.lang.String");
        System.out.println(clz);
        System.out.println(clz1);
        System.out.println(clz2);
        //一个类只有一个Class对象
        System.out.println(clz==clz2);
        System.out.println(clz1==clz2);
~~~
输出
~~~
class java.lang.String
class java.lang.String
class java.lang.String
true
true
~~~

### Class类的常用的方法
- 静态方法：forName(String className):返回与带有给定字符串名的类或接口相关联的 Class 对象。
- 静态方法：forName(String name, boolean initialize, ClassLoader loader):使用给定的类加载器，返回与带有给定字符串名的类或接口相关联的 Class 对象。initialize为true时会进行初始化。
- newInstance() 创建此 Class 对象所表示的类的一个新实例。
- getClassLoader() 返回该类的类加载器。
- getConstructor(Class<?>... parameterTypes) 返回一个 Constructor 对象，它返回此 Class 对象所表示的类的指定**公共**构造方法。
- getConstructors() 返回一个包含某些 Constructor 对象的数组，这些对象反映此 Class 对象所表示的类的所有公共构造方法。
- getDeclaredConstructors() 返回 Constructor 对象的一个数组，这些对象反映此 Class 对象表示的类声明的**所有**构造方法。
- getDeclaredField(String name) 返回一个 Field 对象，该对象反映此 Class 对象所表示的类或接口的指定已声明字段。
- getDeclaredFields() 返回 Field 对象的一个数组，这些对象反映此 Class 对象所表示的类或接口所声明的所有字段。
- getDeclaredMethods() 返回 Method 对象的一个数组，这些对象反映此 Class 对象表示的类或接口声明的所有方法，包括公共、保护、默认（包）访问和私有方法，但不包括继承的方法。
- getField(String name) 返回一个 Field 对象，它反映此 Class 对象所表示的类或接口的指定公共成员字段。
- getFields() 返回一个包含某些 Field 对象的数组，这些对象反映此 Class 对象所表示的类或接口的所有可访问公共字段。
- getInterfaces() 确定此对象所表示的类或接口实现的接口。
- getMethods() 返回一个包含某些 Method 对象的数组，这些对象反映此 Class 对象所表示的类或接口（包括那些由该类或接口声明的以及从超类和超接口继承的那些的类或接口）的公共 member 方法。
- getName() 以 String 的形式返回此 Class 对象所表示的实体（类、接口、数组类、基本类型或 void）名称。
- getResource(String name) 查找带有给定名称的资源。

**注意**：带有Declared的会返回所有权限修饰符修饰的，而不带Declared的只返回被public修饰的。

代码示例：
~~~java
        Class clz=String.class;
        Object obj=clz.newInstance();
        System.out.println(obj.getClass().getName());//java.lang.String
        Constructor constructor=clz.getConstructor(String.class);
        Constructor[] constructors=clz.getConstructors();
        Field field=clz.getDeclaredField("value");
        System.out.println(field.toString());//private final char[] java.lang.String.value
        Method[] methods=clz.getDeclaredMethods();
        for (Method m:methods){
            System.out.println(m.toString());
        }
~~~
输出
~~~
java.lang.String
private final char[] java.lang.String.value
public boolean java.lang.String.equals(java.lang.Object)
public java.lang.String java.lang.String.toString()
public int java.lang.String.hashCode()
public int java.lang.String.compareTo(java.lang.Object)
public int java.lang.String.compareTo(java.lang.String)
public int java.lang.String.indexOf(java.lang.String,int)
static int java.lang.String.indexOf(char[],int,int,java.lang.String,int)
static int java.lang.String.indexOf(char[],int,int,char[],int,int,int)
public int java.lang.String.indexOf(int)
public int java.lang.String.indexOf(java.lang.String)
public int java.lang.String.indexOf(int,int)
...
...
~~~

## Constructor类
Constructor 提供关于类的单个构造方法的信息以及对它的访问权限。

### 常用方法
- newInstance(Object... initargs) 使用此 Constructor 对象表示的构造方法来创建该构造方法的声明类的新实例，并用指定的初始化参数初始化该实例。

代码示例
~~~java
        Object obj2=constructor.newInstance("dfadfa");
        System.out.println(obj2);
~~~
输出
~~~
dfadfa
~~~

## Field类
Field 提供有关类或接口的单个字段的信息，以及对它的动态访问权限。反射的字段可能是一个类（静态）字段或实例字段。

Array 允许在执行 get 或 set 访问操作期间进行扩展转换。

### 常用方法
当获取到field对象时，可以使用以下方法来访问和设置字段

- setAccessible(boolean flag)，这是父类的方法，设置是否有权限访问非共有的字段，以下方法都只能对公有字段操作除非使用次方法。
- get(Object obj) 返回指定对象上此 Field 表示的字段的值。
- getDouble(Object obj) 获取 double 类型或另一个通过扩展转换可以转换为 double 类型的基本类型的静态或实例字段的值。当然还可以获取Int，Float等等
- getType() 返回一个 Class 对象，它标识了此 Field 对象所表示字段的声明类型。
- set(Object obj, Object value) 将指定对象变量上此 Field 对象表示的字段设置为指定的新值。
- setBoolean(Object obj, boolean z) 将字段的值设置为指定对象上的一个 boolean 值。当然也有其他基本类型的包装类。
- getModifiers() 以整数形式返回由此 Field 对象表示的字段的 Java 语言修饰符。
例子：

一个类：
~~~java

public class People {
    public int age;
    private String name;
    public boolean sex;
    public double moeny;

    public People(int age, String name, boolean sex, double moeny) {
        this.age = age;
        this.name = name;
        this.sex = sex;
        this.moeny = moeny;
    }
}
~~~
如果直接访问私有字段
~~~java
        Class clz=Class.forName("javaKnowledge.People");
        People p=new People(18,"zzz",true,10000.0d);
        //name是私有字段
        Field field1=clz.getDeclaredField("name");
        String name= (String) field1.get(p);
~~~
结果
~~~
Exception in thread "main" java.lang.IllegalAccessException: Class javaKnowledge.ReflectDemo can not access a member of class javaKnowledge.People with modifiers "private"
~~~

调用setAccessible(true)方法
~~~java
Class clz=Class.forName("javaKnowledge.People");
        //获取age字段
        Field field=clz.getField("age");
        People p=new People(18,"zzz",true,10000.0d);
        int a= (int) field.get(p);
        System.out.println(a);
        Field field1=clz.getDeclaredField("name");
        //设置访问权限
        field1.setAccessible(true);
        String name= (String) field1.get(p);
        System.out.println(name);
        //设置p对象name字段的值为“hhh”
        field1.set(p,"hhh");
        //验证结果
        System.out.println(p.getName());
~~~
结果
~~~
18
zzz
hhh
~~~

此时已经可以获取一个类的字段信息，构造器信息，以及构造一个对象，设置字段的值。

## Method类

Method 提供关于类或接口上单独某个方法（以及如何访问该方法）的信息。所反映的方法可能是类方法或实例方法（包括抽象方法）。

Method 允许在匹配要调用的实参与底层方法的形参时进行扩展转换.

### 常用方法
- invoke(Object obj, Object... args) 对带有指定参数的指定对象调用由此 Method 对象表示的底层方法。
- getDeclaringClass() 返回表示声明由此 Method 对象表示的方法的类或接口的 Class 对象。
- getParameterTypes() 按照声明顺序返回 Class 对象的数组，这些对象描述了此 Method 对象所表示的方法的形参类型。
- getReturnType() 返回一个 Class 对象，该对象描述了此 Method 对象所表示的方法的正式返回类型。


代码示例：
~~~java
        Method[] methods=clz.getMethods();
        for (Method m:methods){
            System.out.println("方法名："+m.getName()+"      所属类："+m.getDeclaringClass()+"      参数类型："+Arrays.toString(m.getParameterTypes())+"      返回类型"+m.getReturnType());
        }


        Method method=clz.getMethod("say",String.class);
        method.invoke(p,"hello ");
        Method method1=clz.getDeclaredMethod("eat",Integer.class);
        System.out.println(method1.toString());
        method1.setAccessible(true);
        System.out.println(method1.invoke(p,9));
~~~
结果
~~~
方法名：getAge      所属类：class javaKnowledge.People      参数类型：[]      返回类型int
方法名：setAge      所属类：class javaKnowledge.People      参数类型：[int]      返回类型void
方法名：isSex      所属类：class javaKnowledge.People      参数类型：[]      返回类型boolean
方法名：setSex      所属类：class javaKnowledge.People      参数类型：[boolean]      返回类型void
方法名：getMoeny      所属类：class javaKnowledge.People      参数类型：[]      返回类型double
方法名：setMoeny      所属类：class javaKnowledge.People      参数类型：[double]      返回类型void
方法名：say      所属类：class javaKnowledge.People      参数类型：[class java.lang.String]      返回类型void
方法名：getName      所属类：class javaKnowledge.People      参数类型：[]      返回类型class java.lang.String
方法名：setName      所属类：class javaKnowledge.People      参数类型：[class java.lang.String]      返回类型void
方法名：wait      所属类：class java.lang.Object      参数类型：[long, int]      返回类型void
方法名：wait      所属类：class java.lang.Object      参数类型：[long]      返回类型void
方法名：wait      所属类：class java.lang.Object      参数类型：[]      返回类型void
方法名：equals      所属类：class java.lang.Object      参数类型：[class java.lang.Object]      返回类型boolean
方法名：toString      所属类：class java.lang.Object      参数类型：[]      返回类型class java.lang.String
方法名：hashCode      所属类：class java.lang.Object      参数类型：[]      返回类型int
方法名：getClass      所属类：class java.lang.Object      参数类型：[]      返回类型class java.lang.Class
方法名：notify      所属类：class java.lang.Object      参数类型：[]      返回类型void
方法名：notifyAll      所属类：class java.lang.Object      参数类型：[]      返回类型void
hello 
private java.lang.String javaKnowledge.People.eat(java.lang.Integer)
eat 9
~~~

## 配置文件+反射
通常在框架中使用配置文件+反射

代码示例    
在a.properties文件中写入：
~~~
age=19
name=tom
sex=false
money=10000.0
~~~

~~~java
        //使用配置类
        Properties properties=new Properties();

        FileReader in = new FileReader("src/javaKnowledge/a.properties");
        properties.load(in);
        Constructor constructor=clz.getConstructor(int.class,String.class,boolean.class,double.class);
        People people= (People) constructor.newInstance(Integer.parseInt(properties.getProperty("age")),properties.getProperty("name"),Boolean.parseBoolean(properties.getProperty("sex")),Double.parseDouble(properties.getProperty("money")));
        System.out.println(people.getName());
~~~

## 工具类Modifier
Modifier 类提供了 static 方法和常量，对类和成员访问修饰符进行解码。修饰符集被表示为整数，用不同的位位置 (bit position) 表示不同的修饰符。  

### 常用方法
- isAbstract(int mod) 如果整数参数包括 abstract 修饰符，则返回 true，否则返回 false。
- isInterface(int mod) 如果整数参数包括 interface 修饰符，则返回 true，否则返回 false。
- isPublic(int mod) 如果整数参数包括 public 修饰符，则返回 true，否则返回 false。
- 等等

### 一些常量
- public static final int PUBLIC：表示public修饰的int值。
- static int	FINAL ：表示 final 修饰符的 int 值。
- 等等

代码示例：
~~~java
        int a=clz.getModifiers();
        System.out.println(a);
        System.out.println(field.getModifiers());
        field=clz.getDeclaredField("name");
        System.out.println(field.getModifiers());
        field=clz.getDeclaredField("sex");
        System.out.println(field.getModifiers());
        //在People中添加一条语句 public static final int test=1;
        field=clz.getDeclaredField("test");
        System.out.println(field.getModifiers());
        System.out.println(Modifier.classModifiers());
        System.out.println(field.getModifiers());
        System.out.println(Modifier.isPrivate(field.getModifiers()));
~~~
输出：
~~~
1
2
4
25
3103
25
false
~~~

## 缺点
虽然反射很优秀，特别是使用配置文件构造对象，执行方法，用于AOP和IOC等。但是在使用反射时需要注意：
- 效率问题，反射比直接new()会慢一些，1-2倍时间
- 安全问题，反射可以暴力获得本不应该获得的权限，破坏了封闭性和安全性
