---
title: 计算机数值表示&Integer
date: 2019-03-30 19:42:14
tags: 
- java
- JDK源码
- 计算机组成原理
category: java
---

在分析Integer类之前，先复习一下在计算机中数值的表示
## 原码 反码 补码  
<!--more-->
借鉴了[你真的了解补码吗](https://www.jianshu.com/p/3004e5999be4)

有符号数在计算机中存储，用数的最高位存放符号, 正数为0, 负数为1
- 原码：原码就是**符号位**加上真值的**绝对值**，即用第一个二进制位表示符号（正数该位为0，负数该位为1），其余位表示值。
- 反码：正数的反码与其原码相同；负数的反码是对其原码逐位取反，但符号位除外。
- 补码：正数的补码就是其本身；负数的补码是在其反码的基础上+1
例如：
- 2 :  00000010（原码），00000010（反码），00000010（补码）
- -2:  10000010（原码），11111101（反码），11111110（补码）

由补码求原码：
- 如果补码的符号位为“0”，表示是一个正数，所以补码就是该数的原码。 
- 如果补码的符号位为“1”，表示是一个负数，求原码的操作可以是：符号位为1，其余各位取反，然后再整个数加1。

### 为什么在计算机中使用补码
对于计算机而言，加减乘数是最基础的运算, 要设计的尽量简单。我们知道，根据运算法则，减去一个正数等于加上一个负数，所以计算机内部可以只有加法而没有减法，将符号位也参与运算，将减法用加法替代。

补码的优点：　　
- 可将减法变为加法，省去减法器；
- 修复了原码中0的符号(有 [+0] [-0] 之分)以及存在两个编码(0000 0000 和 1000 0000)的问题，而且还能够多表示一个最低数。

补码的由来
负数的补码时相应的正数取反+1   
假如一个数是`10000110`，对应的正数是`00000110`,该数的补码是：`11111010`   
可以看做是0-num
~~~
100000000
-00000110
-----------
 11111010
~~~
1 0000 0000 = 1111 1111 + 1   
因此可以将以上分为两个步骤：按位取反，加1
~~~
 11111111
-00000110
-------------
 11111001  +1
 11111010
~~~

### 将减法转换为加法的正确性证明
已知：X、Y都为正整数，Z = X - Y = X + (-Y)
证明：Z = X的补码 + (-Y的补码)
~~~
X的补码 = X; // 正数的补码为其自身
-Y的补码 = (1111 1111 - Y) + 1; // 负数的补码为除符号位的其它位取反加1
Z = X的补码 + (-Y的补码) 
  = X +  (1111 1111 - Y) + 1 
  = X - Y + 1 0000 0000
  = X - Y + 0000 0000
  = X - Y
~~~

## Integer
佩服Lee Boynton，Arthur van Hoff，Josh Bloch，Joseph D. Darcy   

计算机中二进制运算比普通的加减乘除快很多，因此代码中很多使用了位运算，难以阅读，挑选了几个具有代表性的函数分析   

2^n-1是一个神奇的数，在各种场景中都有它的身影，比如hashmap取索引，比如求2^n的mod  

Integer具有缓存机制，当然不止是Integer,包括Long，Byte等


~~~java
public final class Integer extends Number implements Comparable<Integer> {

    @Native public static final int   MIN_VALUE = 0x80000000;

    @Native public static final int   MAX_VALUE = 0x7fffffff;

    public static final Class<Integer>  TYPE = (Class<Integer>) Class.getPrimitiveClass("int");

    private final int value;

~~~
`MIN_VALUE`表示最小值-2^31，`MAX_VALUE`表示最大值2^32-1

转换进制
~~~java

public static String toString(int i, int radix) {
        if (radix < Character.MIN_RADIX || radix > Character.MAX_RADIX)
            radix = 10;

        /* Use the faster version */
        if (radix == 10) {
            return toString(i);
        }
        //int型的数最长32位，加上符号33位
        char buf[] = new char[33];
        boolean negative = (i < 0);
        int charPos = 32;
        //将所有的数转换为负数，主要是为了统一接下来的计算
        //如果不这样，负数%radix为负数，正数%radix为正数，需要分两种情况判断
        //为什么不转换为正数，因为最小的负数-2^31转换为正数会溢出
        if (!negative) {
            i = -i;
        }
        //开始转换
        while (i <= -radix) {
            buf[charPos--] = digits[-(i % radix)];
            i = i / radix;
        }
        buf[charPos] = digits[-i];

        if (negative) {
            buf[--charPos] = '-';
        }

        return new String(buf, charPos, (33 - charPos));
    }
~~~

转换为无符号进制数,转换为2^shift进制
~~~java
private static String toUnsignedString0(int val, int shift) {
        // assert shift > 0 && shift <=5 : "Illegal shift value";
        //得到除了前导0其他的位数
        int mag = Integer.SIZE - Integer.numberOfLeadingZeros(val);
        //得到最大的长度
        int chars = Math.max(((mag + (shift - 1)) / shift), 1);
        char[] buf = new char[chars];

        formatUnsignedInt(val, shift, buf, 0, chars);

        // Use special constructor which takes over "buf".
        return new String(buf, true);
    }

    static int formatUnsignedInt(int val, int shift, char[] buf, int offset, int len) {
        int charPos = len;
        //要转换的进制
        int radix = 1 << shift;
        //掩码，永远时2^n-1
        int mask = radix - 1;
        //每次得到后shift位，val & mask相当于mod运算！，循环得到结果
        do {
            buf[offset + --charPos] = Integer.digits[val & mask];
            val >>>= shift;
        } while (val != 0 && charPos > 0);

        return charPos;
    }
~~~

将一个整数放入字符数值中
~~~java
static void getChars(int i, int index, char[] buf) {
        int q, r;
        int charPos = index;
        char sign = 0;

        if (i < 0) {
            sign = '-';
            i = -i;
        }

        // Generate two digits per iteration
        while (i >= 65536) {
            q = i / 100;
        // really: r = i - (q * 100);
            r = i - ((q << 6) + (q << 5) + (q << 2));
            i = q;
            //实际上r是一个100以内的数，一次性可以确定两位，根据DigitOnes和DigitTens两个数组
            buf [--charPos] = DigitOnes[r];
            buf [--charPos] = DigitTens[r];
        }

        // Fall thru to fast mode for smaller numbers
        // assert(i <= 65536, i);
        for (;;) {
            q = (i * 52429) >>> (16+3);
            r = i - ((q << 3) + (q << 1));  // r = i-(q*10) ...
            buf [--charPos] = digits [r];
            i = q;
            if (i == 0) break;
        }
        if (sign != 0) {
            buf [--charPos] = sign;
        }
    }
~~~
求前导0的数量
~~~java
public static int numberOfLeadingZeros(int i) {
        // HD, Figure 5-6
        if (i == 0)
            return 32;
        int n = 1;
        //先判断高16位，再判断8位...类似二分法
        if (i >>> 16 == 0) { n += 16; i <<= 16; }
        if (i >>> 24 == 0) { n +=  8; i <<=  8; }
        if (i >>> 28 == 0) { n +=  4; i <<=  4; }
        if (i >>> 30 == 0) { n +=  2; i <<=  2; }
        //判断最高位，若是0，则就是31位，反之30位前导0
        n -= i >>> 31;
        return n;
    }
~~~
求低位第一位为1的值
~~~java
public static int lowestOneBit(int i) {
        // HD, Section 2-1
        return i & -i;
    }
~~~
求高为第一位为1的权值
~~~java
public static int highestOneBit(int i) {
        // HD, Figure 3-1
        i |= (i >>  1);
        i |= (i >>  2);
        i |= (i >>  4);
        i |= (i >>  8);
        i |= (i >> 16);
        //此时i为11111111...
        return i - (i >>> 1);
    }
~~~

缓存机制：默认缓存范围是-128～127，因此，两个数值在此区间的Integer使用`==`判断返回的是`true`，而其他相等范围的Integer则返回false。例如：
~~~java
        Integer c=55;
        Integer d=55;
        System.out.println(c==d);(true)
        Integer e=999;
        Integer f=999;
        System.out.println(e==f);(false)
~~~
源码：
~~~java
//私有的静态内部类
private static class IntegerCache {
        static final int low = -128;
        static final int high;
        //使用一个数组存储缓存
        static final Integer cache[];

        //静态代码块，在类加载时期就已经执行了
        static {
            
            int h = 127;
            //用于指定缓冲区的大小
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h;

            cache = new Integer[(high - low) + 1];
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
//如果数值在缓存范围，获取的就是缓存的对象，而非重新new
public static Integer valueOf(int i) {
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        return new Integer(i);
    }

~~~

## Short，Long,Byte
这些类与Integer相似，只是范围不同，它们也有缓存机制，范围都是-128～127，并且不可更改

## Double，Float
注意：是允许1/0,-1/0,0/0的,其他类型都不允许

例如：
~~~java
        Double a=-1.0/0.0+100;
        Double b=1.0/0.0;
        Double c=0.0/0.0;

        System.out.println(a);  ( -Infinity )
        System.out.println(b);  ( Infinity )
        System.out.println(c);  ( NaN )

~~~