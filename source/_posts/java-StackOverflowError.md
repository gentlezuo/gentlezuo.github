---
title: java StackOverflowError与OOM
date: 2019-04-21 11:57:19
tags:
- 错误
- java
category: java
---

StackOverflowError 是一个java中常出现的错误：在jvm运行时的数据区域中有一个java虚拟机栈，当执行java方法时会进行压栈弹栈的操作。在栈中会保存局部变量，操作数栈，方法出口等等。jvm规定了栈的最大深度，当执行时栈的深度大于了规定的深度，就会抛出StackOverflowError错误。当然，本地方法栈也会抛出次异常。   
<!--more-->

典型的例子：
~~~java

public class StackOverFlowDemo {

    public static void Foo(){
        Foo();
    }

    public static void main(String[] args) {
        Foo();
    }
}
~~~

今天我遇见了另外一种情况：当两个对象相互引用，在调用toString方法时会产生这个异常，因为它们会循环调用toString方法。
~~~java
//book和student相互循环引用
public class StackOverFlowDemo {

    static class Student{
        String name;
        Book book;

        public Student(String name) {
            this.name = name;
        }
        //循环调用toString方法
        @Override
        public String toString() {
            return "Student{" +
                    "name='" + name + '\'' +
                    ", book=" + book +
                    '}';
        }
    }

    static class Book {
        String isbn;
        Student student;

        public Book(String isbn, Student student) {
            this.isbn = isbn;
            this.student = student;
        }

        @Override
        public String toString() {
            return "Book{" +
                    "isbn='" + isbn + '\'' +
                    ", student=" + student +
                    '}';
        }
    }

    public static void main(String[] args) {
        Student student=new Student("zhang3");
        Book book=new Book("1111",student);
        student.book=book;
        System.out.println(book.toString());
    }

}
~~~
出现的错误：  
![](java-StackOverflowError/stackoverflow.png)

### toString()

说到toString()方法，在打印一个对象时，会先调用这个对象的toString()方法，例如：   
~~~java
ublic class toStringDemo {

    static class A{
        @Override
        public String toString() {
            System.out.print("I ");
            return "";
        }
    }

    public static void main(String[] args) {
        A a=new A();
        System.out.println("love you."+a);
    }
}
~~~

会输出：   
~~~
I love you.
~~~

## OOM
OutOfMemoryError是堆溢出，堆java运行时的一个数据区域，用来存储对象，当对象太多，超过了堆的最大空间大小就会溢出。


复现：配置jvm参数 `-Xmx100M`,用来设置堆的最大大小，更快看见效果。

下方代码，由于一直往list中添加对象，而且全是活动对象，不会被垃圾回收器回收

```java
public class OOM {
    public static void main(String[] args) {
        List<OOM> list=new ArrayList<>();
        while (true){
            list.add(new OOM());
        }
    }
}
```

![](java-StackOverflowError/OOM.png)
