---
title: HashMap
date: 2019-03-31 19:26:04
tags: 
- java
- JDK源码
- 数据结构
category: java
---
# Map 
map是一种键值映射的数据结构，键不允许重复 

介绍一些java当中实现map的关键的几个方法，包括get，put，resize

## AbstractMap
该类是其他具体map类的父类   

有些子类允许键为null，例如HashMap，有些不允许例如Hashtable，ConcurrentHashMap，因此出现了以下的方法    
代码：
~~~java
public boolean containsKey(Object key) {
        Iterator<Map.Entry<K,V>> i = entrySet().iterator();
        if (key==null) {
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                if (e.getKey()==null)
                    return true;
            }
        } else {
            while (i.hasNext()) {
                Entry<K,V> e = i.next();
                if (key.equals(e.getKey()))
                    return true;
            }
        }
        return false;
    }
~~~
<!--more-->
### HashMap   
HashMap是一个很常用很重要的工具类，看看它是怎么实现的   

它是由数组+链表+红黑树实现的，也就是解决hash的冲突的办法是链地址法

数组的长度永远是2的幂（自有作用）  

允许键值null
#### 基本成员属性
~~~java
//实现了可序列化和克隆
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {
     //默认初始化数组长度为16
     static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
     //数组最大长度
     static final int MAXIMUM_CAPACITY = 1 << 30;
     //负载因子，用来判断是否需要扩容
     static final float DEFAULT_LOAD_FACTOR = 0.75f;
     //大于等于这个数会将链表转化为红黑树提高效率
     static final int TREEIFY_THRESHOLD = 8;
     //小于等于这个数会将红黑树转化为链表
     static final int UNTREEIFY_THRESHOLD = 6;
     //可以树化的最短的数组长度，否则就扩表
     static final int MIN_TREEIFY_CAPACITY = 64;
    //一个Node数组，在使用的时候才初始化，并且长度总是2^m，方便计算角标(len-1)&hash
    transient Node<K,V>[] table;

    transient Set<Map.Entry<K,V>> entrySet;
    //键值对的个数，与数组的长度不一样
    transient int size;
    //更改次数，在迭代时判断map是否被更改
    transient int modCount;
    //下一个数组长度
    int threshold;
    final float loadFactor;

    }
~~~
#### Node类
键值结构在map里是由Map来存储的   
~~~java
static class Node<K,V> implements Map.Entry<K,V> {
        //Node的hash值
        final int hash;
        final K key;
        V value;
        Node<K,V> next;

    //让键值都参与hash运算，混合避免冲突
    public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

}
~~~

#### HashMap具体的方法
key的hash值:高16位与低16为进行亦或运算

~~~java
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }

~~~

求不小于一个数的最小的2^n，例如num=15，返回16；num=16，返回16。

~~~java
static final int tableSizeFor(int cap) {
        //减一是考虑到cap是2^m次方的情况
        int n = cap - 1;
        //使最高位的1与第二位或，最高的位为...00011....
        n |= n >>> 1;
        //...01111....
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        //此时从最高位开始其后全是1，n=2^m-1
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
~~~

构造函数（多种），只介绍一种

~~~java
//可以指定cap或加载因子
public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        this.threshold = tableSizeFor(initialCapacity);
    }
~~~

获取Node的方法：流程是：先判断数组相应位置是否为空，在判断第一个节点，再根据是树还是链表查找

~~~java
final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        //检查table不为空，table长度大于0，tablep对应的key的的位置不为空
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            //先判断第一个Node，通过判断hash值（hash值可能相等）、key是否相同或相等
            if (first.hash == hash && 
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                //若是树，就通过树来查找，反之通过链表查找
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
~~~

put Node的方法,通过尾插法；如果没有初始化先初始化

~~~java
//关于参数
//onlyIfAbsent if true, don't change existing value
//evict if false, the table is in creation mode.
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        //先检查是否需要resize（）
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        //延迟初始化
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            //e是要更改的节点
            Node<K,V> e; K k;
            //总是先判断第一个节点
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                for (int binCount = 0; ; ++binCount) {
                    //尾插法，可能需要转化为树
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                            treeifyBin(tab, hash);
                        break;
                    }
                    //key已存在的情况，记录下该node
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            //如果存在旧的key value，返回oldValue
            if (e != null) { // existing mapping for key
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        //表示已经更改map
        ++modCount;
        //扩容
        if (++size > threshold)
            resize();
        afterNodeInsertion(evict);
        //不存在旧值，只是添加，返回null
        return null;
    }
~~~

扩容:一个table[i]上的节点扩容后只可能在两个位置，一个是原位置，一个是oldCap+i  
先更改长度，在通过遍历原table，添加新table

~~~java
final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        //cap，thr都*=2，包括初始化
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1; // double threshold
        }
        else if (oldThr > 0) // initial capacity was placed in threshold
            newCap = oldThr;
        else {               // zero initial threshold signifies using defaults
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        @SuppressWarnings({"rawtypes","unchecked"})
            Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        //移动Node的具体逻辑
        if (oldTab != null) {
            //遍历数组
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    //只有一个Node，直接移动一个即可
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    //执行链表rehash的逻辑
                    else { 
                        //厉害的一段代码，记住cap是2^n
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            //同一个table[j]下的node只可能在两个位置
                            //实际上是一段位运算根据2^n和2^n-1的特殊性质，把nodes分为两个链表，一段在原位置，一段在原位置+oldCap
                            //判断为cap二进制中的那一位对应的key的hash是否为1，如果为不为1，表示该key的角标不变
                            //反之，该值为1，则该node在新table中的位置需要+oldcap
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
~~~


