---
title: java并发编程：Lock
date: 2019-06-12 16:50:33
category: java并发编程
---

## Lock的由来

既然有了synchronized,为什么还要有Lock这个类.
<!--more-->
1. synchronized的锁只有在退出了同步代码块或者发生了异常才会退出,而获取不到锁会一直处于阻塞状态,假如迟迟获取不到,会一直阻塞,程序员无法手动释放
2. 当获取了多个锁时，它们必须以相反的顺序释放,试想获取节点 A 的锁，然后再获取节点 B 的锁，然后释放 A 并获取 C，然后释放 B 并获取 D，依此类推。这种情况synchronized就无法满足需求.Lock带来了更多的灵活性.
3. 当一个线程读数据的时候不会阻塞另一个线程,但是写或阻塞写和读。synchronized难以实现，lock可以实现读写锁

## synchronized与Lock的区别

1. synchronized获取锁和释放锁是隐式的,Lock是显式的.
2. synchronized却强制所有锁获取和释放均要出现在一个块结构中，Lock不会
3. synchronized当发生了异常会释放锁，Lock不会，必须手动释放


## Lock

Lock是一个接口。拥有`ReentrantLock, ReentrantReadWriteLock.ReadLock, ReentrantReadWriteLock.WriteLock`的实现。

```java
public interface Lock {

    void lock();

    void lockInterruptibly() throws InterruptedException;

    boolean tryLock();

    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    void unlock();

    Condition newCondition();
}

```

### lock()

lock()方法获取锁，如果获取不到会阻塞直到获取到锁

一般使用模式：
```java
lock.lock();
try{
    //处理任务
}catch(Exception e){
     
}finally{
    //释放锁
    lock.unlock(); 
}
```

### lockInterruptibly()

lockInterruptibly(),如果当前线程未被中断，则开始尝试获取锁，失败会阻塞；如果此线程设置了中断状态，或者被中断，那么会抛出InterruptedException，并且清除当前线程的一中断状态。

一般使用模式：
```java
try{
    //或者在方法上抛出异常
    lock.lockInterruptibly();
}catch(InterruptedException e){
    //处理
}
    try {  
     //.....
    }
    finally {
        lock.unlock();
    }  
```

### tryLock()和tryLock(long time, TimeUnit unit)
tryLock()：立即返回，成功获取到锁返回true，反之返回false

tryLock(long time, TimeUnit unit)：在一定时间内返回，成功true，返回false

一般使用模式:
```java
if(lock.tryLock()) {
     try{
         //处理任务
     }catch(Exception e){
         
     }finally{
         lock.unlock();
     } 
}else {
    //处理获取不到锁的情况
}
```

### unlock()
unlock()，释放锁，一般在finally里使用

### newCondition()
newCondition()，返回该对象的一个Condition。

具体使用需要结合具体的实现

## ReentrantLock

一个**可重入的互斥锁** Lock，它具有与使用 synchronized 方法和语句所访问的隐式监视器锁相同的一些基本行为和语义，但功能更强大。

ReentrantLock 将由最近成功获得锁，并且还没有释放该锁的线程所拥有。当锁没有被另一个线程所拥有时，调用 lock 的线程将成功获取该锁并返回。如果当前线程已经拥有该锁，此方法将立即返回。

此类的构造方法接受一个可选的**公平参数**。当设置为 true 时，在多个线程的争用下，这些锁倾向于将访问权授予**等待时间最长的线程**。否则此锁将无法保证任何特定访问顺序。与采用默认设置（使用不公平锁）相比，使用公平锁的程序在许多线程访问时表现为**很低的总体吞吐量**（即速度很慢，常常极其慢），但是在获得锁和保证锁分配的均衡性时差异较小。不过要注意的是，公平锁不能保证线程调度的公平性。因此，使用公平锁的众多线程中的一员可能获得多倍的成功机会，这种情况发生在其他活动线程没有被处理并且目前并未持有锁时。还要注意的是，**未定时的 tryLock 方法并没有使用公平设置**。因为即使其他线程正在等待，只要该锁是可用的，此方法就可以获得成功。



## 未完



## 