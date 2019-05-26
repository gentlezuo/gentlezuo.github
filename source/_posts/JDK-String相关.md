---
title: JDK-String相关
date: 2019-03-30 15:39:00
tags: 
- java
- JDK源码
category: java
---
# StringBuilder StringBuffer AbstractStringBuilder

StringBuilder StringBuffer是常用的扩展了String功能的类，它们继承了AbstractStringBuilder。
一般而言在单线程中使用StringBuilder，多线程中使用StringBuffer（线程安全）。在单线程中，由于synchronized锁持有偏向锁，所以效率相差不大。   

StringBuilder在更改字符串时由于缓冲区的原因可以提高效率，而且该类也提供了更丰富的功能。
## 源码分析

### AbstractStringBuilder
~~~java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    //底层使用char数组，具有缓冲功能
    char[] value;
    //真实的字符串的长度
    int count;
    //数组最大长度
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
    //取max(2n+2，minCapacity)，当然不能大于最大长度
    private int newCapacity(int minCapacity) {
        
        int newCapacity = (value.length << 1) + 2;
        if (newCapacity - minCapacity < 0) {
            newCapacity = minCapacity;
        }
        return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
            ? hugeCapacity(minCapacity)
            : newCapacity; 
    }
~~~
<!--more-->
#### 关于append，delete，replace等方法 
大量调用了`System.arraycopy()`,`System.arraycopy()`为 JVM 内部固有方法，它通过手工编写汇编或其他优化方法来进行 Java 数组拷贝，这种方式比起直接在 Java 上进行 for 循环或 clone 是更加高效的。数组越大体现地越明显。
~~~java
    //如果对象为空，会添加"null"，
    public AbstractStringBuilder append(String str) {
        if (str == null)
            return appendNull();
        int len = str.length();
        ensureCapacityInternal(count + len);
        str.getChars(0, len, value, count);
        count += len;
        return this;
    }
}
~~~
### StringBuilder
StringBuilder继承AbstractStringBuilder，线程不安全   
缓冲区默认大小为16，正是这种缓冲区有效的减少了追加字符串带来的内存重复申请的次数，提高了效率
~~~java
public StringBuilder() {
        super(16);
    }
~~~

### StringBuffer
线程安全的类，在每个方法上都使用了`synchronized`修饰   
例如：
~~~java
public synchronized StringBuffer reverse() {
        toStringCache = null;
        super.reverse();
        return this;
    }
~~~
