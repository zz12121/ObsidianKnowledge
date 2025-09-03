---
Title: Java 基础核心要点-关键字-This
Category: Java
tags:
  - java
  - 关键字
---
可用于在方法或构造方法中引用当前对象。  
  
```java  
public class MyClass {
    private int num;

    public MyClass(int num) {
        this.num = num; // 使用 this 关键字引用当前对象的成员变量
    }

    public void doSomething() {
        System.out.println("Doing something with " + this.num); // 使用 this 关键字引用当前对象的成员变量
    }

    public MyClass getThis() {
        return this; // 返回当前对象本身
    }
} 
```  
  
在这个示例中，MyClass 类有一个私有成员变量 num，并定义了一个构造方法、一个方法和一个返回当前对象的方法。在构造方法中，使用 this 关键字引用当前对象的成员变量，并将传入的参数赋值给该成员变量。在方法 `doSomething()` 中，使用 this 关键字引用当前对象的成员变量，并输出该成员变量的值。在方法 `getThis()` 中，直接返回当前对象本身。  

this 关键字有很多种用法，其中最常用的一个是，它可以作为引用变量，指向当前对象。除此之外， this 关键字还可以完成以下工作。  
  
- 调用当前类的方法；  
- `this()` 可以调用当前类的构造方法；  
- this 可以作为参数在方法中传递；  
- this 可以作为参数在构造方法中传递；  
- this 可以作为方法的返回值，返回当前类的对象。  
  
### 01、 指向当前对象  
  
```java  
public class WithoutThisStudent {
    String name;
    int age;

    WithoutThisStudent(String name, int age) {
        name = name;
        age = age;
    }

    void out() {
        System.out.println(name+" " + age);
    }

    public static void main(String[] args) {
        WithoutThisStudent s1 = new WithoutThisStudent("呵呵", 18);
        WithoutThisStudent s2 = new WithoutThisStudent("呵呵", 16);

        s1.out();
        s2.out();
    }
}```  
  
在上面的例子中，构造方法的参数名和实例变量名相同，由于没有使用 this 关键字，所以无法为实例变量赋值。  
  
来看一下程序的输出结果。  
  
```  
null 0  
null 0  
```  
  
从结果中可以看得出来，尽管创建对象的时候传递了参数，但实例变量并没有赋值。这是因为如果构造方法中没有使用 this 关键字的话，name 和 age 指向的并不是实例变量而是参数本身。  
  
那怎么解决这个问题呢？  
  
```java  
public class WithThisStudent {
    String name;
    int age;

    WithThisStudent(String name, int age) {
        this.name = name;
        this.age = age;
    }

    void out() {
        System.out.println(name+" " + age);
    }

    public static void main(String[] args) {
        WithThisStudent s1 = new WithThisStudent("呵呵", 18);
        WithThisStudent s2 = new WithThisStudent("呵呵", 16);

        s1.out();
        s2.out();
    }
}
```  
  
再来看一下程序的输出结果。  
  
```  
呵呵 18  
呵呵 16  
```  
  
这次，实例变量有值了，在构造方法中，`this.xxx` 指向的就是实例变量，而不再是参数本身了。当然了，如果参数名和实例变量名不同的话，就不必使用 this 关键字，但我建议使用 this 关键字，这样的代码更有意义。  
  
### 02、调用当前类的方法  
  
```java  
public class InvokeCurrentClassMethod {
    void method1() {}
    void method2() {
        method1();
    }

    public static void main(String[] args) {
        new InvokeCurrentClassMethod().method1();
    }
} 
```  
  
上面这段代码中没有见到 this 关键字吧？  
  
在 classes 目录下找到 InvokeCurrentClassMethod.class 文件，然后双击打开（IDEA 默认会使用 FernFlower 打开字节码文件）。  
  
```java  
public class InvokeCurrentClassMethod {
    public InvokeCurrentClassMethod() {
    }

    void method1() {
    }

    void method2() {
        this.method1();
    }

    public static void main(String[] args) {
        (new InvokeCurrentClassMethod()).method1();
    }
}
```  
  
`this` 关键字是不是出现了？  
  
我们可以在一个类中使用 this 关键字来调用另外一个方法，如果没有使用的话，编译器会自动帮我们加上。在源代码中，`method2()` 在调用 `method1()` 的时候并没有使用 this 关键字，但通过反编译后的字节码可以看得到。  
  
### 03、调用当前类的构造方法  
  
“再来看下面这段代码。”  
  
```java  
public class InvokeConstrutor {
    InvokeConstrutor() {
        System.out.println("hello");
    }

    InvokeConstrutor(int count) {
        this();
        System.out.println(count);
    }

    public static void main(String[] args) {
        InvokeConstrutor invokeConstrutor = new InvokeConstrutor(10);
    }
}
```  
  
在有参构造方法 `InvokeConstrutor(int count)` 中，使用了 `this()` 来调用无参构造方法 `InvokeConstrutor()`。`this()` 可用于调用当前类的构造方法——构造方法可以重用了。  
  
来看一下输出结果。  
  
```  
hello  
10  
```  
  
无参构造方法也被调用了，所以程序输出了 hello。  
  
也可以在无参构造方法中使用 `this()` 并传递参数来调用有参构造方法。  
  
```java  
public class InvokeParamConstrutor {
    InvokeParamConstrutor() {
        this(10);
        System.out.println("hello");
    }

    InvokeParamConstrutor(int count) {
        System.out.println(count);
    }

    public static void main(String[] args) {
        InvokeParamConstrutor invokeConstrutor = new InvokeParamConstrutor();
    }
}
```  
  
再来看一下程序的输出结果。  
  
```  
10  
hello  
```  
  
不过，需要注意的是，`this()` 必须放在构造方法的第一行，否则就报错了。
  
### 04、作为参数在方法中传递  
  
```java  
public class ThisAsParam {
    void method1(ThisAsParam p) {
        System.out.println(p);
    }

    void method2() {
        method1(this);
    }

    public static void main(String[] args) {
        ThisAsParam thisAsParam = new ThisAsParam();
        System.out.println(thisAsParam);
        thisAsParam.method2();
    }
}
```  
  
`this` 关键字可以作为参数在方法中传递，此时，它指向的是当前类的对象。  
  
来看一下输出结果:  
  
```  
com.itwanger.twentyseven.ThisAsParam@77459877  
com.itwanger.twentyseven.ThisAsParam@77459877  
```  
  
`method2()` 调用了 `method1()`，并传递了参数 this，`method1()` 中打印了当前对象的字符串。 `main()` 方法中打印了 thisAsParam 对象的字符串。从输出结果中可以看得出来，两者是同一个对象。  
  
### 05、作为参数在构造方法中传递  
  
```java  
public class ThisAsConstrutorParam {
    int count = 10;

    ThisAsConstrutorParam() {
        Data data = new Data(this);
        data.out();
    }

    public static void main(String[] args) {
        new ThisAsConstrutorParam();
    }
}

class Data {
    ThisAsConstrutorParam param;
    Data(ThisAsConstrutorParam param) {
        this.param = param;
    }

    void out() {
        System.out.println(param.count);
    }
}
```  
  
在构造方法 `ThisAsConstrutorParam()` 中，我们使用 this 关键字作为参数传递给了 Data 对象，它其实指向的就是 `new ThisAsConstrutorParam()` 这个对象。  
  
`this` 关键字也可以作为参数在构造方法中传递，它指向的是当前类的对象。当我们需要在多个类中使用一个对象的时候，这非常有用。  
  
来看一下输出结果:  
  
```  
10  
```  
  
### 06、作为方法的返回值  
  
```java  
public class ThisAsMethodResult {
    ThisAsMethodResult getThisAsMethodResult() {
        return this;
    }
    
    void out() {
        System.out.println("hello");
    }

    public static void main(String[] args) {
        new ThisAsMethodResult().getThisAsMethodResult().out();
    }
}
```  
  
`getThisAsMethodResult()` 方法返回了 this 关键字，指向的就是 `new ThisAsMethodResult()` 这个对象，所以可以紧接着调用 `out()` 方法——达到了链式调用的目的，这也是 this 关键字非常经典的一种用法。  
  
链式调用的形式在 JavaScript 代码更加常见。  
  
这里`getThisAsMethodResult()` 方法的返回值，需要注意的是，`this` 关键字作为方法的返回值的时候，方法的返回类型为类的类型。  
  
“来看一下输出结果。”  
  
```  
hello  
```  
