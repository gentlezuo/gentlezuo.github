---
title: gc-标记整理算法的两种实现
date: 2019-08-10 23:33:25
tags: 
- java
- gc
- 算法
category: 算法
---

# 标记整理算法的两种实现

传统gc算法有：标记清除，引用计数，复制算法，标记整理。它们各有优点，标记整理的优点就是空间整理功能。来看一下如何实现的吧。

<!--more-->
## 前言
- 活动对象：可以存活的对象
- 非活动对象：可以被回收的对象
- 算法以伪代码表示


## Lisp2 算法

Lisp2算法包括两个阶段
1. 标记
2. 压缩

- 标记阶段：选取gc根对象。从这些对象开始向下遍历其子对象，最终可能会形成一个又向有环图。在此图中的对象就是活动对象，也就是将要压缩的对象。

- 整理阶段：将对象向着一端移动，移动后对象的相对顺序不变，但是对象紧临。

比如：在垃圾回收前：有ABCDEF五个对象，可以看出A和E是非活动对象。其中根引用了B和D，B引用了C，D引用了F

![标记整理-初始状态](/gc-标记整理算法的两种实现/标记整理-初始状态.png)

标记结束后：
![标记整理-标记后](/gc-标记整理算法的两种实现/标记整理-标记后.png)

在垃圾回收后：
![标记整理-整理后](/gc-标记整理算法的两种实现/标记整理-整理后.png)

### 标记阶段算法

选取gc roots，然后以深度优先或者广度优先的方式遍历。

```c
mark_phase(){
    for(r : $roots)
        mark(*r)
}

mark(obj){
    if(obj.mark == FALSE)
        obj.mark = TRUE
        for(child : children(obj))
            mark(*child)
}
```
`$roots`为gc roots，`mark`是对象头的标记位

标记的时候可以在对象头上标记，但是这样与写时复制不兼容。当复制了一个对象，还不需要写的时候开始gc，那么就必须进行对象复制操作。为了解决这个问题可以使用位图标记，不再操作对象头，这样与写时复制兼容，并且标记与清除效率更高。

这种朴素的标记法，在标记阶段必须全局暂停，为了可以并发，可以使用三色标记法。


### 压缩阶段具体实现

lisp2中的对象中的对象：在对象头中有一个forwarding指针，指向将来该对象的位置。

![lisp2中的对象](/gc-标记整理算法的两种实现/lisp2中的对象.png)

步骤：

1. 设置forwarding指针
2. 更新指针
3. 移动对象

```c
compaction_phase(){
    set_forwarding_ptr()
    adjust_ptr()
    move_obj()
}
```
#### 1.设置forwarding指针
在步骤1中，会搜索整个堆，给活动对象设置forwarding的位置。
```c
set_forwarding_ptr(){
    scan = new_address = $heap_start
    while(scan < $heap_end)
        if(scan.mark == TRUE)
            scan.forwarding = new_address
            new_address += scan.size
        scan += scan.size
}
```
`scan`是搜索堆中对象的指针，`new_address`是指向目标地点的指针。当`scan`扫描到活动对象，就会将`new_address`赋值给该对象的`forwarding`，并且移动`new_address`的位置。直到结束。

![lisp2设置forwarding指针](/gc-标记整理算法的两种实现/lisp2设置forwarding指针.png)

这样做的原因是整理前的堆和整理后的堆是同一个空间，不像复制算法中的from和to可以切换。因此有必要设置forwarding的指针。

#### 2.更新指针
第2步需要让每一个对象知道移动后子对象将来的位置。因此需要修改指针.
```c
adjust_ptr(){
    //更改根的指针
    for(r : $roots)
        *r = (*r).forwarding
    
    scan = $heap_start
    while(scan < $heap_end)
        //如果标记过，就将子对象设置为子对象将来的位置（forwarding）
        if(scan.mark == TRUE)
            for(child : children(scan))
                *child = (*child).forwarding
        scan += scan.size
}
```
这里的scan与上面的scan作用相同。

![更改指针](/gc-标记整理算法的两种实现/lisp2更改指针.png)

#### 3.移动对象
第3步，移动对象，过程很简单，每个对象都知道要去的地址，只需要顺序移动便可，然后清除之前的标记。注意要顺序移动，不然会覆盖掉活动对象。
```c
move_obj(){
    scan = $free = $heap_start
    while(scan < $heap_end)
        if(scan.mark == TRUE)
            new_address = scan.forwarding
            copy_data(new_address, scan, scan.size)
            new_address.forwarding = NULL
            new_address.mark = FALSE
            $free += new_address.size
            scan += scan.size
}
```
![lisp2整理后](/gc-标记整理算法的两种实现/lisp2整理后.png)

### 时间复杂度
1. 标记阶段：遍历所有的存活对象，与活动对象数成正比
2. 设置forwarding指针阶段：扫描整个堆
3. 更改子对象指针阶段：扫描整个堆
4. 移动阶段：扫描整个堆


### 优点
- 相比于标记清除与引用计数：没有内存碎片
- 相比于复制算法：有效利用堆

### 缺点
一次遍历活动对象+三次扫描整个堆，吞吐量较小。



## 双指针算法

前提：所有对象大小一致，如果不一致，填充数据。


双指针算法可分为两步：
1. 移动对象
2. 更改子对象的指针

![双指针算法过程](/gc-标记整理算法的两种实现/双指针算法过程.png)

容易理解，当每个对象大小一致时就很容易解决覆盖的问题,可以理解为数组上的数据交换。




### 算法实现

#### 1 移动对象

`free`指针目的是寻找空闲的空间，`live`的目的是寻找活动对象的地址。一个在前，一个在后。

初始状态：

![双指针初始状态](/gc-标记整理算法的两种实现/双指针初始状态.png)

移动：

![双指针交换](/gc-标记整理算法的两种实现/双指针交换.png)

```c
move_obj(){
    $free = $heap_start
    live = $heap_end - OBJ_SIZE
    while(TRUE)
        //跳过活动对象
        while($free.mark == TRUE)
            $free += OBJ_SIZE
        //跳过空闲空间
        while(live.mark == FALSE)
            live -= OBJ_SIZE
        if($free < live)
            //将live指向的对象复制到free的地址，将老对象的forwarding指向新对象的位置，也就是会保留老对象
            copy_data($free, live, OBJ_SIZE)
            live.forwarding = $free
            live.mark = FALSE
        else
            break
}
```
当结束此过程后`free`指针指向的的位置就是分配对象的起始位置。

#### 2 更新指针

![](/gc-标记整理算法的两种实现/双指针初始状态.png)

此时`free`指针指向了空闲区域的开头，free指针指向了free或者前一个对象。

```c
adjust_ptr(){
    for(r : $roots)
        if(*r >= $free)
            *r = (*r).forwarding

    scan = $heap_start
    while(scan < $free)
        scan.mark = FALSE
        for(child : children(scan))
            if(*child >= $free)
                *child = (*child).forwarding
        scan += OBJ_SIZE
}
```

### 时间复杂度

1. 标记：与活动对象成正比
2. 双指针移动阶段：扫描全堆
3. 更新指针阶段：扫描全堆

### 优点
- 相比于lisp2算法，少了一次全堆扫描
- 空间整理

### 缺点
- 限制：对象的大小相同
- 相比于标记清除与复制，吞吐量依旧较小

## 参考
《垃圾回收算法与实现》