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

#todo