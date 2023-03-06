## MyBatis

### 介绍

MyBatis 是一款优秀的持久层框架，它支持定制化 SQL、存储过程以及高级映射。MyBatis 避免了几乎所有的 JDBC  代码和手动设置参数以及获取结果集。MyBatis 可以使用简单的 XML 或注解来配置和映射原生信息，将接口和 Java 的  POJOs(Plain Ordinary Java Object,普通的 Java对象)映射成数据库中的记录。

官网书册：https://mybatis.net.cn/

MyBatis帮助程序员将数据存入到数据库中，非常方便，传统的JDBC代码太复杂了，Mybatis对其进行了简化。



### 搭建测试环境

#### 数据库

![](https://static.sechelper.com/img/2023/03/06/098237bde18e7623e61dd21e33b5dead.png)

新建maven项目并导入依赖

![](https://static.sechelper.com/img/2023/03/06/098237bde18e7623e61dd21e33b5dead.png)



#### MyBatis的核心配置文件

MyBatis的核心配置文件，XML构造SqlSessionFactory；

XML 配置文件中包含了对 MyBatis 系统的核心设置，包括获取数据库连接实例的数据源（DataSource）以及决定事务作用域和控制方式的事务管理器（TransactionManager）。  

名字一般起为：mybatis-config.xml

在resources里

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667286987893-e1f4e39b-83f9-4c2f-825b-1484f46de2d5.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667287899783-07c5c450-d659-4a9d-9dba-afa0c27dae81.png)

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<!--configuration核心配置文件-->
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://localhost:3306/mybatis?useSSL=true&amp;useUnicode=true&amp;characterEncoding=UTF-8&amp;serverTimezone=GMT"/>
                <property name="username" value="root"/>
                <property name="password" value="root"/>
            </dataSource>
        </environment>
    </environments>
    <!--每一个mapper.xml都需要在Mybatis核心配置文件中注册-->
    <mappers>
        <mapper resource="com/dong/dao/UserMapper.xml"/>
    </mappers>
</configuration>
```

jdbc:mysql://localhost:3306是idea连接数据库给的地址

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667288023278-d3ffa132-d41d-40f8-a38f-d361d26c55ae.png)

&amp；是&的意思



#### MyBatis工具类

每个基于 MyBatis 的应用都是以一个 SqlSessionFactory 的实例为核心的。SqlSessionFactory 的实例可以通过SqlSessionFactoryBuilder 获得。而 SqlSessionFactoryBuilder 则可以从 XML 配置文件或一个预先配置的 Configuration 实例来构建出 SqlSessionFactory 实例。  

```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
```

实现创建一个MybatisUtils，java工厂类供获取sqlSession

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667292203898-d6e971d7-eb90-4b13-8707-42db2f4e3307.png)

```java
package com.dong.utils;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;
import java.io.InputStream;

//sqlSessionFactory  -->  sqlSession
public class MybatisUtils {
    static {
        try {
            //使用Mybatis的第一步，获取sqlSessionFactory对象；这一步是固定的
            String resource = "mybatis-config.xml";		//这里是自己的MyBatis配置文件名
            InputStream inputStream = Resources.getResourceAsStream(resource);
            SqlSessionFactory sqlSessionFactory = new SqlSessionFactoryBuilder().build(inputStream);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

从SqlSessionFactory 中获取SqlSession

SqlSession完全包含了面向数据库执行 SQL 命令的所有方法

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667306311490-382b669c-2c8c-4cda-b1c6-dde69d7d67bc.png)

#### 创建实体类

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667306417503-764db543-3cf4-4c3f-9bdb-cfd7ae76e19a.png)

```java
package com.dong.pojo;

public class User {
    private int id;
    private String name;
    private String pwd;

    public User() {
    }

    public User(int id, String name, String pwd) {
        this.id = id;
        this.name = name;
        this.pwd = pwd;
    }

    public int getId() {
        return id;
    }

    public void setId(int id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getPwd() {
        return pwd;
    }

    public void setPwd(String pwd) {
        this.pwd = pwd;
    }

    @Override
    public String toString() {
        return "User{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", pwd='" + pwd + '\'' +
                '}';
    }
}
```

#### Mappr接口

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667306491630-dedcc980-7b3f-433c-b58a-15b1bbadc131.png)

```java
public interface UserDao {
        List<User> getUserList();
}
```



#### 接口实现类（Mapper配置文件）

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667306567523-d924522f-7d06-4927-8d78-5651723bf9c1.png)	

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
"http://mybatis.org/dtd/mybatis-3-mapper.dtd">

<!--namespace=绑定一个对应的Mapper接口-->
<mapper namespace="com.dong.dao.UserDao">
  <!--id是这个接口定义的方法名字--><!--resutType执行sql返回的结果集，要写全限定名：把类及类所在的包都需要写上，后面会有别名简化-->
  <select id="UserDao" resultType="com.dong.pojo.User">
    select * from mybatis.user
  </select>
</mapper>
```











增删改查：

UserMapper.xml

​	1. resutType：			执行sql返回的结果集

​	2. parameterType：	输入类型





![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667390921844-764ea7e5-ccea-4626-8cff-7b4edd6e208e.png)



UserMapper接口

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667390958929-6e6e4295-98e7-4421-b165-b7a2a11befa8.png)

**UserDaoTest** 测试类

```xml
package com.dong.dao;

import com.dong.pojo.User;
import com.dong.utils.MybatisUtils;
import org.apache.ibatis.session.SqlSession;
import org.junit.Test;

import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Objects;

public class UserDaoTest {

    @Test
    public void test(){
        //获取SqlSession对象
        SqlSession sqlSession = MybatisUtils.getSqlSession();
        //方式一：执行SQL；getMapper
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);//获取mapper
        List<User> userList = mapper.getUserList();
        //方式而：
//        sqlSession.selectList("com.dong.dao.UserMapper.getUserList");
        for (User user : userList) {
            System.out.println(user);
        }
        //关闭SqlSession
        sqlSession.close();
    }
    @Test
    public void getUserById(){
        SqlSession sqlSession = MybatisUtils.getSqlSession();
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        User userById = mapper.getUserById(1);
        System.out.println(userById);
        sqlSession.close();
    }
    //增加一个用户；增删改必须提交事务
    @Test
    public void addUser(){
        SqlSession sqlSession = MybatisUtils.getSqlSession();
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        int re = mapper.addUser(new User(4, "筑梦小丑", "23333"));
        if (re > 0){
            System.out.println("添加成功");
        }
        //提交事务
        sqlSession.commit();
        sqlSession.close();
    }
    //修改一个用户
    @Test
    public void updateUser(){
        SqlSession sqlSession = MybatisUtils.getSqlSession();
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        int re = mapper.updateUser(new User(4, "筑夣小丑", "233233"));
        if (re > 0){
            System.out.println("修改成功");
        }
        sqlSession.commit();
        sqlSession.close();
    }

    //删除一个用户
    @Test
    public void deleteUser(){
        SqlSession sqlSession = MybatisUtils.getSqlSession();
        UserMapper mapper = sqlSession.getMapper(UserMapper.class);
        int i = mapper.deleteUser(4);
        if (i > 0 ){
            System.out.println("删除成功");
        }
        sqlSession.commit();
        sqlSession.close();
    }
}
```



万能Map

假设我们的实体类，或者数据库中的表，字段或者参数过多，我们应当考虑使用Map

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667365749047-4cae50dd-74f5-4a13-8397-ebcb9e40e325.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667365763100-7823ea6f-f632-4aed-a2c7-0fc7ce5ab34a.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667365775503-49119333-ee30-45df-bc8d-775541d17744.png)



-----------------------------以上学习完是入门，后面才是核心---------------------------------

### 配置解析

#### 核心配置文件

- mybatis-config.xml
- Mybatis的配置文件包含了会深深影响Mybatis行为的设置和属性信息
- configuration（配置）
- environment（环境变量）
- transactionManager（事务管理器）{默认JDBC，有两种，知道就行}
- dataSource（数据源）{默认POOLED，连接后不会扔掉，下次连接更快。有多个模式，知道就行}
- properties（属性）
- settings（设置）
- typeAliases（类型别名）
- typeHandlers（类型处理器）
- objectFactory（对象工厂）
- plugins（插件）
- environments（环境配置）
- databaseIdProvider（数据库厂商标识）
- mappers（映射器）



#### 环境配置

MyBatis 可以配置成适应多种环境

**不过要记住：尽管可以配置多个环境，但每个 SqlSessionFactory 实例只能选择一种环境。**

学会使用配置多套运行环境，比如如下这种方式就可以选择id为test的配置环境，虽然有多套配置环境，但是最终运行的只会是其中一种



#### 属性（properties）

我们可以通过properties属性来实现引用配置文件

这些属性都是可外部配置且可动态替换的，既可以在典型的Java属性文件中配置，也可通过properties元素的子元素来传递

编写一个配置文件【db.properties】  

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667377542533-2c9ca8ee-8378-48b5-9f3c-162a65505d84.png)

```xml
driver=com.mysql.cj.jdbc.Driver
jdbcUrl=jdbc:mysql://localhost:3306/mybatis?useSSL=false&useUnicode=true&characterEncoding=UTF-8&serverTimezone=GMT
username=root
password=root
```

&amp;要改成& 

url要改成jdbcUrl



引入配置文件：

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667377761720-e7283d70-6268-4760-a2fb-5e3318f2c320.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667377794440-3aa81b10-f10c-4ffa-995f-be13909090e8.png)

属性也可以在外部 property中写入；在读取配置时，遇到相同属性的会进行覆盖，优先级最低的是resource中外部配置文件，所以最终有相同属性的以配置文件为准



#### 类型别名

<typeAliases>

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667378771820-a2a3648e-ad35-4181-810b-865574b31bc4.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667378828192-461fd176-1803-4e24-aee0-cd428fced1fb.png)

通过指定包的名，MyBatis会在包名下搜索需要的java Bean，扫描实体类的包，它默认别名就是这个类的类名

要想自定义别名则需要在实体类上增加注解

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667378886203-3d663d5c-4259-4443-8e40-dfdf9e15fd50.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667379493385-7290b51b-c7d1-4ab1-ab2a-c3f801bde890.png)



#### 映射器

 MapperRegistry:注册绑定我们的Mapper文件；  

每一个Mapper.xml都需要在Mybatis核心配置文件中注册

方式一：xml

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667386528706-4aee6509-8bbc-4435-b16b-f582c17e2f3e.png)

```xml
    <mappers>
        <mapper resource="com/dong/dao/UserMapper.xml"/>
    </mappers>
```

方式二：class

```xml
    <mappers>
        <mapper class="com.dong.dao.UserMapper"/>
    </mappers>
```

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667386478003-f987d4fd-2c2c-4378-a200-bf3dc8c6f0c8.png)

但使用方法二有要求：

接口和它的Mapper配置文件必须同名

接口和它的Mapper配置文件必须在同一个包下



方式三：包

将包的映射器接口实现全部注册为映射器

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667387085117-b84fbde2-8cdb-4c4b-84b5-87354ac7c162.png)

```xml
    <mappers>
        <package name="com.dong.dao"/>
    </mappers>
```

但使用方法三有要求：

接口和它的Mapper配置文件必须同名

接口和它的Mapper配置文件必须在同一个包下

### 生命周期和作用域

不同作用域和生命周期类别是至关重要的，因为错误的使用会导致非常严重的并发问题
**SqlSessionFactoryBuilder**

这个类可以被实例化、使用和丢弃，一旦创建了 SqlSessionFactory，就不再需要它了
**SqlSessionFactory**

可以想象为：数据库连接池

SqlSessionFactory 一旦被创建就应该在应用的运行期间一直存在，没有任何理由丢弃它或重新创建另一个实例

SqlSessionFactory 的最佳作用域是应用作用域

最简单的就是使用单例模式或者静态单例模式
**SqlSession**

连接到连接池的一个请求

SqlSession 的实例不是线程安全的，因此是不能被共享的，所以它的最佳的作用域是请求或方法作用域。

用完之后需要赶紧关闭，否则资源被占用

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667392193495-ac178d39-99bf-44d0-ac39-24e911534c48.png)

这里面的每一个Mapper,就代表一个具体的业务







### 解决属性名和字段名不一致的问题

数据库中字段名为：id，name，pwd

java实体类属性名：id，name，password

查出的结果password属性为noll

#### ResultMap结果集映射

名字任取

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667443573331-5183e4c1-bfbc-4870-b47e-7ab3f82e45eb.png)

result 中 column写表中列名  property 写

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667443765973-ec56e912-3d49-49e1-9255-7cd709fca6d0.png)



#### 日志

如果一个数据库操作出现异常，需要排错，就需要日志了

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667450739773-907a3c19-685e-43ae-8f72-c9b96e01fe69.png)

- SLF4J【要掌握】
- LOG4J（3.5.9 起废弃）
- LOG4J2
- JDK_LOGGING
- COMMONS_LOGGING
- STDOUT_LOGGING【要掌握】
- NO_LOGGING

在Mybatis中具体使用哪个日志实现，在设置中设定



### 注解查询

就不需要接口xml文件了

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667477669070-601d1541-6d4a-4701-b20b-5dcf4ae9f8f0.png)

这种，很方便，但在复杂的sql语句中难使用和维护，所以不推荐

**【关于@Param()注解】**

- 基本类型的参数或者String类型，需要加上@Param
- 引用类型不用加
- 如果只有一个进本类型的话，可以忽略，但是建议也加上
- 我们在SQL中引用的就是我们这里的@Param()中设定的属性名

### 分页

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667477998745-f46c037e-d872-4168-95b6-5c0e2f18b559.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667478005312-f9a6fabd-cc4b-4f0f-bd1b-22870a76bec0.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667478027009-d49565b1-c85b-4336-8261-8ff364638a89.png)







### 复杂查询环境搭建





#### 多对一处理

如学生和老师之间的关系

对于学生这边而言，关联，多个学生，关联一个老师【多对一】

对于老师而言 ，集合，一个老师，有很多学生【一对多】



测试环境搭建：

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667478388734-3efedbaf-921b-4ff4-897f-c79d1d37934e.png)

```sql
CREATE TABLE `teacher` (
  `id` INT(10) NOT NULL,
  `name` VARCHAR(30) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=INNODB DEFAULT CHARSET=utf8;

INSERT INTO teacher(`id`, `name`) VALUES (1, '秦老师'); 

CREATE TABLE `student` (
  `id` INT(10) NOT NULL,
  `name` VARCHAR(30) DEFAULT NULL,
  `tid` INT(10) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `fktid` (`tid`),
  CONSTRAINT `fktid` FOREIGN KEY (`tid`) REFERENCES `teacher` (`id`)
) ENGINE=INNODB DEFAULT CHARSET=utf8;
INSERT INTO `student` (`id`, `name`, `tid`) VALUES (1, '小明', 1); 
INSERT INTO `student` (`id`, `name`, `tid`) VALUES (2, '小红', 1); 
INSERT INTO `student` (`id`, `name`, `tid`) VALUES (3, '小张', 1); 
INSERT INTO `student` (`id`, `name`, `tid`) VALUES (4, '小李', 1); 
INSERT INTO `student` (`id`, `name`, `tid`) VALUES (5, '小王', 1);
```

#### 按照查询嵌套处理

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667480187279-02a72153-16b8-403c-b167-da3f8e5f0f48.png)

由于学生里的结果存在老师这个特殊的对象而不是字段，所以需要单独处理

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667547778291-b4fde62b-ce8a-4060-9190-6edf313667f5.png)

property：注入给实体类的某一个属性

column：在上次查询结果集中，用哪些列值作为条件去执行下一跳SQL语句

javaType：把sql语句查询出的结果集，封装给某个类的对象

select：下一条要执行的sql语句



#### 按照结果嵌套查询

查询sql写完后只要对应里面的每一项的关系

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667552034225-f75a7130-d833-45eb-b0b0-8f817242671e.png)

### 一对多查询



### 动态sql

动态sql就是根据不同的条件生成不同的sql语句

所谓的动态SQL，本质还是SQL语句，只是我们可以在SQL层面，去执行一个逻辑代码。

动态sql就是在拼接sql语句，我们只要保证sql的正确性，按照sql的格式，去排列组合就可以了

建议：

先在mysql中写出完整的sql，在对应的去修改成为我们的动态sql实现通用即可

if where set choose when foreach





<where>标签：where元素只会在至少有一个子元素的条件返回SQL子句的情况下才去插入where字句。而且，若语句的开头为 and 或 or ，where标签也会将它们去除

<set>标签：会动态前置set关键字，同时也会删除无关的逗号

<trim>定制化

<set>标签的真相：<trim prefix="SET" suffixOverrides=",">

<where>标签的真相：<trim prefix="WHERE" prefixOverride="AND | OR">

#### if语句

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667736263912-b85af9d3-aaf5-4dc3-9dd0-577872562cdf.png)



#### choose(when,otherwise)

有时候，我们不想使用所有的条件，而只是想从多个条件中选择一个使用。针对这种情况，MyBatis 提供了 choose 元素，它有点像 Java 中的 switch 语句。

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667736328505-3fbe2437-bf31-4198-9389-72f0bb977a49.png)



#### Foreach

允许你指定一个集合，声明可以在元素体内使用的集合项（item）和索引（index）变量。它也允许你指定开头与结尾的字符串以及集合项迭代之间的分隔符。这个元素也不会错误地添加多余的分隔符  

可以将任何可迭代对象（如 List、Set 等）、Map 对象或者数组对象作为集合参数传递给 foreach。当使用可迭代对象或者数组时，index 是当前迭代的序号，item 的值是本次迭代获取到的元素。当使用 Map 对象（或者 Map.Entry 对象的集合）时，index 是键，item 是值。  

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667820006566-6a7857ec-8099-4a1c-a6e2-0fe20de6df82.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667825143826-20a4722c-104d-48aa-abd7-da43c5ccfbfb.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667825299508-fc6690f5-5e76-442a-89b0-ddfa70f23993.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667825313617-c8c55e4d-fa0f-45b3-bffe-25a1476137c8.png)

#### SQL片段

有时候我们可能会将一些功能的部分抽取出来，方便复用

使用sql标签抽取公共的部分

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667785636987-3af944b0-bea7-4355-84fc-bd169ea6afdf.png)

在需要使用的地方使用lnclude标签引用

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667785651229-861cd24a-4cc6-44e7-b6c1-a3cf5240c3bd.png)

注意事项：

最好基于单表来定义sql片段，不要用太复杂的语句

不要存在where标签,一般SQL片段里放的最多的就是if判断  



### 缓存

每次查询都要连接数据库非常耗资源，所以可以把一次查询的结果，暂存在一个可以直接取到的地方

存到内存里，叫缓存；当再次查询相同的数据的时候，直接走缓存就不用连接数据库了

经常查询并且不经常改变的数据。可以使用缓存



一级缓存默认是开启的，只在一次SqlSession中有效，也就是拿到连接到关闭连接这个区间内。

二级缓存在同一个mapper下就有效



开启全局缓存



开启二级缓存

<cache/>在当前Mapper.xml中开启二级缓存。也可以自定义参数

<cache eviction="FIFO" flushInterval="60000" size="512" readonly="true">



所有数据都会先放在一级缓存中只有当会话提交或者关闭，才会提交到二级缓存中



#### 自定义缓存-ehcache







#### 遇到的错误：

org.apache.ibatis.binding.BindingException: Type interface com.dong.dao.UserDao is not known to the MapperRegistry.

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667350977994-d047923a-c560-49f7-8c12-0cedeab00398.png)

每一个Mapper.XML都需要在Mybatis核心配置文件中注册

mybatis-config.xml

<mappers>

<mapper resource=""/>

</mappers>



![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667719377008-1c39cdb7-4147-4a66-b2f8-673037bd82bf.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667719402806-9399c9ab-7eb2-4e70-8eff-f99d7a9673d0.png)

![img](https://cdn.nlark.com/yuque/0/2022/png/21897286/1667719414083-cab0e41f-d80a-40df-925c-1eb03083d1c6.png)
