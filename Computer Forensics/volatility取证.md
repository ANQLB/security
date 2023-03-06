### 前言

volatility是一款开源的内存取证分析工具，由python编写，支持各种操作系统，能够对导出的windows,linux,mac osx,android等系统内存镜像进行分析。可以通过插件来拓展功能。

### 常见命令

命令格式：

volatility -f [镜像文件] --profile=[操作系统] [插件参数]

```plain
volatility -f 文件名 imageinfo  		     	 得到镜像的基本信息。
volatility -f 文件名 --profile=系统 pslist		查看进程信息
volatility -f 文件名 --profile=系统 pstree	    查看进程树
volatility -f 文件名 --profile=系统 hashdump		查看用户名密码信息
volatility -f 文件名 --profile=系统 john			爆破密码
volatility -f 文件名 --profile=系统 lsadump		查看用户强密码
volatility -f 文件名 --profile=系统 svcscan		查看服务
volatility -f 文件名 --profile=系统 iehistory	查看IE浏览器历史记录
volatility -f 文件名 --profile=系统 netscan		查看网络连接
volatility -f 文件名 --profile=系统 cmdscan		cmd历史命令
volatility -f 文件名 --profile=系统 consoles		命令历史记录
volatility -f 文件名 --profile=系统 cmdline		查看cmd输出， 获取命令行下运行的程序   
volatility -f 文件名 --profile=系统 envars 		查看环境变量，一般很多配合grep筛选，可也是使用-p指定pid
volatility -f 文件名 --profile=系统 filescan		查看文件
volatility -f 文件名 --profile=系统 notepad		查看当前展示的notepad内容
volatility -f 文件名 --profile=系统 hivelist		查看注册表配置单元
volatility -f 文件名 --profile=系统 userassist	查看运行程序相关的记录，比如最后一次更新时间，运行过的次数等。
volatility -f 文件名 --profile=系统 clipboard  	查看剪贴板的信息
volatility -f 文件名 --profile=系统 timeliner	最大程序提取信息
volatility -f 文件名 --profile=系统 Dumpregistry	提取日志文件
volatility -f 文件名 --profile=系统 dlllist		进程相关的dll文件列表 
volatility -f 文件名 --profile=系统 memdump -p xxx --dump-dir=./ 	提取进程
volatility -f 文件名 --profile=系统 dumpfiles -Q 0xxxxxxxx -D ./ 	提取文件
volatility -f 文件名 --profile=系统 procdump -p pid -D ./			转存可执行程序  
volatility -f 文件名 --profile=系统 screenshot --dump-dir=./			屏幕截图
volatility -f 文件名 --profile=系统 hivedump -o 0xfffff8a001032410 查看注册表键名
volatility -f 文件名 --profile=系统 printkey -K "xxxxxxx"			查看注册表键值
```

### OtterCTF取证11题

在分析之前，都需先查看当前镜像的信息，获取是哪个操作系统，使用imageinfo命令查看

![image-20221126101043895](https://static.sechelper.com/img/2022/11/26/7ed786d52f16b44d8d7d03b03c5ed059.png)

然后使用volatility各类工具进行分析即可



#### 获取密码

volatility -f OtterCTF.vmem --profile=Win7SP1x64 hashdump	查看用户名密码信息

volatility -f OtterCTF.vmem --profile=Win7SP1x64 lsadump		查看用户强密码

使用john爆破密码不出密码尝试lsadump查看用户强密码

lsadump：从注册表中提取LSA密钥信息,显示加密以后的数据用户密码

![img](https://static.sechelper.com/img/2022/11/26/3b9f336be6bdcbbec4f185f817ec61d6.png)

#### pc的名称

查看主机名

hivelist查看注册表信息，查看到system

所有用户信息都会存储到注册表，SYSTEM系统信息  

 hivelist查看注册表第一级信息  

第一级只是目录，路径代表文件名

![img](https://static.sechelper.com/img/2022/11/26/1211edbbb2e9c0c93508b068720437b9.png)

如果东西不多可以 hivedump  下来查看，如果很多可以用printkey一步步查看

所以文件的位置使用偏移量来表示，0x开头，使用 printkey打印出来，参数-o，然后根据得到的偏移量，找到系统注册表包含的值  

volatility -f OtterCTF.vmem --profile=Win7SP1x64 printkey -o 0xfffff8a000024010

![img](https://static.sechelper.com/img/2022/11/26/3dfcbd23e4bd80ba6e2bf5a6703b954a.png)

继续寻找直到找到ComputerName关键词，后面的路径要使用 -K 参数使用一步步来，深入路径

![img](https://static.sechelper.com/img/2022/11/26/bebe61de8a6cfbef30883ee6e01ef107.png)

![img](https://static.sechelper.com/img/2022/11/26/994d55a1f564b7408070cadca5a9e47f.png)

 含有目录两层\ComputerName\ComputerName，增加混淆  

![img](https://static.sechelper.com/img/2022/11/26/251760b82eca60fe8bb5928a2da2ec83.png)

#### 内存正在运行什么游戏，游戏连接哪个服务器

游戏应该会连接服务器，所以查看网络

寻找可疑进程，连接外部网络的进程  

volatility -f OtterCTF.vmem --profile=Win7SP1x64 netscan		 对所有网络连接进程扫描  

![img](https://static.sechelper.com/img/2022/11/26/e9bf84cfd16d1454cb8c632a37c7d8a6.png)



#### 账户登录过Lunar-3频道，账户名是什么

寻找登录游戏频道的游戏账户名。

需要在游戏进程中查看，用户登录到进程中的话，那么内存应该有登录用户名

使用strings过滤可打印字符串，grep过滤含有关键字Lunar-3频道字符串

strings OtterCTF.vmem | grep Lunar-3 -A 5 -B 5

（-A 查看关键词前几行，-B查看关键词后几行，也可以使用-C查看前后几行）

![img](https://static.sechelper.com/img/2022/11/26/845cf869a69a701c895eac0d6f106259.png)

都尝试一下发现是0tt3r8r33z3

strings命令在对象文件或二进制文件中查找可打印的字符串。

#### 寻找登录名在0x64 0x??{6-8} 0x40 0x06 0x??{18} 0x5a 0x0c 0x00{2} 后的游戏名

 意思是用户名总在这个签名之后：0x64 0x??{6-8} 0x40 0x06 0x??{18} 0x5a 0x0c 0x00{2} 

先将LunarMS.exe进程转存出来；-p pid号

volatility -f OtterCTF.vmem --profile=Win7SP1x64 memdump -p 708 -D ./ 	提取进程

可以使用010查看 5A 0C 00 片段

也可以使用：hexdump -C 708.dmp |grep "5a 0c 00" -C 3 

![img](https://static.sechelper.com/img/2022/11/26/94dc5b0825e463bda0be815713d6aa19.png)

寻找5a 0c 00 片段眼都快找花了，太多了



volatility -f OtterCTF.vmem --profile=Win7SP1x64 yarascan -Y "/\x64(.{6,8})\x40\x06(.{18})\x5a\x0c\x00\x00/i" -p 708

这个需要安装yarascan插件是最好找的



#### 寻找经常复制粘贴的账号密码

volatility -f OtterCTF.vmem --profile=Win7SP1x64 clipboard 

clipboard  查看剪贴板的信息

![img](https://static.sechelper.com/img/2022/11/26/3a1ab76ef529e9013644365d926302c0.png)



#### 寻找恶意软件名

一般是使用pslist来查看进程寻找可疑进程名，但也有可能进程名进行了混淆分辨不出来或者被合法进程隐藏了

使用 pstree 命令可以查看进程树，可以查看所有进程和依赖关系

pstree代表查看带树结构的进程列表。

寻找可疑进程：通过寻找PPID大于PID的进程，或查看进程依赖寻找可疑的

![img](https://static.sechelper.com/img/2022/11/26/93a6362ce9e0dec32027bbacd84bd259.png)

这里mvware-tray.exe是ppid大于pid的并且很奇怪的是Rick And Morty的子进程  

dlllist代表查看使用的动态链接库是否合法， 查看一下进程相关的dll文件列表  。-p指定pid号

![img](https://static.sechelper.com/img/2022/11/26/ec78290c5074210669031794f30d469f.png)

发现是在temp目录下进行的，一看就不是正经程序， 因为temp这是一个临时目录  



#### 恶意软件是如何进入电脑的

可以查看和恶意软件相关的文件

volatility -f 1.vmem --profile=Win7SP1x64 filescan	查看文件

直接查看文件太多，要加过滤，恶意进程的父进程是Rick And Morty

volatility -f 1.vmem --profile=Win7SP1x64 filescan | grep 'Rick And Morty'

![img](https://static.sechelper.com/img/2022/11/26/d2f7b04b65dd34757f92ad111b7ffb2f.png)

三个种子文件和三个exe文件；寻找来源要关注种子文件，里面可能含有地址信息

volatility -f 1.vmem --profile=Win7SP1x64 dumpfiles -Q 0xxxxxxxx -D ./ 	查看文件内容

![img](https://static.sechelper.com/img/2022/11/26/d8b05d8be57e03af0af939db9cf79db7.png)

在这个文件里发现 website可疑，后面就是flag



#### 恶意软件种子从哪来

查看进程可以发现有很多chrome浏览器进程，种子可能是在浏览器中下载的

先把所有chrome进程转存下来

memdump -n chrome(指定所有chrome进程) -D ./

再查找 download\.exe\.torrent

![img](https://static.sechelper.com/img/2022/11/26/fe8bb6a4b356293d7c158d25b15caa33.png)

#### 通过恶意软件寻找攻击者的比特币地址

题目描述说攻击者在恶意勒索软件中留下了比特币地址，是多少呢？  

首先可以把恶意进程转存到一个可执行文件，使用IDA查看寻找比特币，钱包，支付等关键词，定位地点

也可以使用dnSpy进行逆向分析  

也可以把转出的文件使用strings -e l 进行搜索

strings -e l 3720.dmp | grep -i -A 5 "ransomware"  

![img](https://static.sechelper.com/img/2022/11/26/4374ac24d51e7ddd6445b4ad226345c1.png)



#### 恶意软件的图像的隐藏信息

procdump  命令代表转存可执行程序  

![img](https://static.sechelper.com/img/2022/11/26/62728d4cafc7610252ef0066464dbcea.png)

使用foremost分离图片

![img](https://static.sechelper.com/img/2022/11/26/1f46b9ec8a15824590ee3818866b13ec.png)

![img](https://static.sechelper.com/img/2022/11/26/19ff35dbd3b090b2b8895bafe35629cc.png)

或者使用反编译查看图片
