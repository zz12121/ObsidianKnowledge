---
Title: Java 基础核心要点-关键字-Static
Category: Java
tags:
  - java
  - 关键字
---
static 关键字的作用可以用一句话来描述：‘**方便在没有创建对象的情况下进行调用**，包括变量和方法’。也就是说，只要类被加载了，就可以通过类名进行访问。static 可以用来修饰类的成员变量，以及成员方法。我们一个个来看。  
  
### 01、静态变量  
  
如果在声明变量的时候使用了 static 关键字，那么这个变量就被称为静态变量。静态变量只在类加载的时候获取一次内存空间，这使得静态变量很节省内存空间。  
  
来考虑这样一个 Student 类。  
  
```java  
public class Student {
    String name;
    int age;
    String school = "郑州大学";
}
```  
  
假设郑州大学录取了一万名新生，那么在创建一万个 Student 对象的时候，所有的字段（name、age 和 school）都会获取到一块内存。学生的姓名和年纪不尽相同，但都属于郑州大学，如果每创建一个对象，school 这个字段都要占用一块内存的话，就很浪费，对吧？  
  
因此，最好将 school 这个字段设置为 static，这样就只会占用一块内存，而不是一万块。  
  
```java  
public class Student {
    String name;
    int age;
    static String school = "郑州大学";

    public Student(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public static void main(String[] args) {
        Student s1 = new Student("呵呵", 18);
        Student s2 = new Student("呵呵", 16);
    }
}
```  
  
s1 和 s2 这两个引用变量存放在栈区（stack），呵呵+18 这个对象和呵呵+16 这个对象存放在堆区（heap），school 这个静态变量存放在静态区。    
  
```java  
public class Counter {
    int count = 0;

    Counter() {
        count++;
        System.out.println(count);
    }

    public static void main(String args[]) {
        Counter c1 = new Counter();
        Counter c2 = new Counter();
        Counter c3 = new Counter();
    }
} 
```  
  
我们创建一个成员变量 count，并且在构造函数中让它自增。因为成员变量会在创建对象的时候获取内存，因此每一个对象都会有一个 count 的副本， count 的值并不会随着对象的增多而递增  
  
这段代码输出的结果是什么？  
  
```  
1  
1  
1  
```  
  
每创建一个 Counter 对象，count 的值就从 0 自增到 1。三妹，想一下，如果 count 是静态的呢？  
  
```java  
public class StaticCounter {
    static int count = 0;

    StaticCounter() {
        count++;
        System.out.println(count);
    }

    public static void main(String args[]) {
        StaticCounter c1 = new StaticCounter();
        StaticCounter c2 = new StaticCounter();
        StaticCounter c3 = new StaticCounter();
    }
}
```  
  
来看一下输出结果。  
  
```  
1  
2  
3  
```  
  
简单解释一下哈，由于静态变量只会获取一次内存空间，所以任何对象对它的修改都会得到保留，所以每创建一个对象，count 的值就会加 1，所以最终的结果是 3，明白了吧？这就是静态变量和成员变量之间的差别。  
  
另外，需要注意的是，由于静态变量属于一个类，所以不要通过对象引用来访问，而应该直接通过类名来访问，否则编译器会发出警告。    
  
### 02、静态方法  
  
如果方法上加了 static 关键字，那么它就是一个静态方法。  
  
静态方法有以下这些特征。  
  
- 静态方法属于这个类而不是这个类的对象；  
- 调用静态方法的时候不需要创建这个类的对象；  
- 静态方法可以访问静态变量。  
  
```java  
public class StaticMethodStudent {
    String name;
    int age;
    static String school = "郑州大学";

    public StaticMethodStudent(String name, int age) {
        this.name = name;
        this.age = age;
    }
    
    static void change() {
        school = "河南大学";
    }
    
    void out() {
        System.out.println(name + " " + age + " " + school);
    }

    public static void main(String[] args) {
        StaticMethodStudent.change();
        
        StaticMethodStudent s1 = new StaticMethodStudent("呵呵", 18);
        StaticMethodStudent s2 = new StaticMethodStudent("呵呵", 16);
        
        s1.out();
        s2.out();
    }
}
```  
  
`change()` 方法就是一个静态方法，所以它可以直接访问静态变量 school，把它的值更改为河南大学；并且，可以通过类名直接调用 `change()` 方法，就像 ` StaticMethodStudent.change()` 这样。  
  
来看一下程序的输出结果吧。  
  
```  
呵呵 18 河南大学  
呵呵 16 河南大学  
```  
  
需要注意的是，静态方法不能访问非静态变量和调用非静态方法。  
  
先是在静态方法中访问非静态变量，编译器不允许。  
  
然后在静态方法中访问非静态方法，编译器同样不允许。    
  
为什么 main 方法是静态的啊？  
  
如果 main 方法不是静态的，就意味着 Java 虚拟机在执行的时候需要先创建一个对象才能调用 main 方法，而 main 方法作为程序的入口，创建一个额外的对象显得非常多余。  
  
java.lang.Math 类的几乎所有方法都是静态的，可以直接通过类名来调用，不需要创建类的对象。    
  
### 03、静态代码块  
  
除了静态变量和静态方法，static 关键字还有一个重要的作用。用一个 static 关键字，外加一个大括号括起来的代码被称为静态代码块。  
  
```java  
public class StaticBlock {
    static {
        System.out.println("静态代码块");
    }

    public static void main(String[] args) {
        System.out.println("main 方法");
    }
}
```  
  
静态代码块通常用来初始化一些静态变量，它会优先于 `main()` 方法执行。  
  
来看一下程序的输出结果吧。  
  
```  
静态代码块  
main 方法  
```  
  
既然静态代码块先于 `main()` 方法执行，那没有 `main()` 方法的 Java 类能执行成功吗？  
  
Java 1.6 是可以的，但 Java 7 开始就无法执行了。
  
```java  
public class StaticBlockNoMain {
    static {
        System.out.println("静态代码块，没有 main");
    }
}
```  
  
在命令行中执行 `java StaticBlockNoMain` 的时候，会抛出 NoClassDefFoundError 的错误。
  
```java  
public class StaticBlockDemo {  
    public static List<String> writes = new ArrayList<>();  
    
    static {        
    	writes.add("呵呵1");  
        writes.add("呵呵2");  
        writes.add("呵呵3");  
  
        System.out.println("第一块");  
    }  
    static {        
    	writes.add("呵呵4");  
        writes.add("呵呵5");  
  
        System.out.println("第二块");  
    }
}  
```  
  
writes 是一个静态的 ArrayList，所以不太可能在声明的时候完成初始化，因此需要在静态代码块中完成初始化。  
  
静态代码块在初始集合的时候，真的非常有用。在实际的项目开发中，通常使用静态代码块来加载配置文件到内存当中。  
  
### 04、静态内部类  
  
Java 允许我们在一个类中声明一个内部类，它提供了一种令人信服的方式，允许我们只在一个地方使用一些变量，使代码更具有条理性和可读性。  
  
常见的[内部类](03_Developlanguage/Java/JavaBase/05_面向对象/内部类.md) 有四种，成员内部类、局部内部类、匿名内部类和静态内部类，前三种不在我们本次的讨论范围之内，以后有机会再细说。  
  
```java  
public class Singleton {
    private Singleton() {}

    private static class SingletonHolder {
        public static final Singleton instance = new Singleton();
    }

    public static Singleton getInstance() {
        return SingletonHolder.instance;
    }
}
```  
  
第一次加载 Singleton 类时并不会初始化 instance，只有第一次调用 `getInstance()` 方法时 Java 虚拟机才开始加载 SingletonHolder 并初始化 instance，这样不仅能确保线程安全，也能保证 Singleton 类的唯一性。不过，创建单例更优雅的一种方式是使用枚举。  
  
需要注意的是。第一，静态内部类不能访问外部类的所有成员变量；第二，静态内部类可以访问外部类的所有静态变量，包括私有静态变量。第三，外部类不能声明为 static。”    
  
1. 静态变量
   - 静态变量: 又称为类变量，也就是说这个变量属于类的，类所有的实例都共享静态变量，可以直接通过类名来访问它；静态变量在内存中只存在一份。
   - 实例变量: 每创建一个实例就会产生一个实例变量，它与该实例同生共死。

2. 静态方法

   - 静态方法在类加载的时候就存在了，它不依赖于任何实例。所以静态方法必须有实现，也就是说它不能是抽象方法 (abstract)。

   - 只能访问所属类的静态字段和静态方法，方法中不能有 this 和 super 关键字。

3. 静态语句块
   - 静态语句块在类初始化时运行一次。

4. 静态内部类

   - 非静态内部类依赖于外部类的实例，而静态内部类不需要。

   - 静态内部类不能访问外部类的非静态的变量和方法。

5. 静态导包
   - 在使用静态变量和方法时不用再指明 ClassName，从而简化代码，但可读性大大降低。

#### 初始化顺序

1. 静态属性，静态代码块。

2. 普通属性，普通代码块。

3. 构造方法。

```java
public class InitOrder {

    // 静态属性
    private static String staticField = getStaticField();

    // 静态代码块
    static {
        System.out.println(staticField);
        System.out.println("静态代码块初始化");
    }

    // 普通属性
    private String field = getField();

    // 普通代码块
    {
        System.out.println(field);
        System.out.println("普通代码块初始化");
    }

    // 构造方法
    public InitOrder() {
        System.out.println("构造方法初始化");
    }

    // 静态方法
    public static String getStaticField() {
        String staticFiled = "静态属性初始化";
        return staticFiled;
    }

    // 普通方法
    public String getField() {
        String filed = "普通属性初始化";
        return filed;
    }

    public static void main(String[] argc) {
        new InitOrder();
    }

    /**
     *      静态属性初始化
     *      静态代码块初始化
     *      普通属性初始化
     *      普通代码块初始化
     *      构造方法初始化
     */
}
```



#### 静态方法和变量能否被继承

能

父类 A：

```java
public class A {
    public static String staticStr = "A静态属性";
    public String nonStaticStr = "A非静态属性";
    public static void staticMethod(){
        System.out.println("A静态方法");
    }
    public void nonStaticMethod(){
        System.out.println("A非静态方法");
    }
}
```



子类 B：

```java
public class B extends A{

    public static String staticStr = "B改写后的静态属性";
    public String nonStaticStr = "B改写后的非静态属性";

    public static void staticMethod(){
        System.out.println("B改写后的静态方法");
    }

    @Override
    public void nonStaticMethod() {
        System.out.println("B改写后的非静态方法");
    }
}
```



子类 C：

```java
Public class C extends A{
}
```



测试：

```java
Public class Demo {
    Public static void main (String[] args) {
        C c = new C ();//C 的引用指向 C 的对象
        System.Out.Println (c.nonStaticStr);//A 非静态属性
        System.Out.Println (c.staticStr);//A 静态属性
        c.nonStaticMethod ();//A 非静态方法
        c.staticMethod ();//A 静态方法
        //推出静态属性和静态方法可以被继承

        System.Out.Println ("-------------------------------");

        A c 1 = new C ();//A 的引用指向 C 的对象
        System.Out.Println (c 1. NonStaticStr);//A 非静态属性
        System.Out.Println (c 1. StaticStr);//A 静态属性
        C 1.NonStaticMethod ();//A 非静态方法
        C 1.StaticMethod ();//A 静态方法
        //推出静态属性和静态方法可以被继承

        System.Out.Println ("-------------------------------");
        B b = new B ();//B 的引用指向 B 的对象
        System.Out.Println (b.nonStaticStr);//B 改写后的非静态属性
        System.Out.Println (b.staticStr);//B 改写后的静态属性
        b.nonStaticMethod ();//B 改写后的非静态方法
        b.staticMethod ();//B 改写后的静态方法

        System.Out.Println ("-------------------------------");
        A b 1 = new B ();//A 的引用指向 B 的对象
        System.Out.Println (b 1. NonStaticStr);//A 非静态属性
        System.Out.Println (b 1. StaticStr);//A 静态属性
        B 1.NonStaticMethod ();//B 改写后的非静态方法
        B 1.StaticMethod ();//A 静态方法
        //结果都是父类的静态方法，说明静态方法不可以被重写，不能实现多态
    }
}
```



#### Static 小结

- 子类会继承父类的静态方法和静态变量，但是无法对静态方法进行重写

- 子类中可以直接调用父类的静态方法和静态变量

- 子类可以直接修改（如果父类中没有将静态变量设为 private）静态变量，但这是子类自己的静态变量。

- 子类可以拥有和父类同名的，同参数的静态方法，但是这并不是对父类静态方法的重写，是子类自己的静态方法，子类只是把父类的静态方法隐藏了。

- 当父类的引用指向子类时，使用对象调用静态方法或者静态变量，是调用的父类中的静态方法或者变量（这比较好理解，因为静态方法或变量是属于类的，而引用指向的是一个对象，对象中并不会包含静态的方法和属性）。**也就是说，失去了多态。**

- 当子类的引用指向子类时，使用对象调用静态方法或者静态变量，就是调用的子类中自己的的静态方法或者变量了。



#### 注意

静态变量尤其要注意并发问题。因为静态变量在 Java 中是类级别的变量，它们被所有类的实例共享。由于静态变量是共享资源，当多个线程同时访问和修改静态变量时，就会引发并发问题。

