---
title: maven总结
date: 2019-04-29 18:34:52
tags:
- maven 
category: 工具
---

## 简介

maven是一个Apache下由java开发的开源项目。基于对象模型的概念(POM)，Maven利用能管理一个项目的构建、报告和文档等步骤。
<!--more-->


## 需求
当写一个项目的时候往往需要引入第三方依赖包，但是一个第三方包可能又要依赖其他的包，一旦依赖过多，就会给开发人员造成很大的负担，而且容易出错。maven使用一个仓库存储需要的第三方库，而且可以自动解决依赖问题。构建过程枯燥重复，使用maven可以减轻构建的烦恼。


## 安装
去[官网](http://maven.apache.org/download.cgi)下载相应系统的文件，解压放入一个目录，设置环境变量。      
![](maven总结/maven下载.png)

也可以使用IDEA中自带的maven，一般使用自己安装的maven，可以在`File | Settings | Build, Execution, Deployment | Build Tools | Maven`中修改配置。   

在maven的settings.xml中添加maven镜像，国内下载依赖会更快。   
~~~xml
<mirror> 
　　<id>alimaven</id> 
　　<name>aliyun maven</name> 
　　<url>http://maven.aliyun.com/nexus/content/groups/public/</url> 
　　<mirrorOf>central</mirrorOf> 
  </mirror>
~~~

## 使用

### 常用命令

构建中常用的命令（需要进入到与pom.xml同目录下）：
- mvn clean :清理之前产生的class文件
- mvn compile ：编译项目
- mvn test-compile：编译测试文件
- mvn test：测试
- mvn package ：将项目打包为
- mvn install :安装到本地仓库

### 目录结构：

约定：
~~~
my-app
|-- pom.xml
`-- src
    |-- main
    |   `-- java
    |       `-- com
    |           `-- mycompany
    |               `-- app
    |                   `-- App.java
    `-- test
        `-- java
            `-- com
                `-- mycompany
                    `-- app
                        `-- AppTest.java

~~~

| 目录                               | 目的                                                                   |
| ---------------------------------- | ---------------------------------------------------------------------- |
| ${basedir}                         | 存放pom.xml和所有的子目录                                              |
| ${basedir}/src/main/java           | 项目的java源代码                                                       |
| ${basedir}/src/main/resources      | 项目的资源，比如说property文件，springmvc.xml                          |
| ${basedir}/src/test/java           | 项目的测试类，比如说Junit代码                                          |
| ${basedir}/src/test/resources      | 测试用的资源                                                           |
| ${basedir}/src/main/webapp/WEB-INF | web应用文件目录，web项目的信息，比如存放web.xml、本地图片、jsp视图页面 |
| ${basedir}/target                  | 打包输出目录                                                           |
| ${basedir}/target/classes          | 编译输出目录                                                           |
| ${basedir}/target/test-classes     | 测试编译输出目录                                                       |
| Test.java                          | Maven默认的本地仓库目录位置                                            |


### 创建java项目
`mvn archetype:generate -DgroupId=com.companyname.bank -DartifactId=consumerBanking -DarchetypeArtifactId=maven-archetype-quickstart -DinteractiveMode=false`  
参数：
- -DgourpId: 组织名，公司网址的反写 + 项目名称
- -DartifactId: 项目名-模块名
- -DarchetypeArtifactId: 指定 ArchetypeId，maven-archetype-quickstart，创建一个简单的 Java 应用
- -DinteractiveMode: 是否使用交互模式


### pom.xml
pom（project object model）项目对象模型

pom.xml是核心文件：
#### 坐标
- groupId：公司域名的倒叙+项目
- artifactId：模块名
- version ：版本号

这三者唯一确定了一个jar包   
举例：
~~~xml
<dependency>
      <groupId>org.springframework</groupId>
      <artifactId>spring-core</artifactId>
      <version>4.1.7.RELEASE</version>
</dependency>
~~~
在本地仓库中路径：org/springframework/spring-core/4.1.7.RELEA/spring-core-4.1.7.RELEASE.jar

#### 仓库
- 本地仓库
- 中央仓库
- 镜像仓库

#### scope
* compile，缺省值如果没有提供一个范围，那该依赖的范围就是编译范围。编译范围依赖在所有的classpath 中可用，同时它们也会被打包。
* provided，期望JDK、容器或使用者会提供这个依赖。不是传递性的，也不会被打包。如servlet.jar。
* runtime，只在运行时使用，如JDBC驱动，适用运行和测试阶段。 
* test，只在测试时使用，用于编译和运行测试代码。不会随项目发布。 
* system，类似provided，需要显式提供包含依赖的jar，Maven不会在Repository中查找它。

#### 构建生命周期
- validate - 验证项目是否正确并且所有必要信息都可用
- compile - 编译项目的源代码
- test - 使用合适的单元测试框架测试编译的源代码。这些测试不应要求打包或部署代码
- package - 获取已编译的代码并将其打包为可分发的格式，例如JAR。
- verify - 对集成测试结果进行任何检查，以确保满足质量标准
- install - 将软件包安装到本地存储库中，以便在本地用作其他项目的依赖项
- deploy - 在构建环境中完成，将最终包复制到远程存储库以与其他开发人员和项目共享。

在执行每一个周期都会先执行之前的周期

#### 依赖冲突

在maven中依赖会传递，A依赖B，B依赖C，在A中会依赖B和C。如果两个jar包的版本不同，就出现了冲突。maven会自动解决冲突，两个原则：  
- 最短路径（依赖路径短的）
- 先声明（如果路径长度一致，后声明的会失效）

使用`mvn dependency:tree`查看所有的依赖树

#### 排除依赖

如果想要排除某个依赖，可以直接排除掉
```xml
<dependency>      
     <groupId>org.apache.hbase</groupId>  
     <artifactId>hbase</artifactId>  
     <version>0.94.17</version>   
     <exclusions>    
           <exclusion>        
                <groupId>commons-logging</groupId>            
                <artifactId>commons-logging</artifactId>    
           </exclusion>    
     </exclusions>    
</dependency>   
```

#### 继承
统一管理版本，子pom依赖父pom，当然你可以复写父pom的内容。

#### 聚合

使用`modules`标签，聚合模块


一个例子：一个父工程A，下面有子模块B，C，D三个子模块   
A的pom.xml文件
~~~xml
<!--聚合模块-->
<modules>
        <module>B</module>
        <module>C</module>
        <module>D</module>
</modules>
<!--依赖管理，管理pom的依赖版本问题，在子pom中不必-->
    <dependencyManagement>

        <dependencies>

            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-core</artifactId>
                <version>4.1.7.RELEASE</version>
            </dependency>

            <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>spring-aop</artifactId>
                <version>4.1.7.RELEASE</version>
            </dependency>
        </dependencies>

    </dependencyManagement>
    
    ...
~~~

B的pom.xml文件
~~~xml
<!--认父亲-->
<parent>
        <artifactId>A</artifactId>
        <groupId>com.zzx.maven</groupId>
        <version>1.0-SNAPSHOT</version>
</parent>
<!--可以自定义属性-->
    <properties>
        <junit.version>4.11</junit.version>
    </properties>


    
<!--可以忽略version-->
<dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aop</artifactId>
        </dependency>
<!--将会用于依赖冲突解决方案-->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.5</version>
        </dependency>

        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>${junit.version}</version>
            <scope>test</scope>
        </dependency>

  </dependencies>
~~~

C的pom.xml文件
~~~xml
<dependencies>
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-core</artifactId>
        </dependency>

        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-aop</artifactId>
        </dependency>
        <!--版本变化-->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.7</version>
        </dependency>
</dependencies>
~~~

D依赖B，C，引入B，C
~~~xml

    <dependencies>
<!--此时在B和C中都依赖了sl4j,不过版本不同-->
        <dependency>
            <groupId>com.zzx.maven</groupId>
            <artifactId>B</artifactId>
            <version>1.0-SNAPSHOT</version>
        </dependency>
        <dependency>
            <groupId>com.zzx.maven</groupId>
            <artifactId>C</artifactId>
            <version>1.0-SNAPSHOT</version>

        </dependency>

    </dependencies>
~~~
通过`mvn dependency:list `查看依赖，可以看见是依赖的B中的sl4j的版本，当交换两个依赖的顺序，再次查看会发现依赖的sl4j版本改变了。

#### Error
在我未install A时，install D会出错。解决办法：先install A(父pom)
~~~
[INFO] ------------------------------------------------------------------------
[INFO] BUILD FAILURE
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  4.283 s
[INFO] Finished at: 2019-04-30T19:20:18+08:00
[INFO] ------------------------------------------------------------------------
[ERROR] Failed to execute goal on project D: Could not resolve dependencies for project com.zzx.maven:D:jar:1.0-SNAPSHOT: Failed to collect dependencies at com.zzx.maven:B:jar:1.0-SNAPSHOT: Failed to read artifact descriptor for com.zzx.maven:B:jar:1.0-SNAPSHOT: Could not transfer artifact com.zzx.maven:B:pom:1.0-SNAPSHOT from/to jdk18 (http://www.myhost.com/maven/jdk18): Failed to transfer file http://www.myhost.com/maven/jdk18/com/zzx/maven/B/1.0-SNAPSHOT/B-1.0-SNAPSHOT.pom with status code 500 -> [Help 1]
[ERROR] 
[ERROR] To see the full stack trace of the errors, re-run Maven with the -e switch.
[ERROR] Re-run Maven using the -X switch to enable full debug logging.
[ERROR] 
~~~
当然maven还用其他的很多功能，比如插件。
