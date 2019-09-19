---
title: mybatis笔记-1
date: 2019-05-03 17:47:41
tags: mybatis
category: mybatis
---

# mybatis使用初探

简介：MyBatis 是一款持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC 代码和手动设置参数以及获取结果集。MyBatis 可以使用简单的 XML 或注解来配置和映射原生类型、接口和 Java 的 POJO（Plain Old Java Objects）为数据库中的记录。
<!--more-->
## mybatis全局配置文件
一个简单的例子：
~~~xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">

<configuration>

    <!--引入外部资源,注意路径，-->
    <properties resource="db.properties"/>

    <!--环境配置，可以配置多个环境，default选择哪一个环境，比如开发，测试，上线的环境可能不同-->
    <environments default="development">

        <environment id="development">

            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${driver}"/>
                <property name="url" value="${url}"/>
                <property name="username" value="${username}"/>
                <property name="password" value="${password}"/>
            </dataSource>
        </environment>
    </environments>

    <!--引入映射文件-->
    <mappers>
        <mapper resource="mapper/manageMapper.xml"/>
    </mappers>
</configuration>
~~~
### properties标签
这些属性都是可外部配置且可动态替换的，既可以在典型的 Java 属性文件中配置，亦可通过 properties 元素的子元素来传递。   
例如
~~~xml
<properties resource="org/mybatis/example/config.properties">
  <property name="username" value="dev_user"/>
  <property name="password" value="F2Fa3!33TYyg"/>
</properties>
~~~
如果属性在不只一个地方进行了配置，那么 MyBatis 将按照下面的顺序来加载：

- 在 properties 元素体内指定的属性首先被读取。
- 然后根据 properties 元素中的 resource 属性读取类路径下属性文件或根据 url 属性指定的路径读取属性文件，并覆盖已读取的同名属性。
- 最后读取作为方法参数传递的属性，并覆盖已读取的同名属性。

### environments标签

可以配置成适应多种环境，这种机制有助于将 SQL 映射应用于多种数据库之中。例如，开发、测试和生产环境需要有不同的配置；或者想在具有相同 Schema 的多个生产数据库中 使用相同的 SQL 映射。有许多类似的使用场景。

**尽管可以配置多个环境，但每个 SqlSessionFactory 实例只能选择一种环境。** 所以，如果想连接两个数据库，就需要创建两个 SqlSessionFactory 实例，每个数据库对应一个。而如果是三个数据库，就需要三个实例，依此类推。

每个数据库对应一个 SqlSessionFactory 实例，为了指定创建哪种环境，只要将它作为可选的参数传递给 SqlSessionFactoryBuilder 即可。可以接受环境配置的两个方法签名是：
~~~java
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, environment);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, environment, properties);
~~~

如果忽略了环境参数，那么将会加载默认环境，如下所示：
~~~java
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader);
SqlSessionFactory factory = new SqlSessionFactoryBuilder().build(reader, properties);
~~~


### settings标签
这是 MyBatis 中极为重要的调整设置，它们会改变 MyBatis 的运行时行为。   
内容很多，比如是否开启驼峰命名转换，延迟加载，日志，超时时长等等。   
参考文档[http://www.mybatis.org/mybatis-3/zh/configuration.html](http://www.mybatis.org/mybatis-3/zh/configuration.html)
### typeAliases标签
类型别名是为 Java 类型设置一个短的名字。 它只和 XML 配置有关，存在的意义仅在于用来减少类完全限定名的冗余。可以不使用权限定名而使用简短的别名。 

~~~xml
<typeAliases>
  <typeAlias alias="Author" type="domain.blog.Author"/>
  <typeAlias alias="Blog" type="domain.blog.Blog"/>
</typeAliases>
~~~
当然也可以使用注解：
~~~java
@Alias("author")
public class Author {
    ...
}
~~~
mybatis内置了一些常用的别名。包括java中基础类型以及它们的包装类型还有一些集合类。


### mappers标签
需要告诉 MyBatis 到哪里去找到这些语句。 可以使用相对于类路径的资源引用，或完全限定资源定位符（包括 file:/// 的 URL），或类名和包名等。


### plugins标签
允许在已映射语句执行过程中的某一点进行拦截调用。默认情况下，MyBatis 允许使用插件来拦截的方法调用包括：

- Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
- ParameterHandler (getParameterObject, setParameters)
- ResultSetHandler (handleResultSets, handleOutputParameters)
- StatementHandler (prepare, parameterize, batch, update, query)


## mapper文件

将sql语句写在映射文件中，可以将sql与java代码分离，方便管理，并且可以。mybatis的映射文件具有很强大的功能。

### 参数问题

~~~xml
<!--命名空间，类路径-->
<mapper namespace="dao.ManageMapper">
<!--id是方法名 parameterType 参数类型，可以省略；resultType 返回类型 ；使用#{...}占位-->
<select id="selectPerson" parameterType="int" resultType="hashmap">
  SELECT * FROM PERSON WHERE ID = #{id}
</select>
</mapper>
~~~

如果有多个参数呢：

~~~xml
<select id="selectByIdAndName" resultType="entity.Manage">
        select * from manage
        where id=#{id} and name=#{name}
</select>
~~~
**直接这样写会失败**

因为mybatis使用了一个map来存储参数，map< key,value >
- 使用#{arg0},#{arg1}...;或者#{param1}，#{param2}...
- 在接口方法上使用注解@Param("id),Param("name")，这样就可以使用上述的方法
- 使用map，将参数封装到map中，也可以使用上述的方法
- 封装一个传输数据的类

### # 和$的区别
mybatis支持#{}和${}动态绑定参数，但是它们之间有区别。
- #{}像是jdbc支持的占位符，只能用来为可以加引号’的参数（如参数值）设置动态参数，即用?占位，不可用于表名、字段名等。像是预编译，可以防止sql注入
- ${}是直接替换，可能导致不安全，比如sql注入。

一个例子：
~~~xml
    <select id="selectByMap" resultType="entity.Manage">

        select * from ${table}
        where id=#{id} and name=#{name}

    </select>
~~~

加入此时传入的参数，table="manage ^&%FSUq345",那么将会出错。

### 增删改查
增删改查操作都几乎一致

## java代码调用方法

一个简单的例子
~~~java

public class ManageTest {

    //获取mybatis配置文件，并根据配置文件加载一些环境
    String resource= "mybatis-config.xml";
    InputStream in= Resources.getResourceAsStream(resource);
    //获取SqlSessionFactory
    SqlSessionFactory sqlSessionFactory=new SqlSessionFactoryBuilder().build(in);

    public ManageTest() throws IOException {
    }


    @Test
    public void querytOne() throws IOException {
        //获取SqlSession，数据库的增删改查都需要SqlSession，但是它不是线程安全的，不能将它定义为对象的成员属性或者静态变量
        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            //获取接口的代理对象，使用的动态代理
            ManageMapper mapper = sqlSession.getMapper(ManageMapper.class);
            Manage manage = mapper.selectById(1234567890L);
            System.out.println(manage);
        }finally {
            //关闭资源
            sqlSession.close();
        }
    }

    @Test
    public void insert(){
        //不自动提交，如果需要自动提交：sqlSessionFactory.openSession(true);
        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            ManageMapper mapper = sqlSession.getMapper(ManageMapper.class);
            Manage manage=new Manage(996996996L,"spj","123");
            int result=mapper.insert(manage);
            System.out.println(result);
            //提交
            sqlSession.commit();
        }finally {
            sqlSession.close();
        }

    }
    //省略其他测试
}
~~~
以上使用到的接口
~~~java

public interface ManageMapper {

    Manage selectById(long id);

    int insert(Manage manage);

    Manage selectByIdAndName(long id,String name);

    Manage selectByMap(Map map);

    int update(Manage manage);

}
~~~

使用的实体类
~~~java

public class Manage {

    private long id;

    private String name;

    private String password;

    public Manage(){}

    public Manage(long id, String name, String passWord) {
        this.id = id;
        this.name = name;
        this.password = passWord;
    }

    //省略setter,getter,toString方法
}

~~~

参考[http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html](http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html)  