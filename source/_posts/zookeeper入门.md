---
title: zookeeper入门
date: 2019-05-19 22:37:33
tags: 
- zookeeper
- 分布式
category: zookeeper
---

# zookeeper入门

## 概览

### 简介

ZooKeeper是分布式应用程序的分布式开源协调服务。它被设计成易于编程，并且使用了一种以文件系统的熟悉目录树结构为风格的数据模型。它在Java中运行，并支持Java和c。

### 设计目标

ZooKeeper允许分布式进程通过共享的**分层命名空间**相互协调，该命名空间的组织类似于标准文件系统。这些命名空间包括了数据，zookeeper称其为znodes——这些存储节点类似于文件和目录。与为存储而设计的典型文件系统不同，ZooKeeper数据保存在内存中，这意味着ZooKeeper可以实现高吞吐量和低延迟。

ZooKeeper的实现高度重视**高性能**、**高可用性**和**严格有序**的访问。zookeeper的高性能意味着它可以用在大型分布式系统中。可靠性方面防止它成为单点故障。严格的排序是一致性的保证之一。
<!--more-->
zookeeper的服务结构：   
![zkServer](/zookeeper入门/zkservice.jpg)

zookeeper的服务器必须互相了解。它们维护内存中的状态映像，以及持久存储中的事务日志和快照。只要大多数服务器(**可用的服务器大于总服务器数量的一半**)可用，zookeeper就可用.也就是建议服务器的数量为奇数，比如3台服务器挂掉两台就不能工作了，而4台服务器挂掉两台也zookeeper就不能工作。


客户端连接一个服务器，通过维护一个tcp连接，用来发送请求，接受响应，监听事件，心跳检测，如果服务端的tcp连接断开，那么客户端可以连接不同的服务器。

zookeeper的事件执行有序，并且会记录顺序。

zookeeper很快，在以读为主的系统中会更快，因为读只需要在本地读即可，而写需要通过leader分发任务等多个操作。

### 数据模型和分层命名空间

ZooKeeper提供的名称空间很像标准文件系统。名称是由斜杠(/)分隔的路径元素序列。zookeeper名称空间中的每个节点都由一条路径标识。

![zkNamespace](/zookeeper入门/zknamespace.jpg)

可以看见和文件系统一样是一个树形结构。

### 节点和短暂节点

zookeeper中的每一个节点都可以关联数据，拥有子节点，就像是“文件夹”一样。

#### stat结构：

Znodes维护一个stat结构，其中包括数据更改、ACL（access control list）更改和时间戳的版本号，以允许缓存验证和协调更新。每次znode的数据发生变化，版本号都会增加。

- cZxid: 创建znode的更改的事务ID
- mZxid: znode最后一次修改的更改的事务ID
- pZxid: znode关于添加和移除子结点更改的事务ID
- ctime: 表示znode的创建时间 (以毫秒为单位)
- mtime: 表示znode的最后一次修改时间 (以毫秒为单位)
- dataVersion: znode上数据变化的次数
- cversion: znode子结点变化的次数
- aclVersion znode结点上ACL变化的次数
- ephemeralOwner: 如果znode是短暂类型的结点，这代表了znode拥有者的session ID，如果znode不是短暂类型的结点，这个值为0
- dataLength: znode数据域的长度
- numChildren: znode中子结点的个数


存储在命名空间中每个znode的数据以原子方式读写。读取获取与znode相关联的所有数据字节，写入替换所有数据。每个节点都有一个访问控制列表（ACL），该列表限制了谁可以做什么。类似linux中的文件的权限控制。

#### 持久节点和短暂节点
节点可以分为两种：持久节点，短暂节点。
持久节点就是平时使用的节点，只要不删除它，它会一直存在，与之对应的    
短暂节点：只要创建节点的会话还是active的，那么这个节点就会存在，否则这个节点会被删除。也就是时候它是临时存在的。当然也可以显示的删除。并且短暂节点不能有子节点。

### 有条件的更新和监视

想想一下如果没有监听，那么客户端每次访问服务端都要获取所有的数据，那么代价会很大。如果需要实时的了解服务端的变化，那么需要轮询服务器，代价太大，而zookeeper使用监听器：       
zookeeper支持在监听。客户端可以再znode设置一个监听器，当这个znode发生变化时，这个监听器会被触发，然后发送一条数据给客户端，然后这个监听器会被**删除**，也就是一个监听器只能使用一次。

每一个节点都有一个版本号，随着数据变更而递增，如果需要更改某个节点，那么命令所携带的版本参数必须正确，否则会失败。

### 性质

- 有序性：客户端的updates将会有序执行。
- 原子性：要么不执行，要么全部执行完成
- 单一系统映像：无论连接哪一个服务器，看到的视图都是一致的。
- 可靠性：一旦应用了更新，它将从那时起一直持续到客户端覆盖更新。
- 及时性：保证客户对系统看见的在一定时限内是最新的。


ZooKeeper服务的每台服务器都复制其自己的每个组件的副本。复制的本身的所有的数据树，用来持久化。


## 安装使用

相信你已经对zookeeper有了一定的了解，那么接下来就该实战了：

### 下载安装

参照他人博客即可   

在同一台机器上可以部署单机环境，也可以使用多个配置文件来伪装集群，但是配置得时候注意端口不可重复。

### java访问zookeeper Service、
没有实际意义的demo
```java

import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

import java.io.IOException;
import java.util.List;


public class One {
    private static String connectionName="127.0.0.1:2181";
    private static int sessionTimeout=2000;
    private static ZooKeeper zk;

    static {
        try {
            zk = new ZooKeeper(connectionName, sessionTimeout, null);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public One() throws IOException {
    }

    public static void main(String[] args) throws IOException, KeeperException, InterruptedException {
        /*String path1 = createNode(zk);
        System.out.println(path1);
        byte[] data = getNodeData(zk);
        System.out.println(new String(data));*/

        /*Stat stat=setNodeData(zk);
        System.out.println(stat);*/
        /*delete(zk);*/
        System.out.println(getChild(zk));
    }

    private static List<String> getChild(ZooKeeper zk) throws KeeperException, InterruptedException {
        List<String> children = zk.getChildren("/zzxtest", new Watcher() {
            @Override
            public void process(WatchedEvent watchedEvent) {
                try {
                    System.out.println(zk.getChildren("/zzxtest",null));
                } catch (KeeperException e) {
                    e.printStackTrace();
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        });
        Thread.sleep(Long.MAX_VALUE);
        return children;
    }
    //删除节点
    private static void delete(ZooKeeper zk) throws KeeperException, InterruptedException {
        zk.delete("/zzx0",-1);
    }
    //设置节点数据
    private static Stat setNodeData(ZooKeeper zk) throws KeeperException, InterruptedException {
        Stat stat = zk.setData("/zzx1", "hello zzx".getBytes(), -1);
        return stat;
    }
    //获取节点内容
    private static byte[] getNodeData(ZooKeeper zk) throws KeeperException, InterruptedException {
        byte[] data = zk.getData("/zzx1", false, null);
        return data;
    }
    //创建节点
    private static String createNode(ZooKeeper zk) throws KeeperException, InterruptedException {
        String path = zk.create("/zzx1", "hello".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
        return path;
    }
}
```


参考[https://zookeeper.apache.org/doc/r3.4.14/](https://zookeeper.apache.org/doc/r3.4.14/)