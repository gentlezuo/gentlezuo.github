---
title: TimSort
date: 2019-04-02 13:15:19
tags: 
- JDK源码
- 算法
- 排序
category: 算法
---
Timsort是一种经过优化的，使用了二分排序，归并排序的高效排序算法  

时间复杂度最优O(n),平均&最差O(NlogN)；主要是利用到了“已排好序”的run块   

在java，Python等语言的内置排序算法中都使用了它   

分析一下java中的TimSort:    
如果数组长度小于某个值，二分插入；否则  
将原数组分为n个有序的run块，将每个run块记录下来；  
run进行合并  
<!--more-->
## 暴露的排序接口  
宏观上的排序逻辑
~~~java
static <T> void sort(T[] a, int lo, int hi, Comparator<? super T> c,
                         T[] work, int workBase, int workLen) {
                             

        //判断各种条件是否合法
        assert c != null && a != null && lo >= 0 && lo <= hi && hi <= a.length;
        //获取需要排序的剩余的长度
        int nRemaining  = hi - lo;
        if (nRemaining < 2)
            return;  // Arrays of size 0 and 1 are always sorted

        //长度太短，直接二分插入排序
        if (nRemaining < MIN_MERGE) {
            //获取有序的run块
            int initRunLen = countRunAndMakeAscending(a, lo, hi, c);
            binarySort(a, lo, hi, lo + initRunLen, c);
            return;
        }

        
        TimSort<T> ts = new TimSort<>(a, c, work, workBase, workLen);
        //获取最小的run的长度，范围[16,32]
        int minRun = minRunLength(nRemaining);
        do {
            //run的长度
            int runLen = countRunAndMakeAscending(a, lo, hi, c);

            //run的长度太小，在[lo,force）之间二分插入排序，将[lo,force）变为有序的run块
            if (runLen < minRun) {
                int force = nRemaining <= minRun ? nRemaining : minRun;
                binarySort(a, lo, lo + force, lo + runLen, c);
                runLen = force;
            }

            //将run块的起始地点和长度，压入栈中，最后会归并
            ts.pushRun(lo, runLen);
            //可能进行归并
            ts.mergeCollapse();

            // Advance to find next run
            lo += runLen;
            nRemaining -= runLen;
        } while (nRemaining != 0);

        // Merge all remaining runs to complete sort
        assert lo == hi;
        //合并所有的run
        ts.mergeForceCollapse();
        assert ts.stackSize == 1;
    }
~~~




## 二分插入排序
在数据较少的时候二分插入排序即可

将数组a的lo到hi之间排序，lo到srart之间有序，将start到hi之间的数有序的插入到lo到start之间

~~~java
private static <T> void binarySort(T[] a, int lo, int hi, int start,
                                       Comparator<? super T> c) {
        assert lo <= start && start <= hi;
        if (start == lo)
            start++;
        //将start到hi之间的数插入到前半部分中
        for ( ; start < hi; start++) {
            //pivote是将要插入的数
            T pivot = a[start];

            int left = lo;
            int right = start;
            assert left <= right;
            //开始二分，查找到pivote该在的位置
            while (left < right) {
                int mid = (left + right) >>> 1;
                if (c.compare(pivot, a[mid]) < 0)
                    right = mid;
                else
                    left = mid + 1;
            }
            assert left == right;

            //复制移动数组，并将pivote写到该在的位置
            int n = start - left;  // The number of elements to move
            // Switch is just an optimization for arraycopy in default case
            switch (n) {
                case 2:  a[left + 2] = a[left + 1];
                case 1:  a[left + 1] = a[left];
                         break;
                default: System.arraycopy(a, left, a, left + 1, n);
            }
            a[left] = pivot;
        }
    }
~~~

## 计算run块的长度（连续有序块）
获取连续有序代码块的长度，如果是逆序，就翻转该序列使其正序

~~~java
private static <T> int countRunAndMakeAscending(T[] a, int lo, int hi,
                                                    Comparator<? super T> c) {
        assert lo < hi;
        int runHi = lo + 1;
        if (runHi == hi)
            return 1;

        // Find end of run, and reverse range if descending
        if (c.compare(a[runHi++], a[lo]) < 0) { // Descending
            while (runHi < hi && c.compare(a[runHi], a[runHi - 1]) < 0)
                runHi++;
            reverseRange(a, lo, runHi);
        } else {                              // Ascending
            while (runHi < hi && c.compare(a[runHi], a[runHi - 1]) >= 0)
                runHi++;
        }

        return runHi - lo;
    }
~~~

## minRunLength  
计算一个run的最小长度，在[16,32]之间，估计是为了效率

~~~java
private static int minRunLength(int n) {
        assert n >= 0;
        int r = 0;      // Becomes 1 if any 1 bits are shifted off
        while (n >= MIN_MERGE) {
            r |= (n & 1);
            n >>= 1;
        }
        return n + r;
    }
~~~

## 记录run的位置和长度

~~~java
private void pushRun(int runBase, int runLen) {
        this.runBase[stackSize] = runBase;
        this.runLen[stackSize] = runLen;
        stackSize++;
    }
~~~

## 在添加run时可能归并  
至于为什么需要归并，估计是为了效率
~~~java
private void mergeCollapse() {
        while (stackSize > 1) {
            int n = stackSize - 2;
            if (n > 0 && runLen[n-1] <= runLen[n] + runLen[n+1]) {
                if (runLen[n - 1] < runLen[n + 1])
                    n--;
                mergeAt(n);
            } else if (runLen[n] <= runLen[n + 1]) {
                mergeAt(n);
            } else {
                break; // Invariant is established
            }
        }
    }
~~~

## 最后强制合并

~~~java
private void mergeForceCollapse() {
        while (stackSize > 1) {
            int n = stackSize - 2;
            if (n > 0 && runLen[n - 1] < runLen[n + 1])
                n--;
            mergeAt(n);
        }
    }
~~~

## 优化后的归并

~~~java
private void mergeAt(int i) {
        assert stackSize >= 2;
        assert i >= 0;
        assert i == stackSize - 2 || i == stackSize - 3;

        int base1 = runBase[i];
        int len1 = runLen[i];
        int base2 = runBase[i + 1];
        int len2 = runLen[i + 1];
        assert len1 > 0 && len2 > 0;
        assert base1 + len1 == base2;

        //更改之后的数据
        runLen[i] = len1 + len2;
        if (i == stackSize - 3) {
            runBase[i + 1] = runBase[i + 2];
            runLen[i + 1] = runLen[i + 2];
        }
        stackSize--;

        //找到run2的第一个元素在run1中的位置,可以减少一段len1
        int k = gallopRight(a[base2], a, base1, len1, 0, c);
        assert k >= 0;
        //base和len都会减少，因为run2是递增的
        base1 += k;
        len1 -= k;
        if (len1 == 0)
            return;

        //找到run1的最后一个元素在run2中的位置。可以减少一段len2
        len2 = gallopLeft(a[base1 + len1 - 1], a, base2, len2, len2 - 1, c);
        assert len2 >= 0;
        if (len2 == 0)
            return;
        
        //根据最短的len，决定怎么合并，也是一种优化
        if (len1 <= len2)
            mergeLo(base1, len1, base2, len2);
        else
            mergeHi(base1, len1, base2, len2);
    }
~~~


