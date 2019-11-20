---
title: mybatis笔记-2
date: 2019-05-04 17:05:57
tags: mybatis
category: 框架
---

# mybatis使用初探2

## ResultMap标签
ResultMap是一个很强大的标签，用于处理列名与类属性的映射。

<!--more-->
一个java类：在类的成员属性中，引用了其他自定义的类
~~~java
import org.apache.ibatis.annotations.Param;
import java.util.Date;

public class BorrowedBook {
    private Book book;
    private Student student;
    private Date starttime;
    private Date deadtime;
    public BorrowedBook(){}  
    public BorrowedBook(@Param("starttime") Date starttime,@Param("deadtime") Date deadtime) {
        this.starttime = starttime;
        this.deadtime = deadtime;
    }

    //省略getter，setter，tostring方法
}
~~~
这是查询语句，比较复杂
~~~xml
<select id="selectAll" resultMap="BorrowedBook">

        select
               b.id as b_id,
               b.name as b_name,
               b.totnum as b_totnum,
               b.nownum as b_totnum,
               b.flag as b_flag ,
               bb.starttime as bb_starttime,
               bb.deadtime as bb_deadtime,
               st.id as st_id,
               st.name as st_name,
               st.password as st_password,
               st.booknum as st_booknum
        from
             book b,
             borrowedbook bb,
             student st
        where
              bb.idbook=b.id and bb.idstud=st.id

</select>
~~~

我们可以定义一个ResultMap
~~~xml
<resultMap id="BorrowedBook" type="entity.BorrowedBook">
      <!--可以其使用构造器注入-->
      <!--  <constructor>
            <arg column="bb_starttime" name="starttime"/>
            <arg column="bb_deadtime" name="deadtime"/>
        </constructor>
        -->

        <!--也可以使用result注入-->
        <result column="bb_starttime" property="starttime"/>
        <result column="bb_deadtime" property="deadtime"/>

        <!--注1-->
        <!--
        <association property="book" resultMap="book"/>
        <association property="student" resultMap="student"/>
        -->
        <!--自定义的类的注入-->
        <association property="book" javaType="entity.Book">
        <!--使用id可以进行一些优化-->
            <id column="b_id" property="id"/>
            <result column="b_name" property="name"/>
            <result column="b_totnum" property="totnum"/>
            <result column="b_flag" property="flag"/>
        </association>

        <association property="student" javaType="entity.Student">

            <id column="st_id" property="id"/>
            <result column="st_name" property="name"/>
            <result column="st_password" property="password"/>
            <result column="st_booknum" property="booknum"/>

        </association>
    </resultMap>
~~~

上面的看起来有一些繁杂，可以将一些标签提取出来，如上面注1时那样使用，可以复用：
~~~xml
<resultMap id="book" type="entity.Book">
            <id column="b_id" property="id"/>
            <result column="b_name" property="name"/>
            <result column="b_totnum" property="totnum"/>
            <result column="b_flag" property="flag"/>

</resultMap>


    <resultMap id="student" type="entity.Student">
        <id column="st_id" property="id"/>
        <result column="st_name" property="name"/>
        <result column="st_password" property="password"/>
        <result column="st_booknum" property="booknum"/>
    </resultMap>
~~~

之前都是返回单个对象或者列表，如果想要返回一个HashMap呢，可以其使用以下方法：

~~~java
    //告诉mybatis返回的key是什么
    @MapKey("student.id")
    Map<String,BorrowedBook> selectAllMap();
~~~
## sql标签
重复的片段可以复用，将其写在sql标签中即可。
~~~xml
<select id="selectAllMap" resultMap="BorrowedBook">

        <include refid="selectAbstract"/>

    </select>
    <sql id="selectAbstract">

        select b.id         as b_id,
               b.name       as b_name,
               b.totnum     as b_totnum,
               b.nownum     as b_totnum,
               b.flag       as b_flag,
               bb.starttime as bb_starttime,
               bb.deadtime  as bb_deadtime,
               st.id        as st_id,
               st.name      as st_name,
               st.password  as st_password,
               st.booknum   as st_booknum
        from book b,
             borrowedbook bb,
             student st
        where bb.idbook = b.id
          and bb.idstud = st.id

    </sql>
    <!--使用，可以直接拼接-->
    <!-- 甚至可以<sql id="userColumns"> ${alias}.id,${alias}.username,${alias}.password </sql> 使用的时候替换即可-->
    <select id="selectByBookId" resultMap="BorrowedBook">
        <include refid="selectAbstract"/>
          and bb.idbook = #{id}
    </select>
~~~

## 分步查询与延迟加载
当我想查询所借的书以及借书的学生的资料时，可以先查询idbook，idsutd，然后执行两条sql语句查询book和student的语句，这就是分步查询。

分步查询的一个好处是可以设置延迟加载，咋在属性需要的时候才执行sql语句，节约资源。

延迟加载需要在全局配置文件中settings中设置：`<setting name="lazyLoadingEnabled" value="true"/>`

例子：
~~~xml
<!--分步查询，延迟加载-->

    <resultMap id="MyBorrowedBook" type="entity.BorrowedBook">
        <result column="starttime" property="starttime"/>
        <result column="deadtime" property="deadtime"/>
        <!--dao.BookMapper.selectBookById这个接口是已经实现的，column="idbook"是传入的参数，如果需要多值参数：比如column="id=idbook，name=bookname"-->
        <association property="book" select="dao.BookMapper.selectBookById" column="id=idbook"/>
        <association property="student" select="dao.StudentMapper.selectStudentById" column="idstud"/>

    </resultMap>
    <!--book.id => idbook,idstud,starttime,deadtime => book and student -->
    <select id="selectByBookId" resultMap="MyBorrowedBook">
        select *
        from borrowedbook
        where idbook = #{id}

    </select>
~~~

测试后，可以看家只有在需要book这个属性的时候，才会执行一条查询book的语句。

## collection标签

如果返回列表，可以使用collection标签,例如：
~~~xml
<resultMap id="MyFoo" type="entity.BookFoo">
        <result property="stuName" column="stuname"/>
        <!--列表属性-->
        <!--"ofType" 属性。它用来将 JavaBean（或字段）属性的类型和集合存储的类型区分开来。 -->
        <collection property="books" ofType="entity.Book">
            <id column="id" property="id"/>
            <result property="name" column="name"/>
        </collection>
</resultMap>
~~~

## 动态sql

按照不同的查询条件拼接sql语句是一件很烦的事情，mybatis为我们开发了动态sql。

### if
`if`用于条件判断，支持OGNL表达式，一个例子
~~~xml
    <select id="selectDynamic" resultType="entity.Book">
        select *
        from book
          where
        <if test="id!=null and id!=''">
            id=#{id}
        </if>
        <if test="name!=null">
            and name like #{name}
        </if>
    </select>
~~~
只是用if会有and或or多余的问题，比如第一个条件不满足，以上语句就会多出一个and，语法错误。  
因此可以使用专门的where标签，去除开始的多余的and或者or
### trim
`where` 标签的使用
~~~xml
<select id="selectDynamic" resultType="entity.Book">
        select *
        from book
        <where>
        <if test="id!=null and id!=''">
            id=#{id}
        </if>
        <if test="name!=null">
            and name like #{name}
        </if>
        <if test="totnum!=null">
            and tounum=#{totnum}
        </if>
        
        </where>
    </select>
~~~
`set`标签用于处理set语句时，多余的`，`:
~~~xml
<update id="updateDynamic" parameterType="entity.Book">
        update book
        <set>
            <if test="name!=null and name!=''">name=#{name}</if>
            <if test="totnum!= null">totnum=#{totnum}</if>
            <if test="nownum!=null">nownum=#{nownum}</if>
        </set>
        <where>
            id=#{id}
        </where>
    </update>
~~~

还有更强大的标签`trim`，支持自定义的去重
~~~xml
<trim prefix="WHERE" prefixOverrides="AND |OR ">
  ...
</trim>
~~~

例如：
~~~xml
<select id="selectDynamic" resultType="entity.Book">
<!--bind用于绑定参数-->
<bind name="name" value="'%'+_parameter.name+'%'"/>
        select *
        from book
        <trim prefix="where" prefixOverrides="and">
            <if test="id!=null and id!=''">
                id=#{id}
            </if>
            <if test="name!=null">
                and name like #{name}
            </if>
            <if test="totnum!=null">
                and tounum=#{totnum}
            </if>
            <if test="nownum!=null">
                and nownum=#{nownum}
            </if>
            <if test="flag!=null">
                and flag=#{flag}
            </if>
        </trim>
    </select>
~~~

### choose（when，otherwise）
有时我们不想应用到所有的条件语句，而只想从中择其一项。针对这种情况，MyBatis 提供了 choose 元素，它有点像 Java 中的 switch 语句。  
~~~xml
<select id="selectDynamic" resultType="entity.Book">
        <if test="_parameter.name!=null and _parameter !=''">
            <bind name="name" value="'%'+_parameter.name+'%'"/>
        </if>
        select *
        from book
        <trim prefix="where" prefixOverrides="and">
            <choose>
                <when test="id!=null and id!=''">
                    id=#{id}
                </when>
                <when test="name!=null">
                    and name like #{name}
                </when>
                <otherwise>
                    1=1
                </otherwise>
            </choose>
        </trim>
    </select>
~~~

### foreach
foreach用于遍历集合中的元素。
一种情景：需要查找id在在一个list中的元素。
~~~xml
<select id="selectIn" resultType="entity.Book">
        select * from book where id in
        <foreach collection="idList" item="id" open="(" close=")" separator=",">
            #{id}
        </foreach>
    </select>
~~~

## 缓存
### 一级缓存
一级缓存针对于SqlSession，一次会话中的数据会被缓存，存到一个Map中。mybatis默认会开启一级缓存。   
一下缓存会被清空    
- 会话关闭或提交
- 执行了增删改操作
- 执行sqlSession.clearCache()方法

#### 流程
SqlSession 一级缓存的工作流程：

1. 对于某个查询，根据statementId,params,rowBounds来构建一个key值，根据这个key值去缓存Cache中取出对应的key值存储的缓存结果；
2. 判断从Cache中根据特定的key值取的数据数据是否为空，即是否命中；
3. 如果命中，则直接将缓存结果返回；
4. 如果没命中：  
 去数据库中查询数据，得到查询结果；  
 将key和查询到的结果分别作为key,value对存储到Cache中；   
 将查询结果返回；   
5. 结束。

MyBatis认为，对于两次查询，如果以下条件都完全一样，那么就认为它们是完全相同的两次查询：   
1. 传入的 statementId 
2. 查询时要求的结果集中的结果范围 （结果的范围通过rowBounds.offset和rowBounds.limit表示）；
3. 这次查询所产生的最终要传递给JDBC java.sql.Preparedstatement的Sql语句字符串（boundSql.getSql() ）
4. 传递给java.sql.Statement要设置的参数值

一下结果返回为true
~~~java
    @Test
    public void querytOne() throws IOException {
        SqlSession sqlSession = sqlSessionFactory.openSession();
        try {
            ManageMapper mapper = sqlSession.getMapper(ManageMapper.class);
            
            Manage manage = mapper.selectById(1234567890);
            //sqlSession.clearCache();返回false
            //sqlSession.commit();返回false
            Manage manage1=mapper.selectById(1234567890);
            System.out.println(manage==manage1);
        }finally {
            sqlSession.close();
        }
    }
~~~

### 二级缓存
使用二级缓存需要配置mybatis全局设置和对应的mapper文件，并且缓存的对象对应的类需要实现反序列化接口。   
在全局配置文件中settings中：
~~~xml
<setting name="cacheEnabled" value="true"/>
~~~

在对应的mapper.xml文件中配置：

~~~xml
<cache ></cache>
~~~

- cache中的属性：  
  - eviction：缓存的回收策略  
  - LRU - 最近最少使用，移除最长时间不被使用的对象  
  - FIFO - 先进先出，按对象进入缓存的顺序来移除它们  
  - SOFT - 软引用，移除基于垃圾回收器状态和软引用规则的对象  
  - WEAK - 弱引用，更积极地移除基于垃圾收集器和弱引用规则的对象   
  - 默认的是LRU   
- flushInterval：缓存刷新间隔，缓存多长时间清空一次，默认不清空，设置一个毫秒值  
- readOnly：是否只读  
  - true：只读：mybatis认为所有从缓存中获取数据的操作都是只读操作，不会修改数据。mybatis为了加快获取数据，直接就会将数据在缓存中的引用交给用户 。不安全，速度快   
  - false：读写(默认)：mybatis觉得获取的数据可能会被修改。mybatis会利用序列化&反序列化的技术克隆一份新的数据给你。安全，速度相对慢   
- size：缓存存放多少个元素   
- type：指定自定义缓存的全类名(实现Cache接口即可)   

二级缓存的生命周期是整个application。   

### 第三方缓存
mybatis可以集成第三方缓存框架。需要引入相应的适配jar包。

## 逆向工程
可以根据数据库生成相应的类和mapper文件。

