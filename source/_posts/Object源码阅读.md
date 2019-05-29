---
title: Object源码阅读
date: 2019-03-29 21:26:17
tags: 
- java
- JDK源码
category: java

---

# Object类
Object是java当中所有类的祖先
## registerNatives()方法
对几个本地方法进行注册(也就是初始化java方法映射到C的方法)。
## public final native Class<?> getClass()
获取class对象

## public native int hashCode()
获取一个对象的hashcode码，若不复写该方法，返回的是该类的内存地址   
如果重写的一个类的equals()方法，必须重写该方法。因为在Set中，根据一个对象的hash码来计算角标((length-1)&hash),如果不复写该方法，会存在两个相等的对象存在一个set中，违反了set的元素不可重复性。
## public boolean equals(Object obj)
判断两个对象是否相等，默认比较两个对象地址
<!--more-->
## protected native Object clone()
实现拷贝  
### 深拷贝与浅拷贝
浅拷贝：被复制对象的所有值属性都含有与原来对象的相同，而所有的对象引用属性仍然指向原来的对象。

深拷贝：在浅拷贝的基础上，所有引用其他对象的变量也进行了clone，并指向被复制过的新对象。

## public String toString()
用于打印对象

## public final native void notify()
唤醒正在等待此对象的单个线程。如果有线程正在等待这个对象，那么会选择唤醒其中一个线程。选择是任意的，并由实施者自行决定。

## public final native void notifyAll()
唤醒所有等待此对象的线程

## public final void wait()
wait()的作用是让当前线程进入等待状态，同时，wait()也会让当前线程释放它所持有的锁。“直到其他线程调用此对象的 notify() 方法或 notifyAll() 方法”，当前线程被唤醒(进入“就绪状态”)
## public final native void wait(long timeout)
在上诉的方法中增加了超时，超时会被唤醒

## protected void finalize()
当对象不再被任何对象引用时，GC会调用该对象的finalize()方法

~~~java

package java.lang;

public class Object {

    private static native void registerNatives();
    static {
        registerNatives();
    }

    public final native Class<?> getClass();

    public native int hashCode();

     public boolean equals(Object obj) {
        return (this == obj);
    }

    protected native Object clone() throws CloneNotSupportedException;

    public String toString() {
        return getClass().getName() + "@" + Integer.toHexString(hashCode());
    }

    public final native void notify();


    public final native void notifyAll();

    public final native void wait(long timeout) throws InterruptedException;

    public final void wait(long timeout, int nanos) throws InterruptedException {
        if (timeout < 0) {
            throw new IllegalArgumentException("timeout value is negative");
        }

        if (nanos < 0 || nanos > 999999) {
            throw new IllegalArgumentException(
                                "nanosecond timeout value out of range");
        }

        if (nanos > 0) {
            timeout++;
        }

        wait(timeout);
    }

     public final void wait() throws InterruptedException {
        wait(0);
    }

    protected void finalize() throws Throwable { }

~~~
