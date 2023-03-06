## 前言

官方文档：php.net
![image.png](https://static.sechelper.com/img/2022/11/22/4c5b04593d276630d656b309748c2635.png)
php官方文档是非常详情，好用的，在遇到不清楚作用的函数时可以进行查询

白盒测试做代码审计最主要的知识是要去了解一个漏洞应该有哪些防御方式，因为大部分的漏洞都是因为修复没有做的全面，或者修复没有考虑到一些情况导致漏洞。
MVC：
C：分发处理请求网站的逻辑
M：处理和数据库相关的操作
V：显示给用户的内容

## 代码审计流程
### 正向查找流程
a. 从入口点函数出发（如index.php）
b. 找到控制器，理解URL派发规则（URL具体映射到哪个具体的代码里）
c. 跟踪控制器调用，以理解代码为目标进行源代码阅读
d. 最终在阅读代码的过程和尝试中，可能发现漏洞

本质：程序员疏忽或逻辑问题导致漏洞 

特点：

1. 复杂：需要极其了解目标源码的功能与框架
2. 跳跃性大：涉及M/V/C/Service/Dao等多个层面
3. 漏洞的组合：通常是多个漏洞的组合，很可能存在逻辑相关的漏洞

### 反向查找流程
a. 通过危险函数，回溯可能存在的漏洞
 	1. 查找可控变量
 	2. 传递的过程中触发漏洞

特点：

1. 与上下文无关
2. 危险函数，调用即漏洞

代码审计工具功能大多就是这个原理
### 双向查找流程（手动审计主要方式）

1. 略读代码，了解框架（正向流程，如：网站都有哪些功能，什么样的架构如mvc：它的m在哪v,c在哪，用了什么模板引擎，是否用了orm（如果使用了ORM那么sql注入就很少了，如果没用是手工写的sql语句，可以关注是否存在sql漏洞）等...)
2. 是否有全局过滤机制
   1. 有：是否可以绕过
      1. 可以：寻找漏洞触发点（反向查找流程，找危险函数）
      2. 不可以：寻找没有过滤的变量
   2. 没有：那么就看它具体是如何处理的,具体代码具体分析
      1. 有处理：寻找遗漏的处理点（如忘记处理的地方或者处理不太正确的地方）
      2. 完全没有处理：可以挖成筛子（很少）

3.找到了漏洞点，漏洞利用是否有坑

根源：理解程序执行过程，找寻危险逻辑
特点：
	高效：如挖隧道，双向开工，时间减半（不需要去完全理解网站内部原理和函数作用)
	知识面广：需要同时掌握正向，反向挖掘技巧，并进行结合
	以及所有正向，反向的优点

## SQL注入漏洞挖掘技巧
PHP+mysql链接方式有：
mysql（废弃，但老的仍然有）
mysqli
PDO

### sql注入常见过滤方法
**intval：**							把用户输入的数字后面的所有不是数字的都过滤掉
**addslashes：**				  把 ' 前加\转义掉
**mysql_real_escape：**	和第二个类似，但会考虑用户输入和mysql的编码，避免像宽字节注入问题

**mysqli_escape_string** / **mysqli_real_escape_string** / **mysqli::escape_string**  （和mysqli搭配使用，和前面的功能类似）和他们的差别是会主动加引号包裹

**PDO:**	quote

**参数化查询**

### 常见注入过滤绕过方法
intval：不知道
addslashes  /  mysql_real_escape
	1.宽字节注入
	2.数字型sql语句
	3.寻找字符串转换函数（传入编码好的字符绕过过滤，在后面被转换成sql语句）
		urldecode
		base64_decode
		iconv
		json_decode
		stripshasles
		simple_xml_loadstring
例如：传入id被过滤但后面有一处代码是解码base64，所以我们可以传入 ' 的base64编码绕过

```php
<?php
$id = addslashes($_GET['id']);
....
$id = base64_decode($id);
....
$sql = "select * from flag where id = '$id'";   
?>
```

mysqli::escape_string  //  PDO::quote
	1.宽字节注入
参数化查询
	1.寻找非sql值的位置

### 开发者容易遗漏得输入点

1. HTTP头

   a. X-Forwarded-For

   b. User-Agent

   c. Referer

2. PHP_SELF（访问的页面url名，但用户可控）

3. REQUEST_URI（用户请求得完整路径）

4. 文件名 $_FILES[][name]

5. php://input   (post穿进去得内容)

   

引入单引号(转义符)的方法（'号被过滤，看后面有没有可以引入的地方）
	stripslashes
	base64_decode
	urldecode
	substr
	iconv
	str_replace('0','',$sql)
	xml
	json_encode

## 任意文件操作
PHP上传的文件会被保存在$_FILES下

PHP文件操作函数汇总

1. 文件包含
   1. include/require/include_once/require_once/spl_autoload
2. 文件读取
   1. file_get_contents/fread/readfile/file/ highlight_file/show_source
3. 文件写入
   1. file_put_contents/fwrite/mkdir/fputs
4. 文件删除
   1. unlink/rmdir
5. 文件上传
   1. move_uploaded_file/copy/rename
### 文件上传漏洞
文件上传流程：

1. 检查文件大小，后缀，类型

2. 检查文件内容（如文件头，尾等）

3. 提取文件后缀

4. 生成新文件名

5. 将上传临时文件拷贝到新文件名位置

   

文件上传逻辑常见错误

1. 只检查文件类型不检查文件后缀
2. 文件后缀黑名单有遗漏
3. 使用原始文件名，导致\0截断（一般没有了）
4. 前端检验
### 文件包含漏洞
首先明确一点，文件包含漏洞不等于文件读取漏洞

危害：文件读取  /  代码执行
常见位置：模板文件名（切换模板）
    				语言文件名（切换语言）
常见利用：
	要寻找可被包含利用的文件：上传文件，临时文件，Session文件，日志文件
	后缀无法控制的情况：\0截断，协议利用
	PHP5.3.4+ 对包含\0的文件操作函数进行了限制，基本上没有了

#### 案例
Metinfo 5.3.10版本Getshell漏洞
可控制的部分：include $file . '.php';
http协议利用：http://xxxx.com/1.php (远程文件包含，一般不开设置不能用)
PHP协议利用：zip/phar
	制作2.php的压缩包 -> 2.zip -> 改后缀为 -> 2.jpg
	在服务器中上传2.jpg文件
	再利用：zip://var/www/upload/head/2.jpg#2.php （#意思是访问zip内部的子文件->2.php）

### 文件删除漏洞
危害：
	删除服务器任意文件，DOS服务器
	删除安装锁文件，导致目标环境可被重新安装
	重新安装  -》任意重置管理员密码

#### 案例
上传新头像会把老头像自动删除，但可以把删除老图像的地址换成别的造成任意文件删除

## 命令执行
命令执行指的是执行系统命令 （ls）
https://explainshell.com/可以在这个网站查询复杂命令的意思
代码执行指的是PHP的代码执行
本质：用户输入无过滤，拼接到了系统命令中
PHP命令执行函数：

1. system
2. passthru
3. exec
4. shell_exec
5. popen  (常见的就是上面这5种)
6. proc_open
7. pcntl_exec
8. dl

要像命令执行很难，要先学会如何正确防御命令注入，才能分辨出哪些没有正确处理

防御PHP命令注入漏洞：
PHP中只能使用escapeshellcmd和escapeshellatg进行命令参数的过滤

1. 先区分这2个函数：

​		**escapeshellcmd**
​		**escapeshellatg**
​				escapeshellatg没问题但escapeshellatg是只能限制逃逸不出单引号'但有些命令的不常用参数是可以任意命令执行 

2. 要把用户的输入放在值里

​	如果把输入直接是键值对可能会造成漏洞

```sql
grep {$query}
```

有一些命令的不常用参数可能会导致一些意外发生。直接使用 | 等命令跳出前面的命令实现命令执行
​	修复：

```
grep -i {$query}
grep -- {$query}
```

## XML实体注入漏洞

PHP XML解析函数
	simplexml_load_file
	simplexml_load_string
	SimpleXMLElement
	DOMDocument
	xml_parse
如果发现有这几个函数的地方，基本上可以确定百分之80有xml实体注入 

libxml_disable_entity_loader(true)来禁用掉外部实体的加载，就不存在xml实体注入

PHP中XXE漏洞逐渐减少，到现在的版本里几乎已经绝迹了，因为PHP XML操作依赖libxml库
但在libxml2.9.0+默认是关闭了xml外部实体解析开关的，可以顺势挖一下也比较简单



方法：暴力搜索就行，查看有没有xml解析函数，再看禁没禁止外部加载

无输出点的xxe漏洞：有时候可能存在xxe漏洞但并不会在页面中显示，要利用到blind-xxe
Blind XXE原理：
	利用XML外部实体功能读取文件
	利用XML外部实体功能发送HTTP请求
	利用HTTP协议传递文件内容

## 前端漏洞
建议代码审计不去主要找这种漏洞，进行黑盒测试就能挖到的漏洞，在代码审计过程中没必要太注重

百盒测试中可以关注前端漏洞类型：
	XSS漏洞
	CSRF漏洞
	Jsonp劫持漏洞（前面三个关注多）
	URL跳转漏洞（不多）
	点击劫持漏洞（不多）

### xss
在白盒测试中寻找XSS漏洞：
常见防护方法：
	htmlspecialchars()把用户输入转义成html实体字符（这时候是绝对没有xss的）
	strip_tags()	从字符中去除HTML和PHP标记

自动化FUZZ -》寻找输出函数（危险函数）
富文本XSS挖掘
	什么是富文本：本质就是html，网站给你一个有很多功能的输入框

![image.png](https://static.sechelper.com/img/2022/11/22/773b4680008c6372db041770fbff4669.png)
为什么出现富文本xss：
	前面2种防护方法要么就是转义掉要么就是直接去除掉，但在一些写文章，或者发帖的需要提交html富文本，如果使用前面的方法那么提交的就不是富文本了，会影响业务。
常见富文本过滤方法：
	会使用富文本的xss过滤器：把用户输入的恶意标签，属性去掉
	黑名单（很难过滤掉该过滤的标签属性)
	白名单

### CSRF
在白盒测试中寻找CSRF漏洞：
	检查Referer（来自于当前域名，可信域名，才会执行）
	（可以看正则匹配全面吗，比如正则匹配xxx.nte后缀的域名，那么可以注册个cxxx.nte域名绕过）
	检查Token
	（寻找跨域漏洞，要去跨域请求某一个网站内容的时候需要先去请求这个网站有没有crossdomain.xml，
	要根据这里面配置的信息来认证是否允许用户去发送一个跨站的请求）
		Flash
		Jsonp
		CORS   黑盒测试就可以找到这三个

### Jsonp劫持漏洞
在白盒测试中寻找Jsonp劫持漏洞：
Jsonp介绍：Jsonp是 json 的一种"使用模式"   可以让网页从别的域名（网站）那获取资料，即跨域读取数据；
它利用的是script标签的 src 属性不受同源策略影响的特性，使网页可以得到从其他来源动态产生的 json 数据，因此可以用来实现跨域读取数据。  
更通俗的说法：JSONP 就是利用 <script> 标签的跨域能力实现跨域数据的访问，请求动态生成的 JavaScript 脚本同时带一个 callback 函数名作为参数。服务端收到请求后，动态生成脚本产生数据，并在代码中以产生的数据为参数调用 callback 函数。  



原理：当网站通过 JSONP 方式传递用户敏感信息时，攻击者可以伪造 JSONP 调用页面，诱导被攻击者访问来	达到窃取用户信息的目的；jsonp 数据劫持就是攻击者获取了本应该传给网站其他接口的数据。  
 和 CSRF 类似，都需要用户交互，而 CSRF 主要是以用户的账户进行增删改的操作，jsonp 则主要用来劫持数据。  

Jsonp借此漏洞常见位置：
	web框架默认支持ajax + jsonp方式请求
	程序员主动开发需要支持jsonp的应用
Jsonp劫持防御：

1. Referer检查
2. Toke

## 反序列化漏洞
几乎所有语言都有序列化功能，java，php，python都有相关漏洞
反序列化分类：
	反序列化触发执行任意代码 -》python
	(python的反序列化其实是一门真正的语言，是可以直接进行执行代码的)
	反序列化后，通过已有代码利用链，间接执行任意代码 -》 PHP/java

PHP反序列化注意函数：
	serialize（序列化函数）
	unserialize（反序列化函数）
PHP反序列化特点：
	引入除资源型外任意类型变量
	无法引入函数 -》不能直接执行代码
	迂回战术
		寻找程序中可能存在的漏洞的类
		实例化类对象 -》 触发漏洞

漏洞挖掘过程：
	寻找调用反序列化函数的位置
	寻找包含危险方法的类
	反序列化上下文是否包含该类
		包含：直接生成该类，触发漏洞
		不包含：寻找引用链
怎么利用链，怎么构造利用语句，还有phar反序列化等有很多后面再写
如果只是要找反序列化漏洞，那么找unserialize就够了

## 小技巧篇
代码审计明白了原理，明白了各种漏洞的修复方式，之后想要提高就要仰仗自己积累的一些小技巧
因为你会发现遇到的大部分有漏洞的代码前面都存在一些过滤，检查；
如果你知道一些技巧后会发现很多过滤，检查都是可以绕过的

### web开发框架下的PHP漏洞
Laravel
symfony
slimphp
Yii2
特点： 
	所需PHP版本较高，\0截断等老漏洞绝迹
	提供功能强大的ORM（即使想写出一个sql漏洞都难）
	提供自动处理输出的模板引擎（想写出前端漏洞都难了，因为自带一些转义，实体化功能）
	开发者可以通过composer找到任何需要的功能类，避免因为自己造轮子产生的漏洞
现代web开发框架安全思想
	Secure by default 原则
	文档中，会详细叙述可能出现的安全问题
挖掘思路
	寻找框架本身的安全漏洞
	寻找不规范的开发方式
	寻找错误的配置（debug模式，日志记录等）
	异常的利用（如果开启了debug或者会输出异常，可以构造异常抛出敏感信息，新的框架特有的漏洞老的几乎没有）
第三方服务利用
	spl_autoload的利用

### 压缩包问题
web应用执行了解压缩操作
黑客利用压缩包的一些特性，构造"畸形"压缩包
解压缩时将造成漏洞 

压缩炸弹惯用的方式 
	绕过文件检查失败后的删除操作
	阻止压缩时的文件检查
	绕过压缩时的文件检查
	链接文件的利用
绕过文件检查失败后的删除操作：
	案例：上传压缩包后，后端处理只能删除文件无法删除目录，导致shell

阻止压缩时的文件检查
	案例：压缩包解压失败，程序抛出错误并停止运行 -》 webshell保留
    压缩包三个文件，一个正常图片，一个webshell，第三个文件压缩存在错误
绕过压缩时的文件检查
	案例：使用../解压到上层目录，；构造一个文件名为：../../webshell.php，会让后端解压到上层目录
 			   而删除文件一般是在当前目录递归删除非法文件，不可能在根目录递归
链接文件的利用
	后端解压后未判断文件类型，导致可以上传软链接文件，该软连接导致任意文件读取
	压缩包是允许压缩软连接文件，也运行解压软连接文件；但文件上传传不了

### 条件竞争漏洞
条件竞争：web服务器都是多线程的，同时运行多人访问网站，理论是互相不影响的，但是php，Apache，Nginx只能保证php是不互相影响的，不能保证文件或者数据库链接是不互相影响的

条件竞争漏洞挖掘方向：
	上传后删除的利用
	忘记上锁的数据库
	鸡肋文件包含的妙用
上传后删除的利用：
	上传压缩包后解压递归删除非法文件，但在这个过程中开启多个线程去访问解压出来的webshell，并在上层目录写入新的webshell；打一个时间差，在还没删除时利用，把webshell解压出来后面还有其他文件
发现一个文件上传的逻辑是在上传了后再删除，基本上就确定存在条件竞争漏洞

### 没上锁的数据库
案例：商场漏洞：
	1.查询用户余额
	2.查询购买商品的价格
	3.判断用户余额>商品价格
	4.用户余额=用户余额 - 价格
		如果第三步和第四步是单独的步骤，如果用多个线程去请求同时走到了第三步，判断都可以购买
		购买完后同时走到第四步后同时减去价格，就会导致余额变负 -》成功购买多个商品

### 临时文件包含利用
​	文件包含漏洞需要找到一个能够包含的恶意文件，但网站没有能够上传文件的地方，也没有找到任何可以控制的文件

​	寻找临时文件泄露点，文件上传的时候，用户会发送一个数据包给服务器，服务器会将数据包里的文件保存到当前的临时目录下，变成	临时文件，文件名随机，内容可以控制，phpinfo可以获取文件名
​	我们可以上传一个非常大的文件，需要10秒上传完成，临时文件上传完成后是会被删除的，再开多个线程去包含该文件，生成一个新的webshell  

# 一些容易犯错的点
header('404') 会给用户返回一个页面，但并不会阻止php解释器继续往下运行

正则：应该要写判断只有的，错误写成包含有，那么就有漏洞
```python
if(!preg_match('/liang|ban/i', $cmd)){
	exit("错误")；
}
cmd = union select 1,2,3 # liang
cmd = ls / | ban
```

它会查询传入的语句其实有DESC就绕过了但和DESC一块的有我们的其他命令

# 尝试审计代码
admin 后台管理目录
install 网站的安装目录
api 接口文件目录
data 系统处理数据相关目录
include 用来包含的全局文件
template 模板  
css CSS样式表目录
images 系统图片存放目录
system  管理目录
函数集文件: 这类文件通常命名中包含functions或者common等关键字，这些文件里面是一些公共的函数，提供给其他文件统一调用，所以大多数文件都会在文件头部包含到它们，寻找这些文件一个非常好用的技巧就是去打开index.php或者一些功能性文件，在头部一般都能找到。  
配置文件：这类文件通常命名里面包括config这个关键字，配置文件包括Web程序运行必须的功能性配置选项以及数据库等配置信息，从这个文件里面可以了解程序的小部分功能，另外看这个文件的时候注意观察配置文件中参数值是用单引号还是用的双引号包起来，如果是双引号，则很大可能会存在代码执行漏洞。  
[https://github.com/source-trace/bluecms](https://github.com/source-trace/bluecms)
![image.png](https://static.sechelper.com/img/2022/11/22/cdf2b976d9deec5d9c5791dfd06f48d6.png)
根据流程填完信息后又空白了，但没问题已经安装好了，访问index.php页面就好
![image.png](https://static.sechelper.com/img/2022/11/22/b6a38ff07e2f9a91202ee886c2df5626.png)
![image.png](https://static.sechelper.com/img/2022/11/22/c3a20d6a06a18a4ad0488d2749c563eb.png)
使用Seay工具自动扫描一下
![image.png](https://static.sechelper.com/img/2022/11/22/e57dff2a97d29c0cf63a015c235bb290.png)
先看第一个试试
![image.png](https://static.sechelper.com/img/2022/11/22/6a221b9e530f29b079434bbe99ce2c5f.png)
可以看到先是包含了一个common.inc.php的文件，其次如果有ad_id参数那么会经过trim函数的处理
再sql语句拼接时是没有单引号包裹的，可能存在问题
去看看包含的php文件。粗略的观察一下发现有对于$_GET进行处理
![image.png](https://static.sechelper.com/img/2022/11/22/073b8cd2a93569b53610374f79059c11.png)
if(!get_magic_quotes_gpc())，查看下函数意思
![image.png](https://static.sechelper.com/img/2022/11/22/27054eb67d00b559733392f270fc24bb.png)
就是一直是true，都会经过这个if语句里的处理；if语句的处理追踪一下deep_addslashes看看
![image.png](https://static.sechelper.com/img/2022/11/22/d716b2887e0cb09b6259a9b6af9a124d.png)
就是经过了addslashes的处理，再前面学习过这个函数就是对于单引号双引号反斜线和nul进行转义，但我们sql语句并没有被''单引号包裹，这个过滤对于要审计的sql语句没有用；希望+1
看完包含的函数后在看一下trim函数有没有问题基本上就确定有没有sql注入了
么有定位到这个函数，查下官方手册发现是用来去除字符串首尾处的空白字符（或者其他字符）\t \n \r \0 \x0B
但只是去除首位的，语句中间的并不会，所以没影响；好理论确定存在注入了，实践开始
没有过滤扫描sql语句或者字母数字什么的，所以直接使用order查看下字段数
![image.png](https://static.sechelper.com/img/2022/11/22/b797eff7c8bd77aa6be901b48c39a22a.png)
可以看到查到8时报错并有返回说明确实存在注入，使用联合注入试下
![image.png](https://static.sechelper.com/img/2022/11/22/c526c0a9d94a32dbe28b4af6e96b7cec.png)
它是无回显的，可能要使用到时间盲注了，但在查看到网页源代码时发现![image.png](https://static.sechelper.com/img/2022/11/22/505e89bd78b1cc95e8ed884b7dc431de.png)不确定在试试![image.png](https://static.sechelper.com/img/2022/11/22/5fdc083edaa3c242f56ddd8fe4e6a339.png)
发现是有回显的，位数在7，在尝试查下表，嗯看来什么都可以查到
![image.png](https://static.sechelper.com/img/2022/11/22/80e6832e66d84abbcf48c8a2c7945742.png)
没有单引号，尝试下xss注入；也是存在的
![image.png](https://static.sechelper.com/img/2022/11/22/4c56a4854a0ee3300f100eeaf3ecf066.png)
