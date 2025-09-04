---
Title: Java 基础核心类-String相关
Category: Java
tags:
  - java
  - 数据类型
---
## String

Java 中的 String 是不可变对象

在面向对象及函数编程语言中，不可变对象（英语：Immutable object）是一种对象，在被创造之后，它的状态就不可以被改变。至于状态可以被改变的对象，则被称为可变对象（英语：mutable object）。-- 来自百度百科

### Java 8 String 源码

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];//Java 9已经优化为byte数组了

    /** Cache the hash code for the string */
    private int hash; // Default to 0
    
    ...
}
```

显然 String 字符串内部是使用 char[]数组来存储。

而这个 char[]数组是用 **private final**来修饰的，**private**就体现着面向对象的封装特性，并且 String 没有提供供外部访问的方法，这就意味着这个属性无法被外部访问；**final**则意味着这个属性无法修改，无法重新指向其他对象。且 String 类没有提供/暴露修改这个字符串的方法。

因此，String 是不可变对象

#### 不可变的优点

- 线程安全。同一个字符串实例可以被多个线程共享，因为字符串不可变，本身就是线程安全的。
- 支持 hash 映射。因为 String 的 hash 值经常会使用到，比如作为 Map 的键，不可变的特性也就使得 hash 值不会变，不需要重新计算。
- 字符串常量池优化。String 对象创建之后，会缓存到字符串常量池中，下次需要创建同样的对象时，可以直接返回缓存的引用。

#### 一定不可变吗

事实上，可以通过反射来改变 String 中的值

```java
String str = "abcdef";
System.out.println("修改前的地址值：" + str + ",hash值"+ str.hashCode());
Class<? extends String> aClass = str.getClass();
Field value = aClass.getDeclaredField("value");
value.setAccessible(true);
value.set(str,"seven".getBytes());
System.out.println("修改后的地址值：" + str + ",hash值"+ str.hashCode());
```

显然修改后的还是同一个地址和 hash 值

```java
修改前的地址值：abcdef,hash值-1424385949
修改后的地址值：seven,hash值-1424385949
```

查看源码可以看到，计算 hashcode 后 hash 值是由一个常量缓存下来的，所以通过反射修改后 hashCode 并不会变，除非进行重新计算。

注意：用反射修改 String 的值破坏了 String 的 immutable 特征，可能会带来各种问题，以上只是提供一个思路，不建议这么做。

> 不可变类都建议参考 String 类一样，写个变量缓存 hashcode，从而防止高并发下的计算

```java
public int hashCode() {
    int h = hash;
    if (h == 0 && !hashIsZero) {
        h = isLatin1() ? StringLatin1.hashCode(value)
                       : StringUTF16.hashCode(value);
        if (h == 0) {
            hashIsZero = true;
        } else {
            hash = h;
        }
    }
    return h;
}
```


### String 能存储多少字符

能存储多少字符，通过以下步骤来看：

1. 首先 String 的 length 方法返回是 int。所以理论上长度一定不会超过 int 的最大值。
2. 编译器对字符串字面量长度的限制源自 Java 编译器（如 `javac`）在处理常量池时的实现。编译器源码如下，限制了字符串长度大于等于 65535 就会编译不通过：
   ```java
   // src/jdk.compiler/share/classes/com/sun/tools/javac/jvm/Pool.java
   public class Pool {
       // ...
   
       /**
        * Add a new Utf8 string to the constant pool, checking for duplicates
        * and sharing the entry if one already exists.
        */
       public int putUtf8(String x) {
           Assert.checkNonNull(x);
           byte[] bytes;
           try {
               ByteArrayOutputStream bytearrayoutputstream = new ByteArrayOutputStream();
               DataOutputStream dataoutputstream = new DataOutputStream(bytearrayoutputstream);
               dataoutputstream.writeUTF(x);
               dataoutputstream.close();
               bytes = bytearrayoutputstream.toByteArray();
           } catch (IOException e) {
               throw new AssertionError(e);
           }
           if (bytes.length > 65535)
               throw new UTFDataFormatException("encoded string too long: " + bytes.length + " bytes");
           return put(new Pool.Utf8Entry(bytes));
       }
   
       // ...
   }
   
   ```

Java 中的字符常量都是使用 UTF 8 编码的，UTF 8 编码使用 1~4 个字节来表示具体的 Unicode 字符。所以有的字符占用一个字节，而平时所用的大部分中文都需要 3 个字节来存储。

```java
//65534个字母，编译通过
String s1 = "dd..d";

//21845个中文”自“,编译通过
String s2 = "自自...自";

//一个英文字母d加上21845个中文”自“，编译失败
String s3 = "d自自...自";
```

- 对于 s 1，一个字母 d 的 UTF 8 编码占用一个字节，65534 个字母占用 65534 个字节，长度是 65534，长度和存储都没超过限制，所以可以编译通过。

- 对于 s 2，一个中文占用 3 个字节，21845 个正好占用 65535 个字节，而且字符串长度是 21845，长度和存储也都没超过限制，所以可以编译通过。

- 对于 s 3，一个英文字母 d 加上 21845 个中文”自“占用 65536 个字节，超过了存储最大限制，编译失败。

当然，这个限制是特定于编译器的实现，而不是 Java 语言本身的限制。



3. JVM 规范对常量池有所限制。

量池中的每一种数据项都有自己的类型。Java 中的 UTF-8 编码的 Unicode 字符串在常量池中以 `CONSTANTUtf8` 类型表示。`CONSTANTUtf8` 的数据结构如下：

```c
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
}
```

重点关注长度为 length 的那个 bytes 数组，这个数组就是真正存储常量数据的地方，而 length 就是数组可以存储的**最大字节数**，而不是字符数。Length 的类型是 u 2，u 2 是无符号的 16 位整数，因此理论上允许的的最大长度是 `2^16-1=65535`。所以上面 byte 数组的最大长度可以是 65535。

当然，考虑到 UTF-8 是一种变长编码，一个字符可能需要 1 到 4 个字节来表示（取决于字符的具体值）。因此，如果你的字符串包含大量使用多个字节编码的字符，那么它能包含的实际字符数将会少于 65535。



4. 运行时限制

String 运行时的限制主要体现在 String 的构造函数上。下面是 String 的一个构造函数：

```java
public String(char value[], int offset, int count) {
    ...
}
```

上面的 count 值就是字符串的最大长度。在 Java 中，int 的最大长度是 2^31-1。所以在运行时，String 的最大长度是 2^31-1。

但是这个也是理论上的长度，实际的长度还要**看 JVM 的内存**。来看下，最大的字符串会占用多大的内存。

```
(2^31-1)*16/8/1024/1024/1024 = 2GB
```

所以在最坏的情况下，一个最大的字符串要占用 4 GB 的内存。如果 JVM 不能分配这么多内存的话，会直接报错的。



**总结**：因此，主要的还是看编译器对常量池的限制，使得 byte 数组的最大长度不能超过 65535；以及 JVM 的内存限制

**补充**：JDK 9 以后对 String 的存储进行了优化。底层不再使用 char 数组存储字符串，而是使用 byte 数组。对于 LATIN 1 字符的字符串可以节省一倍的内存空间。详情请看 [Java9新特性](03_Developlanguage/Java/NewFeatures/Java9新特性.md)



### String, StringBuffer 和 StringBuilder 的区别

1. 可变性

   - String 不可变

   - StringBuffer 和 StringBuilder 可变

2. 线程安全

   - String 不可变，因此是线程安全的

   - StringBuilder 不是线程安全的

   - StringBuffer 是线程安全的，内部使用 synchronized 进行同步



StringBuffer 的 append 方法

```java
@Override
public synchronized StringBuffer append(Object obj) {
    toStringCache = null;
    super.append(String.valueOf(obj));
    return this;
}
```



### 循环拼接字符串建议 StringBuilder

#### JDK 8 下的字符串拼接实现

```java
class Demo {
    public static String concatIndy(int i) {
        return  "value " + i;
    }
}
```

编译查看字节码

```
jdk8/bin/javac Demo.java 
jdk8/bin/javap -c Dem
```


Javap 输出的字节码：

```java
class Demo {
  Demo();
    Code:
       0: aload_0
       1: invokespecial #1 // Method java/lang/Object."<init>":()V
       4: return

  public static java.lang.String concatIndy(int);
    Code:
       0: new           #2 // class java/lang/StringBuilder
       3: dup
       4: invokespecial #3 // Method java/lang/StringBuilder."<init>":()V
       7: ldc           #4 // String value
       9: invokevirtual #5 // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      12: iload_0
      13: invokevirtual #6 // Method java/lang/StringBuilder.append:(I)Ljava/lang/StringBuilder;
      16: invokevirtual #7 // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      19: areturn
}
```

反编译后的 Java 代码

```java
public static String concatIndy(int i) {
    return new StringBuilder("value ")
            .append(i)
            .toString();
}
```

可以看出，+ 号操作符其实就是一种语法糖，让字符串的拼接变得更简便了。可以看到编译器自动将 `+` 转换成了 `StringBuilder.append()` 方法，拼接之后再调用 `StringBuilder.toString()` 方法转换成字符串。

但这是单次拼接，如果要拼接大量字符串呢？例如使用 `for` 循环模拟频繁的字符串拼接操作时，使用 `+` 的话，在每一次循环中，都将重复下列操作：

- 新建 `StringBuilder` 对象
- 调用 `StringBuilder.append()` 方法
- 调用 `StringBuilder.toString()` 方法，该方法会通过 `new String()` 创建字符串

如果是几万次循环下来，可以看看创建了多少中间对象，别人要么以空间换时间，要么以时间换空间。这家伙倒好，即浪费时间，又浪费空间。所以，在频繁拼接字符串的情况下，尽量避免使用 `+` 。



#### StringBuilder 源码

String 源码，存放字符串的地方

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    /** The value is used for character storage. */
    private final char value[];
    
    ...
}
```

StringBuilder 自身没有自定义存储的容器，而是继承了其父类的容器

```java
abstract class AbstractStringBuilder implements Appendable, CharSequence {
    /**The value is used for character storage.*/
    char[] value;
    
    ...
}
```

不一样的地方就在于 String 的 value 是不可变的，而 StringBuilder 的 value 是可变的



##### String 拼接字符串案例

```java
String s1 = "第1个字符串";
String s2 = "第2个字符串";
String str = s1 + s2;
```

以上操作可以看成是

```java
//这里只作为理解，相当于新开拓一个字符数组，然后复制
final char c1[] = {'第','1','个','字','符','串'};
final char c2[] = {'第','2','个','字','符','串'};
final char c3[] = new char[12];
c3[] =  {'第','1','个','字','符','串','第','2','个','字','符','串'};
```

- 创建 s 1 的时候其实就是创建了第一个不可变的 char[]数组，创建 s 2 的时候创建了第二个不可变的 char[]数组

- 创建 str 的时候其实就是另外又创建了一个数组，再将 s 1 和 s 2 的数据复制到 str 中

##### StringBuilder 拼接字符串案例

```java
StringBuilder sb = new StringBuilder();
System.out.println("初始容量：" + sb.capacity());
sb.append("十五个十五个十五个十五个十五个");
System.out.println("追加15个字后sb容量：" + sb.capacity());
sb.append("一");
System.out.println("已经十六个字SB容量：" + sb.capacity());
sb.append("添加");
System.out.println("超过16个字的SB容量：" + sb.capacity());
```

输出

```java
初始容量：16
追加15个字后sb容量：16
已经十六个字SB容量：16
超过16个字的SB容量：34
```



**StringBuilder 特征**：StringBuilder 初始化容量是 16 (无参构造)

```java
public StringBuilder() {
  super(16);
}
```

- 追加之前会计算一次容量，大于所需容量则会重新创建一个 char[]数组，计算规则是 newCapacity = (value. Length << 1) + 2; 也就是原来长度*2 + 2

- StringBuilder 在运算的时候每次会计算容量是否足够，如果所需容量不小于自身容量, 那么就会重新分配一个自身容量两倍 +2 的 char[].

```java
//追加操作
public AbstractStringBuilder append(String str) {
    if (str == null)
        return appendNull();
    int len = str.length();
    //count是当前char[]数组的使用大小，len是要追加的字符串的长度
    ensureCapacityInternal(count + len);
    str.getChars(0, len, value, count);
    count += len;
    return this;
}

//这个方法是确保 char[]数组的大小能装下新追加的字符串
private void ensureCapacityInternal(int minimumCapacity) {
     // overflow-conscious code
    //判断所需要的容量是否小于char[]数组的容量
    if (minimumCapacity - value.length > 0) {
        value = Arrays.copyOf(value,
     newCapacity(minimumCapacity));//如果小于，就扩容，并拷贝数组内容
    }
}
    
private int newCapacity(int minCapacity) {
    // overflow-conscious code
   int newCapacity = (value.length << 1) + 2;//扩容数组大小，也就是原来长度*2+2
   if (newCapacity - minCapacity < 0) {
       newCapacity = minCapacity;
   }
   return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
       ? hugeCapacity(minCapacity)
       : newCapacity;
}
```

因此，如果再一次追加的时候容量足够，就无需创建新数组，也就省去了很多创建 char[]的次数.



#### 小结

- 在非循环体内

  - 可以看出，在 JDK 8 中，在非循环体内（少数拼接）使用"+"实现字符串拼接和使用 StringBuilder 是一样的，用"+"做拼接代码更简洁，推荐使用"+"

- 拼接次数较多，如循环体内拼接

  - String 之所以慢是因为，大部分 cpu 资源都被浪费在分配资源，新建 StringBuilder 对象，拷贝资源的部分了，相比 StringBuilder 有更多的内存消耗。

  - StringBuilder 快就快在，相比 String，他在运算的时候分配内存次数小，所以拷贝次数和内存占用也随之减少，当有大量字符串拼接时，StringBuilder 创建 char[]的次数会少很多。

  - 由于 GC 的机制，即使原来的 char[]没有引用了，那么也得等到 GC 触发的时候才能回收，String 运算过多的时候就会产生大量垃圾，消耗内存。


因此：

- 如果目标字符串需要大量拼接的操作，那么这个时候应当使用 StringBuilder。

- 反之，如果目标字符串操作次数极少，或者是常量，那么就直接使用 String。



#### 着重注意

这是 JDK 8 版本的结论，在 JDK 9 之后，JDK 官方已经对字符串拼接做了优化，使用"+"做字符串拼接会比 StringBuilder 快，详情可以看这篇文章，[Java9新特性](03_Developlanguage/Java/NewFeatures/Java9新特性.md)。

但这也是在非循环体内，少数拼接的情况，当多大量拼接，还是建议使用 StringBuilder


### String. Intern ()方法

调用字符串对象的 intern 方法，会将该字符串对象尝试放入到串池中

- 如果串池中没有该字符串对象，则放入成功

- 如果有该字符串对象，则放入失败

无论放入是否成功，都会返回串池中的字符串对象

注意：此时如果调用 intern 方法成功，堆内存与串池中的字符串对象是同一个对象；如果失败，则不是同一个对象

##### 例 1：

```java
//"a" "b" 被放入串池中，str则存在于堆内存之中
String str = new String("a") + new String("b");
//调用str的intern方法，这时串池中没有"ab"，则会将该字符串对象放入到串池中，此时堆内存与串池中的"ab"是同一个对象
String st2 = str.intern();
//给str3赋值，因为此时串池中已有"ab"，则直接将串池中的内容返回
String str3 = "ab";
//因为堆内存与串池中的"ab"是同一个对象，所以以下两条语句打印的都为true
System.out.println(str == st2);
System.out.println(str == str3);
```

##### 例 2：

```java
//此处创建字符串对象"ab"，因为串池中还没有"ab"，所以将其放入串池中
String str3 = "ab";
//"a" "b" 被放入串池中，str则存在于堆内存之中
String str = new String("a") + new String("b");
//此时因为在创建str3时，"ab"已存在于串池中，所以放入失败，但是会返回串池中的"ab"
String str2 = str.intern();
//false，str在堆内存，str2在串池
System.out.println(str == str2);
//false，str在堆内存，str3在串池
System.out.println(str == str3);
//true，str2和str3是串池中的同一个对象
System.out.println(str2 == str3);
```


### 如何自定义一个 String 类

#### 包名不为 java. Lang

```java
package com.seven.jvm;

public final class String {
    /** The value is used for character storage. */
    private final char value[] = {};

    /** Cache the hash code for the string */
    private int hash; // Default to 0

    /** use serialVersionUID from JDK 1.0.2 for interoperability */
    private static final long serialVersionUID = -6849794470754667710L;

    public String(int hash) {
        this.hash = hash;
    }

    public String(){

    }

    static{
        System.out.println("静态代码块--自定义String");
    }

    {
        System.out.println("代码块--自定义String");
    }
    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }

}
```

可以正常使用，但在使用时有有用到 java. Lang 下的 String，那就需要区别包名使用，即其中一个使用全类名表示，一般生产中不会去自定义一个与 JDK 类库中同名的类，这里只作为拓展了解即可~


#### 包名为 java. Lang. String

```java
package java.lang;


public final class String {

    /**
     * The value is used for character storage.
     */
    private final char value[] = {};

    /**
     * Cache the hash code for the string
     */
    private int hash; // Default to 0

    /**
     * use serialVersionUID from JDK 1.0.2 for interoperability
     */
    private static final long serialVersionUID = -6849794470754667710L;

    public String(int hash) {
        this.hash = hash;
    }

    public String() {

    }

    static {
        System.out.println("静态代码块--自定义String");
    }

    {
        System.out.println("代码块--自定义String");
    }

    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }

}
```

##### String 类下写 main 方法

```java
package java.lang;

public final class String {

    public static void main(String[] args) {
        System.out.println("aaa");;
    }
}
```

输出：

```java
错误: 在类 java.lang.String 中找不到 main 方法, 请将 main 方法定义为:
   public static void main(String[] args)
否则 JavaFX 应用程序类必须扩展javafx.application.Application
```

原因在于[解密类加载机制：深入理解 JVM 如何加载你的代码](03_Developlanguage/Java/JVM/解密类加载机制：深入理解%20JVM%20如何加载你的代码.md)，先从父类加载器寻找能不能加载此类，如果没有则再到子类；因此在加载 String 类时，会最终委派给 Bootstrap ClassLoader 去加载，加载的是 rt. Jar 包中的那个 java. Lang. String，而 rt. Jar 包中的 String 类是没有 main 方法的，因此报错误

##### 同包下新建一个类写 main 方法

```java
package java.lang;

public class Main {
    public static void main(String[] args) {
       String str =  new String();
    }
}
```

输出：

```java
java.lang.SecurityException: Prohibited package name: java.lang
    at java.lang.ClassLoader.preDefineClass(ClassLoader.java:662)
    at java.lang.ClassLoader.defineClass(ClassLoader.java:761)
    at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
    at java.net.URLClassLoader.defineClass(URLClassLoader.java:467)
    at java.net.URLClassLoader.access$100(URLClassLoader.java:73)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:368)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:362)
    at java.security.AccessController.doPrivileged(Native Method)
    at java.net.URLClassLoader.findClass(URLClassLoader.java:361)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
    at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:335)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
    at sun.launcher.LauncherHelper.checkAndLoadMain(LauncherHelper.java:495)
Error: A JNI error has occurred, please check your installation and try again
Exception in thread "main" 
```

限制包名，不能自定义这个包名，与 java 类库冲突，安全管理器不通过，这里不管用不用到 String 都会有这个报错

原因：java. Lang 是 java 自带类库包，是属于 rt. Jar 包下的文件，而 rt. Jar 是通过启动类加载器 (Bootstrap ClassLoader) 加载的，由于双亲委派，因此 java. Lang 包肯定早于自定义的 java. Lang 包的加载，就会冲突。

##### 调用方法不在 java. Lang 包中

```java
package com.seven;

public class Test {
    public static void main(String[] args) {
        String string = new String();
    }
}
```

无输出

原因，由于双亲委派，这里加载的 String 包是 rt. Jar 中的 java. Lang. String 类。因此这里并没有用到自定义的 String 类，因为不会加载到自定义的 String（即便改自定义 String 的包名也叫 java. Lang）

#### 小结

1. 可以自定义包名不为 java. Lang 的 String 类，并区别包名正常使用

2. 自定义包名为 java. Lang 的 String 类

   - String 类下写 main 方法：由于双亲委派模型，在加载 String 类时，会最终委派给 Bootstrap ClassLoader 去加载，加载的是 rt. Jar 包中的那个 java. Lang. String，而 rt. Jar 包中的 String 类是没有 main 方法的，因此报错误

   - 启动类也在 java. Lang 包下：这里与是否用到 String 类无关，会报 Prohibited package name: java. Lang 错误。由于双亲委派，java. Lang 包肯定早于自定义的 java. Lang 包的加载，就会冲突.

   - 调用方法不在 java. Lang 包中：此时由于双亲委派模型的存在，并不会加载到自定义的 String 类
