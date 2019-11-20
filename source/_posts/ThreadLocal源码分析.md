---
title: ThreadLocal源码分析
date: 2019-06-04 13:15:19
tags: 
- JDK源码
category: java
---

ThreadLocal类可以使每个线程保存一份线程局部变量，也就是当前线程持有一个变量，各个线程之间的这个变量不受影响。一个线程可以有多个ThreadLocal实例。

<!--more-->
## 简介

用法如下：

```java
public class Temp {
    ThreadLocal<Integer> threadLocal=new ThreadLocal<>();
    public static void main(String[] args){
        new Temp().run();
    }


    public void run(){
        for (int i = 0; i <10 ; i++) {
            new Thread(new Runnable(){
                @Override
                public void run() {
                    System.out.println(threadLocal.get());
                    int a=new Random().nextInt(100);
                    threadLocal.set(a);
                    System.out.println(threadLocal+" "+threadLocal.get()+"  "+Thread.currentThread());


                }
            }).start();
        }
    }
}
-------------------------
控制台： 
null
null
null
null
null
null
java.lang.ThreadLocal@31c6e548 74  Thread[Thread-4,5,main]
java.lang.ThreadLocal@31c6e548 39  Thread[Thread-5,5,main]
java.lang.ThreadLocal@31c6e548 66  Thread[Thread-3,5,main]
java.lang.ThreadLocal@31c6e548 10  Thread[Thread-0,5,main]
java.lang.ThreadLocal@31c6e548 30  Thread[Thread-1,5,main]
null
java.lang.ThreadLocal@31c6e548 68  Thread[Thread-2,5,main]
java.lang.ThreadLocal@31c6e548 37  Thread[Thread-6,5,main]
null
null
null
java.lang.ThreadLocal@31c6e548 1  Thread[Thread-9,5,main]
java.lang.ThreadLocal@31c6e548 81  Thread[Thread-8,5,main]
java.lang.ThreadLocal@31c6e548 40  Thread[Thread-7,5,main]
```

从上可以看出，每一个线程都共享同一个ThreadLocal，但是他们又存有一个线程局部变量，这里是一个Integer，每一个局部变量都不影响对方的存在。



## 成员变量

1. `private static AtomicInteger nextHashCode = new AtomicInteger();`这是一个线程安全的Integer，表示一个ThreadLocal的hashcode。从0开始原子的增加。

2. `private final int threadLocalHashCode = nextHashCode();`表示当前的ThreadLocal的hashcode，显然每一个ThreadLocal的hashcode都不相同。

3. `private static final int HASH_INCREMENT = 0x61c88647;`表示每一次ThreadLocal都自增HASH_INCREMENT大小。

threadLocalHashCode是一个很重要的变量，ThreadLocal保存的值实际上是在一个Map中，而且key就是根据hashcode来计算这个值应该在数组的哪个位置。

在了解这个类的工作机制之前，先了解一些其他要使用的类。
## Entry类
Entry类是一个数据结构类，表示一个map的节点。

它继承类弱引用，当此时的线程消失，它的key就会被垃圾回收器回收。也就是会造成内存泄露，不过在ThreadLocal中会经常检查key为null的值，然后清除掉，所以避免了内存泄露。

它的key就是当前的ThreadLocal，value就是保存的值。

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
            Object value;
            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
```

## ThreadLocalMap

ThreadLocalMap是一个真正实现存储逻辑的一个类，上面的Entry是这个类的静态内部类。

Map的数据结构是一个Entry数组实现的，而处理哈希冲突是采用线性探查法。  


### 注意

ThreadLocalMap使用的解决哈希冲突的方法是线性探测法，并且删除一个节点的时候它真的删除了这个节点（有些方法是标记该位置已经被删除，而不是删除）。

试想一种情况，一个ThreadLocal的实例a，它在set的的时候，hashcode&(len-1)得到的位置应该10,但是从10到15都有节点，那么它被放在了16的位置，假如此时10位置（或者11,12,13等等）的位置的节点被remove掉了，下一次有重新设置a的保存的值，那么此时可能就会将它放在10位置，就存在的两个key相同节点。

ThreadLocalMap并没有这个bug，那么它的解决办法是：在remove时，它会清除该节点之后的无用节点（避免内存泄露），以及将一些节点"往前挪"（保证hash的正确性），也可能不往前挪，保证了一个优先级：一个节点本来应该放在一个数组的位置空闲了，那么这个节节点就会改变现在的位置来占有这个位置。

实际上清除无用节点以及“挪动节点”很频繁，在getEntryAfterMiss，remove，replaceStaleEntry，cleanSomeSlots，expungeStaleEntries都调用了这个方法。那么它为什么不使用链地址法来解决哈希冲突呢，我认为因该是在一个线程中，ThreadLocal不会太多，所以没必要使用链地址法，经常遍历与移动也耗费不了太多时间吧。

因此它具有以下成员变量：  
### 成员变量

1. `private static final int INITIAL_CAPACITY = 16;`初始容量，必须是2的幂。因为2的幂-1用来与&hanshcode可以很方便的寻找该key在数组的坐标。
2. `private Entry[] table;`哈希数组
3. `private int size = 0;`map的节点数
4. `private int threshold;`当节点的数量达到了这个数，就表示该扩容了。


### 成员函数

为了实现它的功能：put，remove，set，rehash等，它具有以下方法。


#### 构造函数
传入了当前的key和value，然后new了一个长为INITIAL_CAPACITY的数组，在获取到该节点的坐标(hash&(cap-1))，然后将new一个节点放进数组
```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
```


#### getEntry

获取一个节点，先找到它应该在的位置，如果不再，就往后探测

```java
private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            if (e != null && e.get() == key)
                return e;
            else
            //由于是线性探测，所以需要向后探测
                return getEntryAfterMiss(key, i, e);
        }

```

getEntryAfterMiss：用该方法获取，依次往后遍历，如果找到那么就返回该节点，如果遇到了null，那么就清除一些节点（清除节点是额外的事，相当于顺带做一些好事）。
```java

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
            Entry[] tab = table;
            int len = tab.length;

            while (e != null) {
                ThreadLocal<?> k = e.get();
                if (k == key)
                    return e;
                    //表示这个节点‘尸位素餐‘,清理掉
                if (k == null)
                    expungeStaleEntry(i);
                else
                    i = nextIndex(i, len);
                e = tab[i];
            }
            return null;
        }

```

#### set(ThreadLocal<?> key, Object value)

set方法用来设置新值或更改旧值

```java
private void set(ThreadLocal<?> key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            //根据线性探测，寻找key应该在的位置
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                ThreadLocal<?> k = e.get();
                //如果找到了已经存在key，那么赋值退出循环
                if (k == key) {
                    e.value = value;
                    return;
                }
                //如果key为空，那么在这个位置放入当前节点
                if (k == null) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }
            //遇到了null，那么在null位置放入新的节点
            tab[i] = new Entry(key, value);
            int sz = ++size;
            //既没有清除的节点，也到了该扩容的地步了，就扩容
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
```


#### void remove(ThreadLocal<?> key)

删除一个节点

```java
private void remove(ThreadLocal<?> key) {
            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);
            for (Entry e = tab[i];
                 e != null;
                 e = tab[i = nextIndex(i, len)]) {
                if (e.get() == key) {
                    e.clear();
                    //清除无用节点，并且把这个节点之后的节点放在改在的位置，比如有的节点可能在在此节点清除后就应该这个节点的位置。
                    expungeStaleEntry(i);
                    return;
                }
            }
        }
```

#### private int expungeStaleEntry(int staleSlot)

很重要的一个方法，用来清除从staleSlot位置之后的应该清除的节点

```java
private int expungeStaleEntry(int staleSlot) {
            Entry[] tab = table;
            int len = tab.length;
            //首先先清除此位置的节点
            tab[staleSlot].value = null;
            tab[staleSlot] = null;
            size--;

            Entry e;
            int i;
            for (i = nextIndex(staleSlot, len);
                 (e = tab[i]) != null;
                 i = nextIndex(i, len)) {
                ThreadLocal<?> k = e.get();
                //k为空，表示这个位置以前的ThreadLocal已经不存在了，可以清除掉
                if (k == null) {
                    e.value = null;
                    tab[i] = null;
                    size--;
                } else {
                    //当这个位置的ThreadLocal还存在时，那么重新计算它的位置，这是解决冲突的重要方法
                    int h = k.threadLocalHashCode & (len - 1);
                    if (h != i) {
                        tab[i] = null;
                        while (tab[h] != null)
                            h = nextIndex(h, len);
                        tab[h] = e;
                    }
                }
            }
            //返回一个节点为null的索引
            return i;
        }
```

#### private boolean cleanSomeSlots(int i, int n)

该方法尝试删除一些节点，也可能不会删除，如果删除了，那么返回true

```java
private boolean cleanSomeSlots(int i, int n) {
            boolean removed = false;
            Entry[] tab = table;
            int len = tab.length;
            do {
                i = nextIndex(i, len);
                Entry e = tab[i];
                if (e != null && e.get() == null) {
                    //每次遇到可以清除的节点，n就变为len，表示重新计数
                    n = len;
                    removed = true;
                    i = expungeStaleEntry(i);
                }
                //如果连续logn次，没有遇到可以清除的节点，就直接返回
            } while ( (n >>>= 1) != 0);
            return removed;
        }
```

#### resize
就是直接简单的扩容，然后重新计算每个节点应该在的位置即可。

## 成员函数

接下来是ThreadLocal的一些常用的函数

### get()
获取当前ThreadLocal所保存的值。

```java
public T get() {
        Thread t = Thread.currentThread();
        //获取当前线程所的ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        //从map中获取这个值
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        return setInitialValue();
    }
```

### set()
设置当前ThreadLocal对应的值。依旧是先获取ThreadLocalMap，然后增加或者设置值而已。

```java
public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }
```


### 其他

还有一些辅助的函数，比如创造一个map，remove节点等等，都很简答。


为什么不将ThreadLocalMap放入Thread中呢？

ThreadLocalMap不是必需品，定义在Thread中增加了成本，定义在ThreadLocal中按需创建即可。

## 总结

每一个Thread实例，都持有一个ThreadLocalMap实例（可能为空），当我们使用ThreadLocal保存一些数据时，实际上是向这个Map中写入数据。key是该ThreadLocal，值就是该ThreadLocal想要保存的值。


