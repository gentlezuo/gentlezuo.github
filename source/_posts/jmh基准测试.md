
jmh用来对java以及其它运行在jvm上的语言进行基准测试，可以达到纳秒，微秒,毫秒，秒级别。
<!--more-->

## 简介
通常简单的测试方法就是在调用一个方法前输出当前时间，调用后再次输出时间，多测试几次，但是这种方法是不准确的。在测试中会收到很多影响，比如jit的优化，cpu缓存，电源管理，超线程技术等等。因此需要有一款专业的基准测试框架。jmh会将这些影响降到最低。

## 使用

### 测试目标
举一个例子。   
比较StringBuilder和StringBuffer的append函数的性能。

预测：StringBuild和StringBuffer的append()函数 ,StringBuffer在该方法上加了synchronize关键字，预测效率会比较低。

### 环境配置
我的电脑配置(有点差)：  
系统：         macOS 10.14.5     
处理器名称：	Intel Core i5  
处理器速度：	2.7 GHz  
处理器数目：	1  
核总数：	2  
L2 缓存（每个核）：	256 KB  
L3 缓存：	3 MB  
内存：	8 GB   
超线程技术：  以启用  
IDE： IDEA 2019.2.3   
VM ： JDK 1.8.0_231, Java HotSpot(TM) 64-Bit Server VM, 25.231-b11

### 测试

1. 添加依赖

```xml
    <properties>
        <jmh.version>1.21</jmh.version>
    </properties>

<dependencies>
    <!-- JMH-->
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-core</artifactId>
        <version>${jmh.version}</version>
    </dependency>
    <dependency>
        <groupId>org.openjdk.jmh</groupId>
        <artifactId>jmh-generator-annprocess</artifactId>
        <version>${jmh.version}</version>
        <scope>provided</scope>
    </dependency>
</dependencies>
```


2. 编写代码：
```java
package com.gentlezuo;

import org.openjdk.jmh.annotations.*;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

import java.util.concurrent.TimeUnit;

@BenchmarkMode(Mode.All)
@Warmup(iterations = 3)
@Measurement(iterations = 3, time = 4, timeUnit = TimeUnit.SECONDS)
@Threads(4)
@Fork(1)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class BuilderVSBuffer {


    @Benchmark
    public void builder() {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < 100000; i++) {
            sb.append("i");
        }
    }

    @Benchmark
    public void buffer() {
        StringBuffer sb = new StringBuffer();
        for (int i = 0; i < 100000; i++) {
            sb.append("i");
        }
    }

    public static void main(String[] args) throws RunnerException {
        Options options = new OptionsBuilder()
                .include(BuilderVSBuffer.class.getSimpleName())
                .output("/Users/gentlezuo/logs/BuilderVSBuffer.log")
                .build();
        new Runner(options).run();
    }
}

```
3. 观察结果

运行结束后会生成该日志文件，文件内容太多，只截取最后几行：
```

Benchmark                                  Mode    Cnt   Score    Error   Units
BuilderVSBuffer.buffer                    thrpt      3   2.757 ±  6.092  ops/ms
BuilderVSBuffer.builder                   thrpt      3   3.170 ±  3.842  ops/ms
BuilderVSBuffer.buffer                     avgt      3   1.396 ±  0.449   ms/op
BuilderVSBuffer.builder                    avgt      3   1.191 ±  0.663   ms/op
BuilderVSBuffer.buffer                   sample  35191   1.357 ±  0.019   ms/op
BuilderVSBuffer.buffer:buffer·p0.00      sample          0.627            ms/op
BuilderVSBuffer.buffer:buffer·p0.50      sample          1.155            ms/op
BuilderVSBuffer.buffer:buffer·p0.90      sample          1.481            ms/op
BuilderVSBuffer.buffer:buffer·p0.95      sample          2.103            ms/op
BuilderVSBuffer.buffer:buffer·p0.99      sample          6.268            ms/op
BuilderVSBuffer.buffer:buffer·p0.999     sample         15.555            ms/op
BuilderVSBuffer.buffer:buffer·p0.9999    sample         29.378            ms/op
BuilderVSBuffer.buffer:buffer·p1.00      sample         31.785            ms/op
BuilderVSBuffer.builder                  sample  38499   1.239 ±  0.016   ms/op
BuilderVSBuffer.builder:builder·p0.00    sample          0.565            ms/op
BuilderVSBuffer.builder:builder·p0.50    sample          1.094            ms/op
BuilderVSBuffer.builder:builder·p0.90    sample          1.307            ms/op
BuilderVSBuffer.builder:builder·p0.95    sample          1.792            ms/op
BuilderVSBuffer.builder:builder·p0.99    sample          4.678            ms/op
BuilderVSBuffer.builder:builder·p0.999   sample         12.927            ms/op
BuilderVSBuffer.builder:builder·p0.9999  sample         33.594            ms/op
BuilderVSBuffer.builder:builder·p1.00    sample         46.727            ms/op
BuilderVSBuffer.buffer                       ss      3   2.699 ± 18.377   ms/op
BuilderVSBuffer.builder                      ss      3   1.605 ±  3.475   ms/op
```

直接看最后的结论：
- 吞吐量：builder是 3.170 ± 3.842 ops/ms；buffer是 2.757 ±  6.092，builder的吞吐量更高
- 平均耗时：builder依旧表现更好
- SampleTime，采样:比如p0.999表示满足99.9%的情况的耗时；可以看见在每一个分位值下builder都表现的更好
- 只调用一次：builder的表现更好

4. 结论：StringBuilder的效率好，因此在不需要线程同步的时候，应该选择StringBuilder。

更多请参考[官方的例子](http://hg.openjdk.java.net/code-tools/jmh/file/tip/jmh-samples/src/main/java/org/openjdk/jmh/samples/)。


## 注解

### @BenchmarkMode
基准测试的模式，有四种值
- Throughput("thrpt", "Throughput, ops/time"), 吞吐量，每一个时间单元执行的次数
- AverageTime("avgt", "Average time, time/op"), 每个操作的的平均时间
- SampleTime("sample", "Sampling time"), 随机取样，会给出满足百分之多少的情况的最坏的时间
- SingleShotTime("ss", "Single shot invocation time"), SingleShotTime 只运行一次。往往同时把 warmup 次数设为0，用于测试冷启动时的性能。
- All("all", "All benchmark modes")，以上所有都测试，最常用

### Warmup
预热，由于JIT的存在，因此预热后的数据更平稳

### Measurement
测试的一些基本的参数
- iterations：测试的次数
- time：每一次测试的时间
- TimeUnit：时间单位
- 每一个op调用方法的个数

### Threads
测试的线程数，可以注解在类上，也可以在方法上

### Fork
会fork出新的几个java虚拟机减少影响，需要设置一系列的参数

### outputTimeUnit
基准测试的时间类型

### @Benchmark
方法级的注解，每一个要测试的方法。

### Param
属性级注解，可以用来指定某项参数的多种情况，在一个函数需要测试多组参数的时候很有用。

### Setup
方法级注解，在测试之前做一些准备工作

### TearDown
方法级注解，在测试之后进行一些结束工作

### State
设置一个类在测试的时候在线程间的共享状态：
- Thread: 线程私有
- Group: 同一个组里面所有线程共享
- benchmark: 所有线程共享

## 总结
jmh是一款优秀的测试框架，但是测试的结果依旧依赖不同的环境，不可只测一次就相信。

参考：   
[https://www.xncoding.com/2018/01/07/java/jmh.html](https://www.xncoding.com/2018/01/07/java/jmh.html)   
[http://openjdk.java.net/projects/code-tools/jmh/](http://openjdk.java.net/projects/code-tools/jmh/)
