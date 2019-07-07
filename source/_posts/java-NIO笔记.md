---
title: java NIO笔记
date: 2019-05-01 12:40:06
tags:
- java
- NIO
category: java
---

# NIO笔记

在以前传统的java IO是阻塞的，面向字节的。而NIO是非阻塞的，面向缓冲区的。
<!--more-->
在NIO中有三个组件：Buffer，Channel，Selector。

## Buffer
Buffer是缓冲区，JDK提供了抽象的Buffer类，以及基本数据类型（除了boolean）的Buffer，例如IntBuffer。   
缓冲区是存取数据的一个容器。   
指针可以在缓冲区上前进后退，可以读缓冲区，写缓冲区，标记复位。正因为可以一次性写入读取一串数据，因此比传统IO更快。    

### Buffer的属性
- position：下一个可以读取的位置
- limit：读取的位置的界限
- capacity：缓冲区的容量
- mark：标记，可以用来重置position
恒等式：mark <= position <= limit <=capacity　　　

### Buffer的主要方法 
- allocate(int capacity),分配空间，实际上是返回一个子类HeapByteBuffer的对象
- put(),向缓冲区添加数据(Buffer类没有这个方法)
- get()，从获取数据
- flip(),倒置，limit变为position，position变为0，mark变为-1
- clear()，清除缓冲区

以ByteBuffer类为例

~~~java
//创建缓冲对象
ByteBuffer buffer= ByteBuffer.allocate(1024);
~~~


~~~java

public ByteBuffer put(byte x) {
    // 设置该位置的值，并且更新position
        hb[ix(nextPutIndex())] = x;
        return this;

    }

public byte get() {
        return hb[ix(nextGetIndex())];
    }
//limit为position，position置为0
public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }
//数据还在，只是被遗忘了
public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }

~~~
### 直接缓存和非直接缓存区
[https://blog.csdn.net/qiuwenjie123/article/details/79795699](https://blog.csdn.net/qiuwenjie123/article/details/79795699)    
正常情况下，JVM创建一个缓冲区的时候，实际上做了如下几件事：
1.JVM确保Heap区域内的空间足够，如果不够则使用触发GC在内的方法获得空间;
2.获得空间之后会找一组堆内的连续地址分配数组, 这里需要注意的是，在物理内存上，这些字节是不一定连续的;
3.对于不涉及到IO的操作，这样的处理没有任何问题，但是当进行IO操作的时候就会出现一点性能问题.

所有的IO操作都需要操作系统进入内核态才行，而JVM进程属于用户态进程, 当JVM需要把一个缓冲区写到某个Channel或Socket的时候，需要切换到内核态.

而内核态由于并不知道JVM里面这个缓冲区存储在物理内存的什么地址，并且这些物理地址并不一定是连续的(或者说不一定是IO操作需要的块结构)，所以在切换之前JVM需要把缓冲区复制到物理内存一块连续的内存上, 然后由内核去读取这块物理内存，整合成连续的、分块的内存.

也就是说如果我们这个时候用的是非直接缓存的话，我们还要进行“复制”这么一个操作，而当我们申请了一个直接缓存的话，因为他本是就是一大块连续地址，我们就可以直接在它上面进行IO操作，省去了“复制”这个步骤

当然缺点也是有的，他的分配和释放都比较昂贵，相对于非直接缓存而言；因为直接缓冲区都是JVM提前申请好的。

## Channel
通道连接io流，使得数据可以在通道中流动。

通道类型分为两种，一种是面向文件的，另一种是面向网络的：
- FileChannel（面向文件）
- DatagramChannel
- SocketChannel
- ServerSocketChannel

### 使用

#### 文件Channel
获取channel，使用Buffer进行传输数据，关闭资源。

比如实现文件复制
~~~java
public static void copy2() throws IOException {

        //获取io
        FileInputStream in = new FileInputStream(new File("src/javaKnowledge/niostudy/1.jpeg"));
        FileOutputStream out = new FileOutputStream(new File("src/javaKnowledge/niostudy/2.jpeg"));

        //根据io获取channel
        FileChannel inChannel =in.getChannel();
        FileChannel outChannel =out.getChannel();

        //直接缓冲区
        //inChannel.transferTo(0,inChannel.size(),outChannel);
        outChannel.transferFrom(inChannel,0,inChannel.size());

        //关闭资源
        in.close();
        out.close();
        inChannel.close();
        outChannel.close();


    }
~~~

可以使用内存映射  
~~~java
public static void copy4() throws IOException {
    //获取channel的另一种方式
        FileChannel inChannel=FileChannel.open(Paths.get("src/javaKnowledge/niostudy/1.jpeg"), StandardOpenOption.READ);
        FileChannel outChannel=FileChannel.open(Paths.get("src/javaKnowledge/niostudy/2.jpeg"),StandardOpenOption.CREATE,StandardOpenOption.READ,StandardOpenOption.WRITE);

        //内存映射文件
        MappedByteBuffer inMappedByteBuffer=inChannel.map(FileChannel.MapMode.READ_ONLY,0,inChannel.size());
        MappedByteBuffer outMappedByteBuffer=outChannel.map(FileChannel.MapMode.READ_WRITE,0,inChannel.size());

        //读到bytes中
        byte[] bytes=new byte[inMappedByteBuffer.limit()];
        inMappedByteBuffer.get(bytes);
        outMappedByteBuffer.put(bytes);
        inChannel.close();
        outChannel.close();
    }
~~~

#### 网络Channel

- DatagramChannel：UDP网络套接字
- SocketChannel：TCP网络套接字
- ServerSocketChannel：TCP服务端套接字通道

这些Channel可以实现非阻塞，阻塞IO在一个读或写请求如果没有读到或者写入就会一直阻塞，那么服务端要想实现并发就必须每一个连接就建立一个线程，但是这样很浪费资源。所以就有了非阻塞IO模式，非阻塞就是在一个请求当中，无论是否有结果都返回，实际上网络Channel一般配合Selector使用。

通道的使用
~~~java

        //一般客户端这么使用
        //打开套接字通道
        SocketChannel socketChannel = SocketChannel.open();
        //设置为非阻塞
        socketChannel.configureBlocking(false);
        //连接某个主机的某个端口
        socketChannel.connect(new InetSocketAddress("localhost", 10000));
        //如果已连接了此通道，则不阻塞此方法并且立即返回 true。如果此通道处于非阻塞模式，那么当连接过程尚未完成时，此方法将返回 false。
        //如果此通道处于阻塞模式，则在连接完成或失败之前将阻塞此方法，并且总是返回 true 或抛出描述该失败的、经过检查的异常。

        //死循环，与阻塞相似
        while(!socketChannel.finishConnect()){
        ...
       }

        //关闭
        socketChannel.close();

~~~

~~~java

        ServerSocketChannel serverSocketChannel=ServerSocketChannel.open();
        serverSocketChannel.configureBlocking(false);
        //绑定端口号，监听
        serverSocketChannel.socket().bind(new InetSocketAddress(10000));
        
        ServerSocket socket=serverSocketChannel.socket();
        socket.accept();
~~~


### Selector

选择器实现了复用，很多通道都注册在这个选择器上，当选择器发现通道有了请求就可以对每一个通道进行处理，服务端可以对每一个请求建立一个线程，比BIO要高效。

![Selector](/java-NIO笔记/selector.jpg)

~~~java
//创建选择器
Selector selector = Selector.open();
~~~

选择键：SelectionKey（表示是哪一个请求）
- OP_READ = 1 << 0;
- OP_WRITE = 1 << 2;
- OP_CONNECT = 1 << 3;
- OP_ACCEPT = 1 << 4;



事件集合：
interestOps 和 readyOps，interestOps是感兴趣的事件集合，在通道调用register方法时会设置此值，表示对什么感兴趣，readyOps是就绪事件集合。


如果对不止一种事件感兴趣，使用或运算符即可   
`int interestSet = SelectionKey.OP_READ | SelectionKey.OP_WRITE;`

readyOps相关的方法：
~~~java
selectionKey.isAcceptable();
selectionKey.isConnectable();
selectionKey.isReadable();
selectionKey.isWritable();
~~~

选择方法：
- int select()：阻塞方法，至少有一个通道处于就绪状态才返回，返回新就绪通道的数量
- int select(long timeout)：阻塞方法，超时返回，返回新就绪通道的数量
- int selectNow()：非阻塞方法，返回新就绪通道的数量


服务端模板代码
~~~java

ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.socket().bind(new InetSocketAddress("localhost", 8080));
ssc.configureBlocking(false);

Selector selector = Selector.open();
ssc.register(selector, SelectionKey.OP_ACCEPT);

while(true) {
    int readyNum = selector.select();
    if (readyNum == 0) {
        continue;
    }
    
    Set<SelectionKey> selectedKeys = selector.selectedKeys();
    Iterator<SelectionKey> it = selectedKeys.iterator();
    
    while(it.hasNext()) {
        SelectionKey key = it.next();
        
        if(key.isAcceptable()) {
            // 接受连接
        } else if (key.isReadable()) {
            // 通道可读
        } else if (key.isWritable()) {
            // 通道可写
        }
        
        it.remove();
    }
}
~~~

参考[http://www.tianxiaobo.com/2018/04/03/Java-NIO%E4%B9%8B%E9%80%89%E6%8B%A9%E5%99%A8/](http://www.tianxiaobo.com/2018/04/03/Java-NIO%E4%B9%8B%E9%80%89%E6%8B%A9%E5%99%A8/)