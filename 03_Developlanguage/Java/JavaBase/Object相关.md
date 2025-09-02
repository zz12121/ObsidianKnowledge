---
Title: Java 基础核心要点-Object相关
Category: Java
tags:
  - java
  - 数据类型
---
## Object 通用方法

```java
public final native Class<?> getClass()

public native int hashCode()

public boolean equals(Object obj)

protected native Object clone() throws CloneNotSupportedException

public String toString()

public final native void notify()

public final native void notifyAll()

public final native void wait(long timeout) throws InterruptedException

public final void wait(long timeout, int nanos) throws InterruptedException

public final void wait() throws InterruptedException

protected void finalize() throws Throwable {}
```



### Equals 和 hashcode

#### ==

== 是运算符

1. 如果比较的对象是基本数据类型，则比较的是其存储的值是否相等；

2. 如果比较的是引用数据类型，则比较的是所指向对象的地址值是否相等（是否是同一个对象）。

```java
Person p1 = new Person("123");
Person p2 = new Person("123");
int a = 10;
int b = 10;
System.out.println(a == b);//true
System.out.println(p1 == p2); //显然不是同一个对象,false
```



#### Equals

作用是用来判断两个对象是否相等。通过判断两个对象的地址是否相等 (即，是否是同一个对象) 来区分它们是否相等。源码如下：

```java
public boolean equals(Object obj) {
    return (this == obj);
}
```

Equals 方法不能用于比较基本数据类型，如果没有对 equals 方法进行重写，则相当于 == ，比较的是引用类型的变量所指向的对象的地址值。

一般情况下，类会重写 equals 方法用来比较两个对象的内容是否相等。比如 String 类中的 equals () 是被重写了，比较的是对象的值。

#### Hashcode

1. Hashcode 特性体现主要在它的查找效率上，O (1) 的复杂度，在 Set 和 Map 这种使用哈希表结构存储数据的集合中。HashCode 方法的就大大体现了它的价值，主要用于在这些集合中确定对象在整个哈希表中存储的区域。

2. 如果两个对象相同，则这两个对象的 equals 方法返回的值一定为 true，两个对象的 hashCode 方法返回的值也一定相同。(equals 相同，hashcode 一定相同，因为重写的 hashcode 就是计算属性的 hashcode 值)

3. 如果两个对象返回的 HashCode 的值相同，但不能够说明这两个对象的 equals 方法返回的值就一定为 true，只能说明这两个对象在存储在哈希表中的同一个桶中。

#### 只重写了 equals 方法，未重写 hashCode 方法

在 Java 中 equals 方法用于判断两个对象是否相等，而 HashCode 方法在 Java 中主要由于哈希算法中的寻域的功能（也就是寻找数据应该存储的区域的）。在类似于 set 和 map 集合的结构中，Java 为了提高在集合中查询匹配元素的效率问题，引入了哈希算法，通过 HashCode 方法得到对象的 hash 码，再通过 hash 码推算出数据应该存储的位置。然后再进行 equals 操作进行匹配，减少了比较次数，提高了效率。

```java
public class Person {
    String name;

    public Person(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Person person = (Person) o;
        return Objects.equals(name, person.name);
    }

    public static void main(String[] args) {
        Person p1 = new Person("123");
        Person p2 = new Person("123");

        System.out.println(p1 == p2);//false
        System.out.println(p1.hashCode() == p2.hashCode());//false
        System.out.println(p1.equals(p2));//true

        Set<Person> set = new HashSet<>();
        set.add(p1);
        set.add(p2);
        System.out.println(set.size());//2
    }
}
```

- 当只重写了 equals 方法，未重写 hashCode 方法时，equals 方法判断两个对象是否相等时，返回的是 true (第三个输出)，这是因为我们重写 equals 方法时，是对属性的比较；但判断两个对象的 hashCode 值是否相等时，返回的是 false (第二个输出)，在没有重写 hashCode 方法的情况下，调用的是 Object 的 hashCode 方法，返回的是本对象的 hashCode 值，两个对象不一样，因此 hashCode 值不一样。

- 在 set 和 map 中，首先判断两个对象的 hashCode 方法返回的值是否相等，如果相等然后再判断两个对象的 equals 方法，如果 hashCode 方法返回的值不相等，则直接会认为两个对象不相等，不进行 equals 方法的判断。因此在 set 添加对象时，因为 hashCode 值已经不一致，判断出 p 1 和 p 2 是两个对象，都会添加进 set 集合中，因此返回集合中数据个数为 2 (第四个输出)

![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202404250650480.png)

重写 hashCode 方法：重写 hashcode 方法时，一般也是对属性值进行 hash

```java
@Override
public int hashCode() {
    return Objects.hash(name);
}
```

重写了 hashCode 后，其是对属性值的 hash，p 1 和 p 2 的属性值一致，因此 p 1. HashCode () == p 2. HashCode () 为 true，再进行 equals 方法的判断也为 true，认为是一个对象，因此 set 集合中只有一个对象数据。

##### 为什么重写 hashCode 一定也要重写 equals 方法？

如果两个对象的 hashCode 相同，它们是并不一定相同的，因为 equals 方法不相等而 hashCode 方法返回的值却有可能相同的，比如两个不同的对象 hash 到同一个桶中

HashCode 方法实际上是通过一种算法得到一个对象的 hash 码，这个 hash 码是用来确定该对象在哈希表中具体的存储区域的。返回的 hash 码是 int 类型的所以它的数值范围为 [-2147483648 - +2147483647] 之间的，而超过这个范围，实际会产生溢出，溢出之后的值实际在计算机中存的也是这个范围的。比如最大值 2147483647 + 1 之后并不是在计算机中不存储了，它实际在计算机中存储的是-2147483648。在 java 中 hash 码也是通过特定算法得到的，所以很难说在这个范围内情况下不会不产生相同的 hash 码的。也就是说常说的哈希碰撞，因此不同对象可能有相同的 hashCode 的返回值。

**因此 equals 方法返回结果不相等，而 hashCode 方法返回的值却有可能相同！**

##### 为什么重写 equals 一定也要重写 hashCode 方法？

这个是针对 set 和 map 这类使用 hash 值的对象来说的

1. 只重写 equals 方法，不重写 hashCode 方法：

   * 有这样一个场景有两个 Person 对象，可是如果没有重写 hashCode 方法只重写了 equals 方法，equals 方法认为如果两个对象的 name 相同则认为这两个对象相同。这对于 equals 判断对象相等是没问题的。

   * 对于 set 和 map 这类使用 hash 值的对象来说，由于没有重写 hashCode 方法，此时返回的 hash 值是不同的，因此不会去判断重写的 equals 方法，此时也就不会认为是相同的对象。

1. 重写 hashCode 方法不重写 equals 方法

   * 不重写 equals 方法实际是调用 Object 方法中的 equals 方法，判断的是两个对象的堆内地址。而 hashCode 方法认为相等的两个对象在 equals 方法处并不相等。因此也不会认为是用一个对象

   * 因此重写 equals 方法时一定也要重写 hashCode 方法，重写 hashCode 方法时也应该重写 equals 方法。

总结：**对于普通判断对象是否相等来说，只 equals 是可以完成需求的，但是如果使用 set，map 这种需要用到 hash 值的集合时，不重写 hashCode 方法，是无法满足需求的**。尽管如此，也一般建议两者都要重写，几乎没有见过只重写一个的情况



#### 扩展：解决哈希冲突的三种方法

Todo

### Clone ()

Clone 方法是 Java 中拷贝对象的方法

在 Java 中拷贝对象其实是指复制一个与原对象一样的新对象出来，但是在 Java 中赋值 = 是复制对象引用，如果我们想要得到一个对象的副本，使用赋值操作是无法达到目的的。

看下面例子：

```java
Animal a1 = new Animal();
a1.category = "人类";
Animal a2 = a1;
System.out.println(a1==a2);//true
a2.category = "猫科动物";
System.out.println(a1.category);//猫科动物
```

显然，a 1 == a 2 返回为 true，a 1 和 a 2 指向的是同一个引用

并且对对象 a 2 的属性值进行修改后，a 1 的属性值也跟着改变，因此这不是拷贝，要实现拷贝，需要用到 Object 对象的 clone () 方法，实现对对象中各个属性的复制，但它的可见范围是 protected 的

```java
protected native Object clone() throws CloneNotSupportedException;
```

- 所以实体类使用 clone () 方法的前提是：实现 Cloneable 接口，这是一个标记接口，自身没有方法，这算是一种约定。调用 clone 方法时，会去判断有没有实现 Cloneable 接口，没有实现 Cloneable 的话会抛异常 CloneNotSupportedException。

- 覆盖 clone () 方法，可见性提升为 public。

```java
public class Animal implements Cloneable {
    String category;

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    public static void main(String[] args) throws CloneNotSupportedException {
        Animal a1 = new Animal();
        a1.category = "人类";
        Animal a2 = (Animal) a1.clone();
        System.out.println(a1 == a2);
        a2.category = "猫科动物";

        System.out.println(a1.category);
    }
}
//输出：
false
人类
```

拷贝了一个新对象，与原对象引用不同，因此返回 false，并且修改新对象的属性值，旧对象的属性值不改变

#### 浅拷贝

当类中含有引用类型的属性时，新对象和旧对象的引⽤类型属性指向的是同⼀个对象，即为浅拷贝

```java
public class Animal implements Cloneable {
    String category;
    Person person;

    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }

    public static void main(String[] args) throws CloneNotSupportedException {
        Animal a1 = new Animal();
        Person person = new Person("旧对象");
        a1.category = "人类";
        a1.person = person;

        Animal a2 = (Animal) a1.clone();
        a2.category = "新人类";
        a2.person.name = "新对象";

        System.out.println(a1.category + ":" + a1.person.name);//人类：新对象
    }
}
```

如上，改变 a 2 中的引用类型 person. Name = "新对象"后，a 1 的的引用类型 person. Name 也发生了改变。

也就是说，新对象和旧对象的引⽤类型属性指向的是同⼀个对象，因此当改变新对象的引用类型 (person) 的属性值时，旧对象的引用类型的属性值也被改变。因此只是浅拷贝

> 这里可能就有疑问了，String 类型也是引用类型，为什么进行了深拷贝，即如上所示，改变 a 2 的属性值 category，但却没有改变 a 1 的 category？ String 类型有点特殊，它本身没有实现 Cloneable 接口，故根本无法克隆，只能传递引用（注意：Java 只有值传递，只是这里传递是原来引用地址值）。在 clone () 后，克隆后的对象开始也是指向的原来引用地址值 (刚克隆完检查 a 1. Category == a 2. Category 为 true)，但是一旦 String 的值发生改变（String 作为不可更改的类——immutable class，在新赋值的时候，会创建了一个新的对象）就改变了克隆后对象指向的地址，让它指向了一个新的 String 地址，不会影响原对象的指向和值，原来的 String 对象还是指向的它自己的的地址。这样 String 在拷贝的时候就**表现出了深拷贝**的特点

#### 深拷贝

当类中含有引用类型的属性时，新对象和旧对象的引⽤类型属性指向的不是同⼀个对象，即为深拷贝

要实现深拷贝，在 clone 方法中不仅调用了 super. Clone，而且需要调用 Person 对象的 clone 方法（Person 也要实现 Cloneable 接口并重写 clone 方法），从而实现了深拷贝。

```java
public class Animal implements Cloneable {
    String category;
    Person person;

    @Override
    protected Object clone() throws CloneNotSupportedException {
        Animal animal = null;
        animal = (Animal) super.clone();
        animal.person = (Person) person.clone();
        return animal;
    }

    public static void main(String[] args) throws CloneNotSupportedException {
        Animal a1 = new Animal();
        Person person = new Person("旧对象");
        a1.category = "人类";
        a1.person = person;

        Animal a2 = (Animal) a1.clone();
        a2.person.name = "新对象";

        System.out.println(a1.person.name);//旧对象
    }
}
```

可以看到，新对象的引用类型 person 不会再受到旧对象的影响。

但是，在 EffectiveJava 中，反对使用 clone 方法来进行克隆，详情关注[谨慎重写 clone 方法](https://www.seven97.top/books/effectivejava-summary.html#_13、谨慎重写-clone-方法)
