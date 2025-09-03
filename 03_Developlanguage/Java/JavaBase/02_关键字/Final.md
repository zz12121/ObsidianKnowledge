---
Title: Java 基础核心要点-关键字-Final
Category: Java
tags:
  - java
  - 关键字
---
### 01、final 变量  
  
被 final 修饰的变量无法重新赋值。换句话说，final 变量一旦初始化，就无法更改。  
  
```java  
final int age = 18;  
```  
  
当尝试将 age 的值修改为 30 的时候，编译器就生气了。  
  
```java  
public class Pig {
   private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}
```  
  
这是一个很普通的 Java 类，它有一个字段 name。  
  
然后，我们创建一个测试类，并声明一个 final 修饰的 Pig 对象。  
  
```java  
final Pig pig = new Pig();  
```  
  
如果尝试将 pig 重新赋值的话，编译器同样会生气。    
  
但我们仍然可以去修改 pig 对象的 name。  
  
```java  
final Pig pig = new Pig();  
pig.setName("特立独行");  
System.out.println(pig.getName()); // 特立独行  
```  
  
另外，final 修饰的成员变量必须有一个默认值，否则编译器将会提醒没有初始化。    
  
final 和 static 一起修饰的成员变量叫做常量，常量名必须全部大写。  
  
```java  
public class Pig {
   private final int age = 1;
   public static final double PRICE = 36.5;
}
```  
  
有时候，我们还会用 final 关键字来修饰参数，它意味着参数在方法体内不能被再修改。  
  
来看下面这段代码。  
  
```java  
public class ArgFinalTest {  
    public void arg(final int age) {    
    }  
    public void arg1(final String name) {    
    }
}  
```  
  
如果尝试去修改它的话，编译器会提示错误。    

数据：声明数据为常量，可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量。

   - 对于基本类型，final 使数值不变；

   - 对于引用类型，final 使引用不变，也就不能引用其它对象，但是被引用的对象本身是可以修改的。
  
### 02、final 方法  
  
被 final 修饰的方法不能被重写。如果我们在设计一个类的时候，认为某些方法不应该被重写，就应该把它设计成 final 的。  
  
Thread 类就是一个例子，它本身不是 final 的，这意味着我们可以扩展它，但它的 `isAlive()` 方法是 final 的。  
  
```java  
public class Thread implements Runnable {  
    public final native boolean isAlive();
}  
```  
需要注意的是，该方法是一个本地（[Native](03_Developlanguage/Java/JavaBase/05_面向对象/Native方法.md)）方法，用于确认线程是否处于活跃状态。而本地方法是由操作系统决定的，因此重写该方法并不容易实现。  
  
来看这段代码。  
  
```java  
public class Actor {  
    public final void show() {  
    }
	}  
```  
  
当我们想要重写该方法的话，就会出现编译错误。    
  
一个类是 final 的，和一个类不是 final，但它所有的方法都是 final 的，考虑一下，它们之间有什么区别？  
  
我能想到的一点，就是前者不能被继承，也就是说方法无法被重写；后者呢，可以被继承，然后追加一些非 final 的方法。  

Private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法。
  
### 03、final 类  
  
如果一个类使用了 final 关键字修饰，那么它就无法被继承.....  
  
[String](03_Developlanguage/Java/JavaBase/03_基础核心类/String相关.md) 类就是一个 final 类。  
  
```java  
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence,
               Constable, ConstantDesc {}
```  
  
为什么 String 类要设计成 final 吗？  
  
原因大致有 3 个。  
  
- 为了实现字符串常量池  
- 为了线程安全  
- 为了 HashCode 的不可变性  
  
任何尝试从 final 类继承的行为将会引发编译错误。来看这段代码。  
  
```java  
public final class Writer {
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
} 
```  
  
尝试去继承它，编译器会提示以下错误，Writer 类是 final 的，无法继承。    
  
不过，类是 final 的，并不意味着该类的对象是不可变的。  
  
来看这段代码。  
  
```java  
Writer writer = new Writer();  
writer.setName("呵呵");  
System.out.println(writer.getName()); // 呵呵  
```  
  
Writer 的 name 字段的默认值是 null，但可以通过 settter 方法将其更改为呵呵。也就是说，如果一个类只是 final 的，那么它并不是不可变的全部条件。  
  
把一个类设计成 final 的，有其安全方面的考虑，但不应该故意为之，因为把一个类定义成 final 的，意味着它没办法继承，假如这个类的一些方法存在一些问题的话，我们就无法通过重写的方式去修复它。  


