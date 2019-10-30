---
title: mybatis-cache
catalog: true
date: 2019-10-30 20:13:13
subtitle:
header-img:
tags:
---


# Mybatis 缓存
&emsp;使用缓存可以使应用更快地获取数据，避免频繁的数据库交互，尤其是在查询越多、缓存命中率越高的情况下，使用缓存的作用就越明显。MyBatis 作为持久化框架， 提供了非常强大的查询缓存特性，可以非常方便地配置和定制使用。

## 一级缓存
&emsp;Mybatis对缓存提供支持，但是在没有配置的默认情况下，它只开启一级缓存，一级缓存只是相对于同一个SqlSession而言。

![L1](https://github.com/JP6907/Pic/blob/master/java/mybatis/mybatis-cache-L1.png?raw=true)

&emsp;每个SqlSession中持有了Executor，每个Executor中有一个LocalCache。当用户发起查询时，MyBatis根据当前执行的语句生成MappedStatement，在Local Cache进行查询，如果缓存命中的话，直接返回结果给用户，如果缓存没有命中的话，查询数据库，结果写入Local Cache，最后返回结果给用户。使用SelSession第一次查询后，MyBatis会将其放在缓存中，以后再查询的时候，如果没有声明需要刷新，并且缓存没有超时的情况下，SqlSession都会取出当前缓存的数据，而不会再次发送SQL到数据库。

![Sqlsession-cache-class](https://github.com/JP6907/Pic/blob/master/java/mybatis/Sqlsession-cache-class.jpg?raw=true)


&emsp;有几个问题需要注意一下：
1. 一级缓存的生命周期有多长？
- a. 一级缓存存在于 SqlSession 的生命周期中，MyBatis在开启一个数据库会话时，会创建一个新的SqlSession对象，SqlSession对象中会有一个新的Executor对象。Executor对象中持有一个新的PerpetualCache对象；当会话结束时，SqlSession对象及其内部的Executor对象还有PerpetualCache对象也一并释放掉。
- b. 如果SqlSession调用了close()方法，会释放掉一级缓存PerpetualCache对象，一级缓存将不可用。
- c. 如果SqlSession调用了clearCache()，会清空PerpetualCache对象中的数据，但是该对象仍可使用。
- d. SqlSession中执行了任何一个update操作(update()、delete()、insert()) ，都会清空PerpetualCache对象的数据，但是该对象可以继续使用。

2. 怎么判断某两次查询是完全相同的查询？
mybatis认为，对于两次查询，如果以下条件都完全一样，那么就认为它们是完全相同的两次查询。
- a. 传入的statementId
- b. 查询时要求的结果集中的结果范围
- c. 这次查询所产生的最终要传递给JDBC java.sql.Preparedstatement的Sql语句字符串（boundSql.getSql() ）
- d. 传递给java.sql.Statement要设置的参数值

MyBatis 会把执行的方法和参数通过算法生成缓存的键值，将键值和查询结果存入一个Map对象中。如果同一个SqlSession 中执行的方法和参数完全一致，那么通过算法会生成相同的键值，当Map 缓存对象中己经存在该键值时，则会返回缓存中的对象。

&emsp;我们看一下一个例子：
```java
public class BaseMapperTest {
    private static SqlSessionFactory sqlSessionFactory;

    @BeforeClass
    public static void init(){
        try{
            Reader reader = Resources.getResourceAsReader("mybatis-config.xml");
            sqlSessionFactory = new SqlSessionFactoryBuilder().build(reader);
            reader.close();
        }catch (IOException ex){
            ex.printStackTrace();
        }
    }

    public SqlSession getSqlSession(){
        return sqlSessionFactory.openSession();
    }
}
```
```java
public class CacheTest extends BaseMapperTest {
    @Test
    public void testL1Cache(){
        SqlSession sqlSession = getSqlSession();
        SysUser user1 = null;
        try{
            UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
            user1 = userMapper.selectById(1L);
            System.out.println(user1);
            //对当前获取的对象重新赋值
            user1.setUserName("New Name");
            //再次获取相同id的用户
            SysUser user2 = userMapper.selectById(1L);
            Assert.assertEquals("New Name",user2.getUserName());
            //user1和user2是同一个实例
            System.out.println(user1==user2);
            Assert.assertEquals(user1,user2);
            System.out.println(user1);
            System.out.println(user2);
            //对user1的操作等同于对user2的操作
            user1.setUserName("Hello World");
            Assert.assertEquals("Hello World",user2.getUserName());
            //看日志，两次select，实际上只执行了一个数据库操作
        }finally {
            sqlSession.close();
        }

        System.out.println("开启新的sqlSession");
        sqlSession = getSqlSession();
        try{
            UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
            SysUser user2 = userMapper.selectById(1L);
            Assert.assertEquals("admin",user2.getUserName());
            //跟前一个session的查询结果不是同一个实例
            Assert.assertNotEquals(user1,user2);
            //执行删除操作
            userMapper.deleteById(2L);
            SysUser user3 = userMapper.selectById(1L);
            //不是同个实例
            Assert.assertNotEquals(user2,user3);
        }finally {
            sqlSession.close();
        }
    }
}
```
&emsp;执行结果为：
```java
DEBUG [main] - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@3d121db3]
DEBUG [main] - ==>  Preparing: select * from sys_user where id = ?; 
DEBUG [main] - ==> Parameters: 1(Long)
TRACE [main] - <==    Columns: id, user_name, user_password, user_info, head_img, create_time
TRACE [main] - <==        Row: 1, admin, 123456, <<BLOB>>, <<BLOB>>, 2019-10-28 20:37:10
DEBUG [main] - <==      Total: 1
SysUser{id=1, userName='admin', userPassword='123456', userInfo='管理员', headImg=null, createTime=2019-10-28}
true
SysUser{id=1, userName='New Name', userPassword='123456', userInfo='管理员', headImg=null, createTime=2019-10-28}
SysUser{id=1, userName='New Name', userPassword='123456', userInfo='管理员', headImg=null, createTime=2019-10-28}
DEBUG [main] - Resetting autocommit to true on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@3d121db3]
DEBUG [main] - Closing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@3d121db3]
开启新的sqlSession
DEBUG [main] - Opening JDBC Connection
DEBUG [main] - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@7880cdf3]
DEBUG [main] - ==>  Preparing: select * from sys_user where id = ?; 
DEBUG [main] - ==> Parameters: 1(Long)
TRACE [main] - <==    Columns: id, user_name, user_password, user_info, head_img, create_time
TRACE [main] - <==        Row: 1, admin, 123456, <<BLOB>>, <<BLOB>>, 2019-10-28 20:37:10
DEBUG [main] - <==      Total: 1
DEBUG [main] - ==>  Preparing: delete from sys_user where id = ? 
DEBUG [main] - ==> Parameters: 2(Long)
DEBUG [main] - <==    Updates: 0
DEBUG [main] - ==>  Preparing: select * from sys_user where id = ?; 
DEBUG [main] - ==> Parameters: 1(Long)
TRACE [main] - <==    Columns: id, user_name, user_password, user_info, head_img, create_time
TRACE [main] - <==        Row: 1, admin, 123456, <<BLOB>>, <<BLOB>>, 2019-10-28 20:37:10
DEBUG [main] - <==      Total: 1
DEBUG [main] - Rolling back JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@7880cdf3]
DEBUG [main] - Resetting autocommit to true on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@7880cdf3]
DEBUG [main] - Closing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@7880cdf3]
```
&emsp;可以看到在第一个 sqlSession 里面执行了两次 select 操作，但是实际日志却只有一次数据库查询操作，说明第二次 select 是从缓存中获取的。另外，两次查询操作返回的是同一个实例，对 user1 的操作能够在 user2 上体现出来，同时也证明了 user1 和 user2 是对同一个实例的引用。
&emsp;在第二个 sqlsession 我们通过 user1 不等于 user2 证明了不同 session 的相同操作返回的是不同的实例。另外，user2 不等于 user3 说明在执行 delete 之后缓存被清除了，user3 得到的是新的实例。

&emsp;在同个 sqlsession 中反复使用相同参数执行同一个方法时， 总是返回同一个对象，因此就会出现上面测试代码中的情况。在使用 MyBatis 的过程中，要避免在使用如上代码中的 user2 时出现的错误。我们可能以为获取的user2 应该是数据库中的数据，却不知道 userl 的一个重新赋值会影响到 user2 。如果不想让 selectByid 方法使用一级缓存，可以设置 flushCache= "true"，这个属性配置为true 后， 会在查询数据前清空当前的一级缓存，因此该方法每次都会重新从数据库中查询数据，此时的 user2 和userl 就会成为两个不同的实例， 可以避免上面的问题。但是由于这个方法清空了一级缓存， 会影响当前SqlSession 中所有缓存的查询，因此在需要反复查询获取只读数据的情况下，会增加数据库的查询次数，所以要避免这么使用。


## 二级缓存
&emsp;MyBatis的二级缓存是Application级别的缓存，它可以提高对数据库查询的效率，以提高应用的性能。

![L2](https://github.com/JP6907/Pic/blob/master/java/mybatis/mybatis-cache-L2.png?raw=true)


&emsp;在一级缓存中，其最大的共享范围就是一个SqlSession内部，如果多个SqlSession之间需要共享缓存，则需要使用到二级缓存。开启二级缓存后，会使用CachingExecutor装饰Executor，进入一级缓存的查询流程前，先在CachingExecutor进行二级缓存的查询，具体的工作流程如下所示。

![mybatis-cache-L2-class](https://github.com/JP6907/Pic/blob/master/java/mybatis/mybatis-cache-L2-class.jpg?raw=true)

&emsp;二级缓存开启后，同一个 namespace下的所有操作语句，都影响着同一个Cache，即二级缓存被多个 SqlSession 共享，是一个全局的变量。
&emsp;当开启缓存后，数据的查询执行的流程就是 二级缓存 -> 一级缓存 -> 数据库。

&emsp;首先需要在 mybatis-config.xml 中开启二级缓存
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <settings>
        <!--这个配置使全局的映射器(二级缓存)启用或禁用缓存-->
        <setting name="cacheEnabled" value="true" />
        .....
    </settings>
    ....
</configuration>
```
&emsp;还需要在 mapper 映射文件中配置二级缓存：
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
    <mapper namespace="com.jp.mybatis.dao.StudentMapper">
        <cache eviction="LRU" flushInterval="100000" readOnly="true" size="1024"/>
    </mapper>
</mapper>   
```
&emsp;也可以在 Mapper 类使用注解配置：
```java
@CacheNamespace(
        eviction = FifoCache.class,
        flushInterval = 60000,
        size = 512,
        readWrite = true
)
```

&emsp;默认的二级缓存会有如下效果：
- 映射语句文件中的所有SELECT 语句将会被缓存。
- 映射语句文件中的所有时SERT 、UPDATE 、DELETE 语句会刷新缓存。
- 缓存会使用Least Rece ntly U sed ( LRU ，最近最少使用的）算法来收回。
- 根据时间表（如no Flush Int erv al ，没有刷新间隔），缓存不会以任何时间顺序来刷新。
- 缓存会存储集合或对象（无论查询方法返回什么类型的值）的102 4 个引用。
- 缓存会被视为read/write （可读／可写）的， 意味着对象检索不是共享的，而且可以安全地被调用者修改，而不干扰其他调用者或线程所做的潜在修改。

&emsp;下面介绍一些几个重要的参数：
- eviction （收回策略）
    - LRU （最近最少使用的） ： 移除最长时间不被使用的对象，这是默认值。
    - FIFO （先进先出〉： 按对象进入缓存的顺序来移除它们。
    - SOFT （软引用） ： 移除基于垃圾回收器状态和软引用规则的对象。
    - WEAK （弱引用） ： 更积极地移除基于垃圾收集器状态和弱引用规则的对象。
- flushinterval （刷新间隔〉。可以被设置为任意的正整数， 而且它们代表一个合理的毫秒形式的时间段。默认情况不设置，即没有刷新间隔， 缓存仅仅在调用语句时刷新。
- size （引用数目） 。可以被设置为任意正整数，要记住缓存的对象数目和运行环境的可用内存资源数目，默认值是1024。
- readOnly （只读）。属性可以被设置为true 或false 。只读的缓存会给所有调用者返回缓存对象的相同实例，因此这些对象不能被修改， 这提供了很重要的性能优势。可读写的缓存会通过序列化返回缓存对象的拷贝， 这种方式会慢一些，但是安全，因此默认是false 。

&emsp;如果配置的是可读写的缓存，而MyBatis使用SerializedCache (org.apache.ibatiS.cache.decorators.SerializedCache) 序列化缓存来实现可读写缓存类，井通过序列化和反序列化来保证通过缓存获取数据时，得到的是一个新的实例，此时缓存类需要实现 Serializable 接口。如果配置为只读缓存， MyBatis 就会使用 Map 来存储缓存值，这种情况下，从缓存中获取的对象就是同一个实例。

&emsp;下面看一个例子：
```java
@Test
    public void testL2Cache(){
        SqlSession sqlSession = getSqlSession();
        SysRole role1 = null;
        try {
            //此时二级缓存没有数据，使用的是一级缓存
            RoleMapper roleMapper = sqlSession.getMapper(RoleMapper.class);
            role1 = roleMapper.selectById(1L);
            //读写是不安全的，会影响下面的role2
            role1.setRoleName("New Name");

            SysRole role2 = roleMapper.selectById(1L);
            //虽然没有更新，但是名字和role1一样
            Assert.assertEquals("New Name",role2.getRoleName());
            //是同个实例
            ///这里使用的是一级缓存，所以是同个实例
            System.out.println(role1==role2);
            Assert.assertEquals(role1,role2);
            //二级缓存中没有，所以两次查询名中率都是0
        }finally {
            //close之后，sqlSession会保存查询数据到二级缓存中
            sqlSession.close();
        }

        System.out.println("开启新的sqlSession");
        sqlSession = getSqlSession();
        try{
            RoleMapper roleMapper = sqlSession.getMapper(RoleMapper.class);
            //第三次查询，日志没有输出数据库查询，二级缓存命中，命中率为0.333
            SysRole role2 = roleMapper.selectById(1L);
            Assert.assertNotEquals(role1,role2);
            //第四次查询，2次命中，命中率为0.5
            SysRole role3 = roleMapper.selectById(1L);
            //二级缓存设置为可读写缓存，role2和role3是反序列化得到的结果
            //所以是不同的实例
            //这两个实例是读写安全的，其属性不会相互影响
            Assert.assertNotEquals(role2,role3);
        }finally{
            sqlSession.close();
        }
    }
```
&emsp;输出结果为：
```java
DEBUG [main] - Setting autocommit to false on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@65d6b83b]
DEBUG [main] - ==>  Preparing: select id,role_name,enabled,create_by,create_time from sys_role where id = ? 
DEBUG [main] - ==> Parameters: 1(Long)
TRACE [main] - <==    Columns: id, role_name, enabled, create_by, create_time
TRACE [main] - <==        Row: 1, 管理员, 1, 1, 2019-10-28 20:38:35
DEBUG [main] - <==      Total: 1
DEBUG [main] - Cache Hit Ratio [com.jp.mapper.RoleMapper]: 0.0
true
DEBUG [main] - Resetting autocommit to true on JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@65d6b83b]
DEBUG [main] - Closing JDBC Connection [com.mysql.cj.jdbc.ConnectionImpl@65d6b83b]
开启新的sqlSession
DEBUG [main] - Cache Hit Ratio [com.jp.mapper.RoleMapper]: 0.3333333333333333
DEBUG [main] - Cache Hit Ratio [com.jp.mapper.RoleMapper]: 0.5
```
&emsp;可以看到，只有第一个 sqlSession 的第一次 select 操作会触发实际的数据库查询，第一个 sqlSession 的第二次查询返回的是和第一次 select 的同个实例，此时使用的是一级缓存。当调用close 方法关闭SqlSession 时， SqlSession 才会保存查询数据到二级缓存中。在这之后二级缓存才有了缓存数据。所以可以看到在第一部分的两次查询时，命中率都是0 。

&emsp;在第二部分测试代码中，再次获取role2 时，日志中并没有输出数据库查询，而是输出了命中率，这时的命中率是0.3333333333333333 。这是第3 次查询，并且得到了缓存的值，因此该方法一共被请求了3 次，有l 次命中，所以命中率就是三分之一。后面再获取role3 的时候，就是4 次请求， 2 次命中，命中率为0.5 。并且因为可读写缓存的缘故， role2 和role3 都是反序列化得到的结果，所以它们不是相同的实例。在这一部分，这两个实例是读写安全的，其属性不会互相影响。

在这个例子中并没有真正的读写安全，为什么？
因为这个测试中加入了一段不该有的代码，即 rolel.setRoleName("New Name");,这里修改 role1 的属性位后，按照常理应该更新数据，更新后会清空一、二级缓存，这样在第二部分的代码中就不会出现查询结果的 roleName 都是 "New Name"，的情况了。所以想要安全使用，需要避免毫无意义的修改。这样就可以避免人为产生的脏数据，避免缓存和数据库的数据不一致。



## EhCache缓存/Redis缓存


## 脏数据的产生和避免


> 参考：
《Mybatis从入门到精通》
https://www.cnblogs.com/happyflyingpig/p/7739749.html
https://www.oschina.net/question/54100_2279823