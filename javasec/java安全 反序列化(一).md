## 前言

序列化就是把对象转换成字节流，便于保存在内存、文件、数据库中;反序列化即逆过程，由字节流还原成对象。序列化是一种对象持久化的手段，可以将对象的状态转换为字节数组，来便于存储或者传输的机制;可以有效地实现多平台之间的通信、对象持久化存储。

Java中的`ObjectOutputStream`类的`writeObject()`方法可以实现序列化，类`ObjectlnputStream`类的`readObject()`方法用于反序列化。

就像游戏的存档，中途退出后存档，再次游玩时读取存档恢复上次游戏离开时的状态



## 序列化基础知识：

一个类对象要想实现序列化，必须满足两个条件：

​	该类的所有属性必须是可序列化的。

​	需要实现Serializable或Externalizable接口

实现其中一个接口就可以了

java.io.Serializable

java.io.Externalizable



### java.io.Serializable

public interface Serializable {}

这个是标记接口里面什么内容都没有，本身是没有意思的。编译器知道这个标记有什么含义，对实现了这个接口的类会进行特殊处理。

实现了这个接口的类，编译器就知道这个对象是可以用来序列化



### java.io.Externalizable

```java
public interface Externalizable extends java.io.Serializable{
    void writeExternal(ObjectOutput out) throws IOException;
    void readExternal(ObjectInput in) throws IOException,ClassNotFoundException;}
```

Externalizable接口也是实现了Serializable接口，并且有2个方法，要继承这个接口必须要实现接口定义的方法。



## 尝试序列化和反序列化

对一个类进行序列化需要执行ObjectOutputStream.writeObject方法写入对象。

对一个类进行反序列化需要ObjectIputStream.readObject从输入流中读取字节然后转换成对象。

在反序列化的过程中，是直接拿到对象而不是new一个所以被反序列化操作的类不会执行构造方法

注意看注解

TestSerialize

```java
import java.io.Serializable;

public class TestSerialize implements Serializable {
    private static final long serialVersionUID = 1;
    public String username;
    //被transient关键字修饰的成员属性变量不被序列化
    transient private String password;
	
    public TestSerialize(String name, String pass) {
        this.username = name;
        this.password = pass;
    }
    public void testUse(){
        System.out.println("uasrname: "+username);
        System.out.println("password: "+password);
    }
    @Override
    public String toString() {
        return "TestSerialize{" +
                "username='" + username + '\'' +
                ", password='" + password + '\'' +
                '}';
    }
}

```

main

```java
import java.io.*;

public class main {


    public static void main(String[] args) {
        TestSerialize ser = new TestSerialize("liangban","123123");
        try {
            // 创建一个FIleOutputStream类
            FileOutputStream fos = new FileOutputStream("./Test.ser");
            // FileOutputStream类,字节输出流，用于处理原始二进制数据。将数据写到文件,需要将数据转换成字节并将其保存到文件。

            // 将这个FIleOutputStream类封装到ObjectOutputStream中
            ObjectOutputStream oos = new ObjectOutputStream(fos);
            // ObjectOutputStream，对象的输出流，将指定的对象写入到文件完成对象的序列化过程

            // 调用writeObject方法，序列化对象到文件Test.ser中
            oos.writeObject(ser);

            // 创建一个FIleInutputStream类
            FileInputStream fis = new FileInputStream("./Test.ser");
            // FileInputStream文件输入流，是将文本文件中的数据输入到内存中。他是一个字节输入流，是InputStream抽象类的一个子类

            // 将FileInputStream类封装到ObjectInputStream中
            ObjectInputStream ois = new ObjectInputStream(fis);
            // ObjectInputStream,反序列化流，将使用ObjectOutputStream序列化的原始数据恢复为对象，以流的方式读取对象

            // 调用readObject从user.ser中反序列化出对象，默认是Object类型,需要进行类型转换，
            TestSerialize test = (TestSerialize)ois.readObject();

            test.testUse();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}

```

![image-20221126145422550](https://static.sechelper.com/img/2022/11/27/3d294d853148ac117f8c2a55f1d2b93a.png)

![image-20221127125955115](https://static.sechelper.com/img/2022/11/27/d76c429bf9b91abf29696f1c9f89d49a.png)

0xACED：STREAM_MAGIC，声明使用了序列化协议，**从这里可以判断保存的内容是否为序列化数据。** （这是在黑盒挖掘反序列化漏洞很重要的一个点）

0x0005：STREAM_VERSION，序列化协议版本。

0x73:	TC_OBJECT

0x72:	TC_CLASSDESC

0x00...01：serialVersionUID

### serialVersionUID

`private static final long serialVersionUID = 1;`

作用：在反序列化的时候保证与本地类的版本相同

不自定义会自动生成UID

如果两个不同内容的类，在包名类名都一样时，是不能进行相互反序列化的，但如果定义的UID一样，那么生成的序列化文件就可以进行反序列化操作。



## 自定义序列化

在序列化一个类的时候并不想写入多余的数据，需要自定义读取和写入 readObject	write Object

java是支持自定义readObject与writeObject方法的,只要某个类中按照特定的要求实现了readObject方法，那么在反序列化的时候就会自动调用它.

```java
private void writeObject(ObjectOutputStream oos)throws IOException {
    oos.writeUTF(username);
    oos.writeUTF(password);
    System.out.println("Test Serialize writeObject");
}
```

```java
private void readObject(ObjectInputStream ois)throws IOException,ClassNotFoundException {
	username = ois.readUTF();
    type = ois.readUTF();
    System.out.println("Test Serialize writeObject");
}
```

读写顺序要一致，先写入username变量，那么读取时也要先读取username变量，不然会报错。

```java
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;

public class TestSerialize implements Serializable {

    private static final long serialVersionUID = -123123123L;
    public String username;
    transient private String password;
    public int age;

    public TestSerialize(String name, String pass, int age) {
        this.username = name;
        this.password = pass;
        this.age = age;
    }
    public void testUse(){
        System.out.println("usernaem: "+username);
        System.out.println("password: "+password);
        System.out.println("age: "+age);
    }
    private void writeObject(ObjectOutputStream oos)throws IOException {
        oos.writeUTF(username);
        oos.writeUTF(password);
        System.out.println("Test Serialize writeObject");}

    private void readObject(ObjectInputStream ois)throws IOException,ClassNotFoundException {
        username = ois.readUTF();
        password = ois.readUTF();
        System.out.println("Test Serialize writeObject");}
}
```

```java
import java.io.*;

public class main {

    public static void main(String[] args) {
        TestSerialize ser = new TestSerialize("liangban","123123",19);
        try {
            FileOutputStream fos = new FileOutputStream("./Test.ser");
            ObjectOutputStream oos = new ObjectOutputStream(fos);
            oos.writeObject(ser);
            
            FileInputStream fis = new FileInputStream("./Test.ser");
            ObjectInputStream ois = new ObjectInputStream(fis);
            TestSerialize test = (TestSerialize)ois.readObject();
            test.testUse();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
    }
}
```

![image-20221126162950498](https://static.sechelper.com/img/2022/11/27/2e2c9aabc38a1c18029404b350ee609a.png)

自定义读写只有name和pass没有age，所以这里就算给age传入了19但输出age为0

在自定义writeObject时手动把被transient修饰的变量写进去，在读取readObject时也手动写出来，pass是可以被修改拿到的。



## 反序列化漏洞成因

序列化和反序列化本身并不存在问题。但当输入的反序列化的数据可被用户控制，那么攻击者即可通过构造恶意输入，让反序列化产生非预期的对象，非预期的对象在产生过程中就可能带来安全问题。

广义上来讲，传的xml，json等内容可以进行反序列化操作，再次拿到java对象，也可以叫反序列化漏洞



### 例子

如果自定义的readObject方法里进行了一些危险操作，那么就会导致反序列化漏洞的发生了。

TestSeria

```java
import java.io.IOException;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;

public class TestSerialize implements Serializable {
    public String cmd = null;
    public TestSerialize(String cmd) {
        this.cmd = cmd;
    }
    private void writeObject(ObjectOutputStream oos)throws IOException {
        oos.defaultWriteObject();
        System.out.println("Test Serialize writeObject");}

    private void readObject(ObjectInputStream ois)throws IOException,ClassNotFoundException {
        ois.defaultReadObject();
        //调用系统执行命令功能，去执行cmd这个变量的命令
        Runtime.getRuntime().exec(cmd);
        System.out.println("Test Serialize writeObject");}
}

```

main

```java
import java.io.*;

public class main {
    public static void main(String[] args) {
        TestSerialize ser = new TestSerialize("notepad");
        try {
            FileOutputStream fos = new FileOutputStream("./Test02.ser");
            ObjectOutputStream oos = new ObjectOutputStream(fos);
            oos.writeObject(ser);
            FileInputStream fis = new FileInputStream("./Test02.ser");
            ObjectInputStream ois = new ObjectInputStream(fis);
            TestSerialize test = (TestSerialize)ois.readObject();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
    }
}
```

![image-20221127133925800](https://static.sechelper.com/img/2022/11/27/cfc9a12983737f9d6466abb6df8e7b9f.png)

在反序列化的时候会主动调用readObject就会触发命令执行，弹出了记事本。

这里只是演示，应该没有人会在写Runtime.getRuntime().exec(cmd);大多时是利用反射去构造java方法，再通过反射去调用java方法

Runtime.exec()：直接在目标环境执行命令

Method.invoke()：需要适当的选择方法和参数，通过反射执行java方法

RMI/JNDI/JRMP等：通过引用远程对象，间接实现任意代码执行的效果

后续介绍java反射，CC6链源码分析，LazyMap利用连，ysoser工具的使用
