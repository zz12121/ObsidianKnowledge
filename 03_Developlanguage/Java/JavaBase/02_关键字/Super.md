---
Title: Java 基础核心要点-关键字-Super
Category: Java
tags:
  - java
  - 关键字
---
### 01、super 关键字  
  
super 关键字的用法主要有三种。  
  
- 指向父类对象；  
- 调用父类的方法；  
- `super()` 可以调用父类的构造方法。  
  
其实和 this 有些相似，只不过用意不大相同。每当创建一个子类对象的时候，也会隐式的创建父类对象，由 super 关键字引用。  
  
如果父类和子类拥有同样名称的字段，super 关键字可以用来访问父类的同名字段。  
  
```java  
public class ReferParentField {
    public static void main(String[] args) {
        new Dog().printColor();
    }
}

class Animal {
    String color = "白色";
}

class Dog extends Animal {
    String color = "黑色";

    void printColor() {
        System.out.println(color);
        System.out.println(super.color);
    }
}
```  
  
父类 Animal 中有一个名为 color 的字段，子类 Dog 中也有一个名为 color 的字段，子类的 `printColor()` 方法中，通过 super 关键字可以访问父类的 color。  
  
来看一下输出结果。  
  
```  
黑色  
白色  
```  
  
当子类和父类的方法名相同时，可以使用 super 关键字来调用父类的方法。换句话说，super 关键字可以用于方法重写时访问到父类的方法。  
  
  
```java  
public class ReferParentMethod {
    public static void main(String[] args) {
        new Dog().work();
    }
}

class Animal {
    void eat() {
        System.out.println("吃...");
    }
}

class Dog extends Animal {
    @Override
    void eat() {
        System.out.println("吃...");
    }

    void bark() {
        System.out.println("汪汪汪...");
    }

    void work() {
        super.eat();
        bark();
    }
}   
```  
  
父类 Animal 和子类 Dog 中都有一个名为 `eat()` 的方法，通过 `super.eat()` 可以访问到父类的 `eat()` 方法。  
  
```java  
public class ReferParentConstructor {
    public static void main(String[] args) {
        new Dog();
    }
}

class Animal {
    Animal(){
        System.out.println("动物来了");
    }
}

class Dog extends Animal {
    Dog() {
        super();
        System.out.println("狗狗来了");
    }
}
```  
  
子类 Dog 的构造方法中，第一行代码为 `super()`，它就是用来调用父类的构造方法的。  
  
来看一下输出结果。  
  
```  
动物来了  
狗狗来了  
```  
  
当然了，在默认情况下，`super()` 是可以省略的，编译器会主动去调用父类的构造方法。也就是说，子类即使不使用 `super()` 主动调用父类的构造方法，父类的构造方法仍然会先执行。  
  
```java  
public class ReferParentConstructor {
    public static void main(String[] args) {
        new Dog();
    }
}

class Animal {
    Animal(){
        System.out.println("动物来了");
    }
}

class Dog extends Animal {
    Dog() {
        System.out.println("狗狗来了");
    }
}
```  
  
输出结果和之前一样。  
  
```  
动物来了  
狗狗来了  
```  
  
`super()` 也可以用来调用父类的有参构造方法，这样可以提高代码的可重用性。  
  
```java  
class Person {
    int id;
    String name;

    Person(int id, String name) {
        this.id = id;
        this.name = name;
    }
}

class Emp extends Person {
    float salary;

    Emp(int id, String name, float salary) {
        super(id, name);
        this.salary = salary;
    }

    void display() {
        System.out.println(id + " " + name + " " + salary);
    }
}

public class CallParentParamConstrutor {
    public static void main(String[] args) {
        new Emp(1, "沉默王二", 20000f).display();
    }
} 
```  
  
Emp 类继承了 Person 类，也就继承了 id 和 name 字段，当在 Emp 中新增了 salary 字段后，构造方法中就可以使用 `super(id, name)` 来调用父类的有参构造方法。  
  
来看一下输出结果。  
  
```  
1 呵呵 20000.0
```  
