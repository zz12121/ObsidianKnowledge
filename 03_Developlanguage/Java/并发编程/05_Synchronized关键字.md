---
Title: Java 并发编程-Synchronized关键字
Category: Java
tags:
  - java
  - 并发编程
---
在 Java 中，关键字 synchronized 可以保证在同一个时刻，只有一个线程可以执行某个方法或者某个代码块(主要是对方法或者代码块中存在共享数据的操作)，同时我们还应该注意到 synchronized 的另外一个重要的作用，synchronized 可保证一个线程的变化(主要是共享数据的变化)被其他线程所看到（保证可见性，完全可以替代 [Volatile](03_Developlanguage/Java/并发编程/04_Volatile关键字.md) 功能）。  
  
synchronized 关键字最主要有以下 3 种应用方式：  
  
- 同步方法，为当前对象（this）加锁，进入同步代码前要获得当前对象的锁；  
- 同步静态方法，为当前类加锁（锁的是 Class 对象），进入同步代码前要获得当前类的锁；  
- 同步代码块，指定加锁对象，对给定对象加锁，进入同步代码库前要获得给定对象的锁。  
  
## synchronized同步方法  
  
通过在方法声明中加入 synchronized 关键字，可以保证在任意时刻，只有一个线程能执行该方法。  
  
来看代码：  
  
```java  
public class AccountingSync implements Runnable {
    //共享资源(临界资源)
    static int i = 0;
    // synchronized 同步方法
    public synchronized void increase() {
        i ++;
    }
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
            increase();
        }
    }
    public static void main(String args[]) throws InterruptedException {
        AccountingSync instance = new AccountingSync();
        Thread t1 = new Thread(instance);
        Thread t2 = new Thread(instance);
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println("static, i output:" + i);
    }
}
```  
  
输出结果：  
  
```  
static, i output:2000000 
 ```  
  
如果在方法 `increase()` 前不加 synchronized，因为 i++ 不具备原子性，所以最终结果会小于 2000000，具体分析可以参考《volatile》的内容。  
  
>注意：一个对象只有一把锁，当一个线程获取了该对象的锁之后，其他线程无法获取该对象的锁，所以无法访问该对象的其他 synchronized 方法，但是其他线程还是可以访问该对象的其他非 synchronized 方法。  
  
但是，如果一个线程 A 需要访问对象 obj1 的 synchronized 方法 f1(当前对象锁是 obj1)，另一个线程 B 需要访问对象 obj2 的 synchronized 方法 f2(当前对象锁是 obj2)，这样是允许的：  
  
```java  
public class AccountingSyncBad implements Runnable {
    //共享资源(临界资源)
    static int i = 0;
    // synchronized 同步方法
    public synchronized void increase() {
        i ++;
    }

    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
            increase();
        }
    }

    public static void main(String args[]) throws InterruptedException {
        // new 两个AccountingSync新实例
        Thread t1 = new Thread(new AccountingSyncBad());
        Thread t2 = new Thread(new AccountingSyncBad());
        t1.start();
        t2.start();
        t1.join();
        t2.join();
        System.out.println("static, i output:" + i);
    }
} 
```  
  
输出结果：  
  
```  
static, i output:1224617
 ```  
  
上述代码与前面不同的是，我们创建了两个对象 AccountingSyncBad，然后启动两个不同的线程对共享变量 i 进行操作，但很遗憾，操作结果是 1224617 而不是期望的结果 2000000。  
  
因为上述代码犯了严重的错误，虽然使用了 synchronized 同步 increase 方法，但却 new 了两个不同的对象，这也就意味着存在着两个不同的对象锁，因此 t1 和 t2 都会进入各自的对象锁，也就是说 t1 和 t2 线程使用的是不同的锁，因此线程安全是无法保证的。  
  
> 每个对象都有一个对象锁，不同的对象，他们的锁不会互相影响。  
  
解决这种问题的的方式是将 synchronized 作用于静态的 increase 方法，这样的话，对象锁就锁的是当前的类，由于无论创建多少个对象，类永远只有一个，所有在这样的情况下对象锁就是唯一的。  
  
## synchronized同步静态方法  
  
当 synchronized 同步静态方法时，锁的是当前类的 Class 对象，不属于某个对象。当前类的 Class 对象锁被获取，不影响实例对象锁的获取，两者互不影响，本质上是 this 和 Class 的不同。  
  
由于静态成员变量不专属于任何一个对象，因此通过 Class 锁可以控制静态成员变量的并发操作。  
  
需要注意的是如果线程 A 调用了一个对象的非静态 synchronized 方法，线程 B 需要调用这个对象所属类的静态 synchronized 方法，是不会发生互斥的，因为访问静态 synchronized 方法占用的锁是当前类的 [Class 对象](https://javabetter.cn/basic-extra-meal/fanshe.html)，而访问非静态 synchronized 方法占用的锁是当前对象（this）的锁，看如下代码：  
  
```java  
public class AccountingSyncClass implements Runnable {
    static int i = 0;
    /**
     * 同步静态方法,锁是当前class对象，也就是
     * AccountingSyncClass类对应的class对象
     */
    public static synchronized void increase() {
        i++;
    }
    // 非静态,访问时锁不一样不会发生互斥
    public synchronized void increase4Obj() {
        i++;
    }
    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
            increase();
        }
    }
    public static void main(String[] args) throws InterruptedException {
        //new新实例
        Thread t1=new Thread(new AccountingSyncClass());
        //new新实例
        Thread t2=new Thread(new AccountingSyncClass());
        //启动线程
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
}

输出结果:
2000000

```  
  
由于 synchronized 关键字同步的是静态的 increase 方法，与同步实例方法不同的是，其锁对象是当前类的 Class 对象。  
  
注意代码中的 increase4Obj 方法是实例方法，其对象锁是当前实例对象（this），如果别的线程调用该方法，将不会产生互斥现象，毕竟锁的对象不同，这种情况下可能会发生线程安全问题(操作了共享静态变量 i)。  
  
## synchronized同步代码块  
  
某些情况下，我们编写的方法代码量比较多，存在一些比较耗时的操作，而需要同步的代码块只有一小部分，如果直接对整个方法进行同步，可能会得不偿失，此时我们可以使用同步代码块的方式对需要同步的代码进行包裹。  
  
示例如下：  
  
```java  
public class AccountingSync2 implements Runnable {
    static AccountingSync2 instance = new AccountingSync2(); // 饿汉单例模式

    static int i=0;

    @Override
    public void run() {
        //省略其他耗时操作....
        //使用同步代码块对变量i进行同步操作,锁对象为instance
        synchronized(instance){
            for(int j=0;j<1000000;j++){
                i++;
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
} 
```  
  
输出结果：  
  
```  
2000000
 ```  
  
我们将 synchronized 作用于一个给定的实例对象 instance，即当前实例对象就是锁的对象，当线程进入 synchronized 包裹的代码块时就会要求当前线程持有 instance 实例对象的锁，如果当前有其他线程正持有该对象锁，那么新的线程就必须等待，这样就保证了每次只有一个线程执行 `i++` 操作。  
  
当然除了用 instance 作为对象外，我们还可以使用 this 对象(代表当前实例)或者当前类的 Class 对象作为锁，如下代码：  
  
```java  
//this,当前实例对象锁  
synchronized(this){  
        for(int j=0;j<1000000;j++){i++;  
        }        }//Class对象锁  
synchronized(AccountingSync.class){  
        for(int j=0;j<1000000;j++){i++;  
        }        }  
```  
  
## synchronized与happens before  
  
指令重排我们前面讲 [Java内存模型](03_Developlanguage/Java/并发编程/03_Java内存模型.md) 的时候讲过，这里我们再结合 synchronized 关键字来讲一下。  
  
看下面这段代码：  
  
```java  
class MonitorExample {
    int a = 0;
    public synchronized void writer() {  //1
        a++;                             //2
    }                                    //3
    public synchronized void reader() {  //4
        int i = a;                       //5
        //……
    }                                    //6
}
```  
  
假设线程 A 执行 `writer()` 方法，随后线程 B 执行 `reader()` 方法。根据 happens before 规则，这个过程包含的 happens before 关系可以分为：  
  
- 根据程序次序规则，1 happens before 2, 2 happens before 3; 4 happens before 5, 5 happens before 6。  
- 根据监视器锁规则，3 happens before 4。  
- 根据 happens before 的传递性，2 happens before 5。  
  
>在 Java 内存模型中，监视器锁规则是一种 happens-before 规则，它规定了对一个监视器锁（monitor lock）或者叫做互斥锁的解锁操作 happens-before 于随后对这个锁的加锁操作。简单来说，这意味着在一个线程释放某个锁之后，另一个线程获得同一把锁的时候，前一个线程在释放锁时所做的所有修改对后一个线程都是可见的。  

也就是说，synchronized 会防止临界区内的代码与外部代码发生重排序，`writer()` 方法中 a++ 的执行和 `reader()` 方法中 a 的读取之间存在 happens-before 关系，保证了执行顺序和内存可见性。  

## Synchronized原理分析

### 加锁和释放锁的原理

> 现象、时机(内置锁this)、深入JVM看字节码(反编译看monitor指令)

深入JVM看字节码，创建如下的代码：

```java
public class SynchronizedDemo2 {

    Object object = new Object();
    public void method1() {
        synchronized (object) {

        }
        method2();
    }

    private static void method2() {

    }
}
```

使用javac命令进行编译生成.class文件

```java
>javac SynchronizedDemo2.java
```

使用javap命令反编译查看.class文件的信息

```java
>javap -verbose SynchronizedDemo2.class
```

得到如下的信息：

```
public SynchronizedDemo2();
   descriptor: ()V
   flags: ACC_PUBLIC
   Code:
     stack=3, locals=1, args_size=1
        0: aload_0
        1: invokespecial #1                  // Method java/lang/Object."<init>":()V
        4: aload_0
        5: new           #2                  // class java/lang/Object
        8: dup
        9: invokespecial #1                  // Method java/lang/Object."<init>":()V
       12: putfield      #3                  // Field object:Ljava/lang/Object;
       15: return
     LineNumberTable:
       line 1: 0
       line 3: 4

 public void method1();
   descriptor: ()V
   flags: ACC_PUBLIC
   Code:
     stack=2, locals=3, args_size=1
        0: aload_0
        1: getfield      #3                  // Field object:Ljava/lang/Object;
        4: dup
        5: astore_1
        6: monitorenter    锁开始的位置
        7: aload_1
        8: monitorexit     锁结束的位置
        9: goto          17
       12: astore_2
       13: aload_1
       14: monitorexit     锁结束的位置
       15: aload_2
       16: athrow
       17: invokestatic  #4                  // Method method2:()V
       20: return
```

关注红色方框里的`monitorenter`和`monitorexit`即可。

`Monitorenter`和`Monitorexit`指令，会让对象在执行，使其锁计数器加1或者减1。每一个对象在同一时间只与一个monitor(锁)相关联，而一个monitor在同一时间只能被一个线程获得，一个对象在尝试获得与这个对象相关联的Monitor锁的所有权的时候，monitorenter指令会发生如下3中情况之一：

- monitor计数器为0，意味着目前还没有被获得，那这个线程就会立刻获得然后把锁计数器+1，一旦+1，别的线程再想获取，就需要等待
- 如果这个monitor已经拿到了这个锁的所有权，又重入了这把锁，那锁计数器就会累加，变成2，并且随着重入的次数，会一直累加
- 这把锁已经被别的线程获取了，等待锁释放

`monitorexit指令`：释放对于monitor的所有权，释放过程很简单，就是讲monitor的计数器减1，如果减完以后，计数器不是0，则代表刚才是重入进来的，当前线程还继续持有这把锁的所有权，如果计数器变成0，则代表当前线程不再拥有该monitor的所有权，即释放锁。

下图表现了对象，对象监视器，同步队列以及执行线程状态之间的关系：

![[09_Attachments/Java/并发编程/Java_Synchronized_004.png]]

该图可以看出，任意线程对Object的访问，首先要获得Object的监视器，如果获取失败，该线程就进入同步状态，线程状态变为BLOCKED，当Object的监视器占有者释放后，在同步队列中得线程就会有机会重新获取该监视器。

## synchronized属于可重入锁  
  
从互斥锁的设计上来说，当一个线程试图操作一个由其他线程持有的对象锁的临界资源时，将会处于阻塞状态，但当一个线程再次请求自己持有对象锁的临界资源时，这种情况属于重入锁，请求将会成功。  
  
synchronized 就是可重入锁，因此一个线程调用 synchronized 方法的同时，在其方法体内部调用该对象另一个 synchronized 方法是允许的，如下：  
  
```java  
public class AccountingSync implements Runnable{
    static AccountingSync instance=new AccountingSync();
    static int i=0;
    static int j=0;

    @Override
    public void run() {
        for(int j=0;j<1000000;j++){
            //this,当前实例对象锁
            synchronized(this){
                i++;
                increase();//synchronized的可重入性
            }
        }
    }

    public synchronized void increase(){
        j++;
    }

    public static void main(String[] args) throws InterruptedException {
        Thread t1=new Thread(instance);
        Thread t2=new Thread(instance);
        t1.start();t2.start();
        t1.join();t2.join();
        System.out.println(i);
    }
} 
```  
  
1、AccountingSync 类中定义了一个静态的 AccountingSync 实例 instance 和两个静态的整数 i 和 j，静态变量被所有的对象所共享。  
  
2、在 run 方法中，使用了 `synchronized(this)` 来加锁。这里的锁对象是 this，即当前的 AccountingSync 实例。在锁定的代码块中，对静态变量 i 进行增加，并调用了 increase 方法。  
  
3、increase 方法是一个同步方法，它会对 j 进行增加。由于 increase 方法也是同步的，所以它能在已经获取到锁的情况下被 run 方法调用，这就是 synchronized 关键字的可重入性。  
  
4、在 main 方法中，创建了两个线程 t1 和 t2，它们共享同一个 Runnable 对象，也就是共享同一个 AccountingSync 实例。然后启动这两个线程，并使用 join 方法等待它们都执行完成后，打印 i 的值。  
  
此程序中的 `synchronized(this)` 和 synchronized 方法都使用了同一个锁对象（当前的 AccountingSync 实例），并且对静态变量 i 和 j 进行了增加操作，因此，在多线程环境下，也能保证 i 和 j 的操作是线程安全的。  
  
### 可重入原理：加锁次数计数器

- **什么是可重入？可重入锁**？

**可重入**：（来源于维基百科）若一个程序或子程序可以“在任意时刻被中断然后操作系统调度执行另外一段代码，这段代码又调用了该子程序不会出错”，则称其为可重入（reentrant或re-entrant）的。即当该子程序正在运行时，执行线程可以再次进入并执行它，仍然获得符合设计时预期的结果。与多线程并发执行的线程安全不同，可重入强调对单个线程执行时重新进入同一个子程序仍然是安全的。

**可重入锁**：又名递归锁，是指在同一个线程在外层方法获取锁的时候，再进入该线程的内层方法会自动获取锁（前提锁对象得是同一个对象或者class），不会因为之前已经获取过还没释放而阻塞。

- **看如下的例子**

```
public class SynchronizedDemo {

    public static void main(String[] args) {
        SynchronizedDemo demo =  new SynchronizedDemo();
        demo.method1();
    }

    private synchronized void method1() {
        System.out.println(Thread.currentThread().getId() + ": method1()");
        method2();
    }

    private synchronized void method2() {
        System.out.println(Thread.currentThread().getId()+ ": method2()");
        method3();
    }

    private synchronized void method3() {
        System.out.println(Thread.currentThread().getId()+ ": method3()");
    }
}
```

结合前文中加锁和释放锁的原理，不难理解：

- 执行monitorenter获取锁
    - （monitor计数器=0，可获取锁）
    - 执行method1()方法，monitor计数器+1 -> 1 （获取到锁）
    - 执行method2()方法，monitor计数器+1 -> 2
    - 执行method3()方法，monitor计数器+1 -> 3
- 执行monitorexit命令
    - method3()方法执行完，monitor计数器-1 -> 2
    - method2()方法执行完，monitor计数器-1 -> 1
    - method2()方法执行完，monitor计数器-1 -> 0 （释放了锁）
    - （monitor计数器=0，锁被释放了）

这就是Synchronized的重入性，即在**同一锁程**中，每个对象拥有一个monitor计数器，当线程获取该对象锁后，monitor计数器就会加一，释放锁后就会将monitor计数器减一，线程不需要再次获取同一把锁。
  
我们讲了 synchronized 关键字的基本使用，它能用来同步方法和代码块，那 synchronized 到底锁的是什么呢？随着 JDK 版本的升级，synchronized 又做出了哪些改变呢？“synchronized 性能很差”的谣言真的存在吗？   
  
首先需要明确的一点是：**Java 多线程的锁都是基于对象的**，Java 中的每一个对象都可以作为一个锁。  
  
还有一点需要注意的是，我们常听到的**类锁**其实也是对象锁。  
  
这里再多说几句吧。Class 对象是一种特殊的 Java 对象，代表了程序中的类和接口。Java 中的每个类型（包括类、接口、数组以及基础类型）在 JVM 中都有一个唯一的 Class 对象与之对应。这个 Class 对象被创建的时机是在 JVM 加载类时，由 JVM 自动完成。  
  
Class 对象中包含了与类相关的很多信息，如类的名称、类的父类、类实现的接口、类的构造方法、类的方法、类的字段等等。这些信息通常被称为元数据（metadata）。  
  
可以通过 Class 对象来获取类的元数据，甚至动态地创建类的实例、调用类的方法、访问类的字段等。这就是Java 的反射（Reflection）机制。  
  
所以我们常说的类锁，其实就是 Class 对象的锁。  
  
## 锁的基本用法  
  
`synchronized` 翻译成中文就是“同步”的意思。  
  
我们通常使用`synchronized`关键字来给一段代码或一个方法上锁，我们已经讲过了，这里简单回顾一下，因为 synchronized 真的非常重要，面试常问，开发常用。它通常有以下三种形式：  
  
```java  
// 关键字在实例方法上，锁为当前实例
public synchronized void instanceLock() {
    // code
}

// 关键字在静态方法上，锁为当前Class对象
public static synchronized void classLock() {
    // code
}

// 关键字在代码块上，锁为括号里面的对象
public void blockLock() {
    Object o = new Object();
    synchronized (o) {
        // code
    }
}
```  
  
这里介绍一下“临界区”的概念。所谓“临界区”，指的是某一块代码区域，它同一时刻只能由一个线程执行。在上面的例子中，如果`synchronized`关键字在方法上，那临界区就是整个方法内部。而如果是 synchronized 代码块，那临界区就指的是代码块内部的区域。  
  
通过上面的例子我们可以看到，下面这两个写法其实是等价的作用：  
  
```java  
// 关键字在实例方法上，锁为当前实例
public synchronized void instanceLock() {
    // code
}

// 关键字在代码块上，锁为括号里面的对象
public void blockLock() {
    synchronized (this) {
        // code
    }
}
```  
  
同理，下面这两个方法也应该是等价的：  
  
```java  
// 关键字在静态方法上，锁为当前Class对象
public static synchronized void classLock() {
    // code
}

// 关键字在代码块上，锁为括号里面的对象
public void blockLock() {
    synchronized (this.getClass()) {
        // code
    }
}
```  
  
## 锁的四种状态及锁降级  
  
在 JDK 1.6 以前，所有的锁都是”重量级“锁，因为使用的是操作系统的互斥锁，当一个线程持有锁时，其他试图进入synchronized块的线程将被阻塞，直到锁被释放。涉及到了线程上下文切换和用户态与内核态的切换，因此效率较低。  
  
这也是为什么很多开发者会认为 synchronized 性能很差的原因。  
  
那为了减少获得锁和释放锁带来的性能消耗，JDK 1.6 引入了“偏向锁”和“轻量级锁” 的概念，对 synchronized 做了一次重大的升级，升级后的 synchronized 性能可以说上了一个新台阶。  
  
在 JDK 1.6 及其以后，一个对象其实有四种锁状态，它们级别由低到高依次是：  
  
1. 无锁状态  
2. 偏向锁状态  
3. 轻量级锁状态  
4. 重量级锁状态  
  
无锁就是没有对资源进行锁定，任何线程都可以尝试去修改它，很好理解。  
  
几种锁会随着竞争情况逐渐升级，锁的升级很容易发生，但是锁降级发生的条件就比较苛刻了，锁降级发生在 Stop The World（Java 垃圾回收中的一个重要概念，JVM 篇会细讲）期间，当 JVM 进入安全点的时候，会检查是否有闲置的锁，然后进行降级。  
  
关于锁降级有一点需要说明：  
  
不同于大部分文章说的锁不能降级，实际上 HotSpot JVM 是支持锁降级的。  
  
> In its current implementation, monitor deflation is performed during every STW pause, while all Java threads are waiting at a safepoint. We have seen safepoint cleanup stalls up to 200ms on monitor-heavy-applications。  
  
大致的意思就是重量级锁降级发生于 STW（Stop The World）阶段，降级对象为仅仅能被 VMThread 访问而没有其他 JavaThread 访问的对象。  
  
各种锁的优缺点对比（来自《Java 并发编程的艺术》）：  
  
| 锁       | 优点                                                               | 缺点                                             | 适用场景                             |  
| -------- | ------------------------------------------------------------------ | ------------------------------------------------ | ------------------------------------ |  
| 偏向锁   | 加锁和解锁不需要额外的消耗，和执行非同步方法比仅存在纳秒级的差距。 | 如果线程间存在锁竞争，会带来额外的锁撤销的消耗。 | 适用于只有一个线程访问同步块场景。   |  
| 轻量级锁 | 竞争的线程不会阻塞，提高了程序的响应速度。                         | 如果始终得不到锁竞争的线程使用自旋会消耗 CPU。   | 追求响应时间。同步块执行速度非常快。 |  
| 重量级锁 | 线程竞争不使用自旋，不会消耗 CPU。                                 | 线程阻塞，响应时间缓慢。                         | 追求吞吐量。同步块执行时间较长。     |  
  
## 对象的锁放在什么地方  
  
前面我们提到，Java 的锁都是基于对象的。  
  
首先我们来看看一个对象的“锁”是存放在什么地方的。  
  
每个 Java 对象都有一个对象头。如果是非数组类型，则用 2 个字宽来存储对象头，如果是数组，则会用 3 个字宽来存储对象头。在 32 位处理器中，一个字宽是 32 位；在 64 位虚拟机中，一个字宽是 64 位。对象头的内容如下表所示：  
  
| 长度     | 内容                   | 说明                           |  
| -------- | ---------------------- | ------------------------------ |  
| 32/64bit | Mark Word              | 存储对象的 hashCode 或锁信息等 |  
| 32/64bit | Class Metadata Address | 存储到对象类型数据的指针       |  
| 32/64bit | Array length           | 数组的长度（如果是数组）       |  
  
我们主要来看看 Mark Word 的格式：  
  
| 锁状态   | 29 bit 或 61 bit             | 1 bit 是否是偏向锁？       | 2 bit 锁标志位 |  
| -------- | ---------------------------- | -------------------------- | -------------- |  
| 无锁     |                              | 0                          | 01             |  
| 偏向锁   | 线程 ID                      | 1                          | 01             |  
| 轻量级锁 | 指向栈中锁记录的指针         | 此时这一位不用于标识偏向锁 | 00             |  
| 重量级锁 | 指向互斥量（重量级锁）的指针 | 此时这一位不用于标识偏向锁 | 10             |  
| GC 标记  |                              | 此时这一位不用于标识偏向锁 | 11             |  
  
可以看到，当对象状态为偏向锁时，`Mark Word`存储的是偏向的线程 ID；当状态为轻量级锁时，`Mark Word`存储的是指向线程栈中`Lock Record`的指针；当状态为重量级锁时，`Mark Word`为指向堆中的 monitor（监视器）对象的指针。  
  
>在 Java 中，监视器（monitor）是一种同步工具，用于保护共享数据，避免多线程并发访问导致数据不一致。在 Java 中，每个对象都有一个内置的监视器。  
  
监视器包括两个重要部分，一个是锁，一个是等待/通知机制，后者是通过 Object 类中的`wait()`, `notify()`, `notifyAll()`等方法实现的（我们会在讲Condition和生产者-消费者模式）详细地讲。  
  
下面分别介绍这几种锁以及它们之间是如何升级的。  
  
## 偏向锁  
  
Hotspot 的作者经过以往的研究发现大多数情况下**锁不仅不存在多线程竞争，而且总是由同一线程多次获得**，于是引入了偏向锁。  
  
偏向锁会偏向于第一个访问锁的线程，如果在接下来的运行过程中，该锁没有被其他的线程访问，则持有偏向锁的线程将永远不需要触发同步。也就是说，**偏向锁在资源无竞争情况下消除了同步语句**，连 [CAS](https://javabetter.cn/thread/cas.html)（后面会细讲，戳链接直达） 操作都不做了，着极大地提高了程序的运行性能。  
  
大白话就是对锁设置个变量，如果发现为 true，代表资源无竞争，则无需再走各种加锁/解锁流程。如果为 false，代表存在其他线程竞争资源，那么就会走后面的流程。  
  
### 偏向锁的实现原理  
  
一个线程在第一次进入同步块时，会在对象头和栈帧中的锁记录里存储锁偏向的线程 ID。当下次该线程进入这个同步块时，会去检查锁的 Mark Word 里面是不是放的自己的线程 ID。  
  
如果是，表明该线程已经获得了锁，以后该线程在进入和退出同步块时不需要花费 CAS 操作来加锁和解锁；如果不是，就代表有另一个线程来竞争这个偏向锁。这个时候会尝试使用 CAS 来替换 Mark Word 里面的线程 ID 为新线程的 ID，这个时候要分两种情况：  
  
- 成功，表示之前的线程不存在了， Mark Word 里面的线程 ID 为新线程的 ID，锁不会升级，仍然为偏向锁；  
- 失败，表示之前的线程仍然存在，那么暂停之前的线程，设置偏向锁标识为 0，并设置锁标志位为 00，升级为轻量级锁，会按照轻量级锁的方式进行竞争锁。  
  
CAS 是比较并设置的意思，用于在硬件层面上提供原子性操作。在 在某些处理器架构（如x86）中，比较并交换通过指令 CMPXCHG 实现（（Compare and Exchange），一种原子指令），通过比较是否和给定的数值一致，如果一致则修改，不一致则不修改。  
  
线程竞争偏向锁的过程如下：  
  
![[09_Attachments/Java/并发编程/Java_Synchronized_001.png]]  
  
图中涉及到了 lock record 指针指向当前堆栈中的最近一个 lock record，是轻量级锁按照先来先服务的模式进行了轻量级锁的加锁。  
  
### 撤销偏向锁  
  
偏向锁使用了一种**等到竞争出现才释放锁的机制**，所以当其他线程尝试竞争偏向锁时，持有偏向锁的线程才会释放锁。  
  
偏向锁升级成轻量级锁时，会暂停拥有偏向锁的线程，重置偏向锁标识，这个过程看起来容易，实则开销还是很大的，大概的过程如下：  
  
1. 在一个安全点（在这个时间点上没有字节码正在执行）停止拥有锁的线程。  
2. 遍历线程栈，如果存在锁记录的话，需要修复锁记录和 Mark Word，使其变成无锁状态。  
3. 唤醒被停止的线程，将当前锁升级成轻量级锁。  
  
所以，如果应用程序里所有的锁通常处于竞争状态，那么偏向锁就会是一种累赘，对于这种情况，我们可以一开始就把偏向锁这个默认功能给关闭：  
  
```java  
-XX:UseBiasedLocking=false  
```  
  
下面这个经典的图总结了偏向锁的获得和撤销：  
  
![[09_Attachments/Java/并发编程/Java_Synchronized_002.png]]  
  
## 轻量级锁  
  
多个线程在不同时段获取同一把锁，即不存在锁竞争的情况，也就没有线程阻塞。针对这种情况，JVM 采用轻量级锁来避免线程的阻塞与唤醒。  
  
JVM 会为每个线程在当前线程的栈帧中创建用于存储锁记录的空间，我们称为 Displaced Mark Word。如果一个线程获得锁的时候发现是轻量级锁，会把锁的 Mark Word 复制到自己的 Displaced Mark Word 里面。  
  
然后线程尝试用 CAS 将锁的 Mark Word 替换为指向锁记录的指针。如果成功，当前线程获得锁，如果失败，表示 Mark Word 已经被替换成了其他线程的锁记录，说明在与其它线程竞争锁，当前线程就尝试使用自旋来获取锁。  
  
> 自旋：不断尝试去获取锁，一般用循环来实现。  
  
自旋是需要消耗 CPU 的，如果一直获取不到锁的话，那该线程就一直处在自旋状态，白白浪费 CPU 资源。解决这个问题最简单的办法就是指定自旋的次数，例如让其循环 10 次，如果还没获取到锁就进入阻塞状态。  
  
但是 JDK 采用了更聪明的方式——适应性自旋，简单来说就是线程如果自旋成功了，则下次自旋的次数会更多，如果自旋失败了，则自旋的次数就会减少。  
  
自旋也不是一直进行下去的，如果自旋到一定程度（和 JVM、操作系统相关），依然没有获取到锁，称为自旋失败，那么这个线程会阻塞。同时这个锁就会**升级成重量级锁**。  
  
### 轻量级锁的释放  
  
在释放锁时，当前线程会使用 CAS 操作将 Displaced Mark Word 的内容复制回锁的 Mark Word 里面。如果没有发生竞争，那么这个复制的操作会成功。如果有其他线程因为自旋多次导致轻量级锁升级成了重量级锁，那么 CAS 操作会失败，此时会释放锁并唤醒被阻塞的线程。  
  
一张图说明加锁和释放锁的过程：  
  
![[09_Attachments/Java/并发编程/Java_Synchronized_003.png]]  
  
## 重量级锁  
  
重量级锁依赖于操作系统的互斥锁（mutex，用于保证任何给定时间内，只有一个线程可以执行某一段特定的代码段） 实现，而操作系统中线程间状态的转换需要相对较长的时间，所以重量级锁效率很低，但被阻塞的线程不会消耗 CPU。  
  
前面说到，每一个对象都可以当做一个锁，当多个线程同时请求某个对象锁时，对象锁会设置几种状态用来区分请求的线程：  
  
- Contention List：所有请求锁的线程将被首先放置到该竞争队列  
- Entry List：Contention List 中那些有资格成为候选人的线程被移到 Entry List- Wait Set：那些调用 wait 方法被阻塞的线程被放置到 Wait Set- OnDeck：任何时刻最多只能有一个线程正在竞争锁，该线程称为 OnDeck- Owner：获得锁的线程称为 Owner- !Owner：释放锁的线程  
  
当一个线程尝试获得锁时，如果该锁已经被占用，则会将该线程封装成一个`ObjectWaiter`对象插入到 Contention List 队列的队首，然后调用`park` 方法挂起当前线程。  
  
当线程释放锁时，会从 Contention List 或 EntryList 中挑选一个线程唤醒，被选中的线程叫做`Heir presumptive`即假定继承人，假定继承人被唤醒后会尝试获得锁，但`synchronized`是非公平的，所以假定继承人不一定能获得锁。  
  
这是因为对于重量级锁，如果线程尝试获取锁失败，它会直接进入阻塞状态，等待操作系统的调度。  
  
如果线程获得锁后调用`Object.wait`方法，则会将线程加入到 WaitSet 中，当被`Object.notify`唤醒后，会将线程从 WaitSet 移动到 Contention List 或 EntryList 中去。需要注意的是，当调用一个锁对象的`wait`或`notify`方法时，**如当前锁的状态是偏向锁或轻量级锁则会先膨胀成重量级锁**。  
  
## 锁的升级流程  
  
每一个线程在准备获取共享资源时：  
第一步，检查 MarkWord 里面是不是放的自己的 ThreadId ,如果是，表示当前线程是处于 “偏向锁” 。  
  
第二步，如果 MarkWord 不是自己的 ThreadId，锁升级，这时候，用 CAS 来执行切换，新的线程根据 MarkWord 里面现有的 ThreadId，通知之前线程暂停，之前线程将 Markword 的内容置为空。  
  
第三步，两个线程都把锁对象的 HashCode 复制到自己新建的用于存储锁的记录空间，接着开始通过 CAS 操作，  
把锁对象的 MarKWord 的内容修改为自己新建的记录空间的地址的方式竞争 MarkWord。  
  
第四步，第三步中成功执行 CAS 的获得资源，失败的则进入自旋 。  
  
第五步，自旋的线程在自旋过程中，成功获得资源(即之前获的资源的线程执行完成并释放了共享资源)，则整个状态依然处于 轻量级锁的状态，如果自旋失败 。  
  
第六步，进入重量级锁的状态，这个时候，自旋的线程进行阻塞，等待之前线程执行完成并唤醒自己。  

## 锁优化
### 锁消除

锁消除是指虚拟机即时编译器再运行时，对一些代码上要求同步，但是被检测到不可能存在共享数据竞争的锁进行消除。锁消除的主要判定依据来源于逃逸分析的数据支持。意思就是：JVM会判断再一段程序中的同步明显不会逃逸出去从而被其他线程访问到，那JVM就把它们当作栈上数据对待，认为这些数据是线程独有的，不需要加同步。此时就会进行锁消除。

​当然在实际开发中，我们很清楚的知道哪些是线程独有的，不需要加同步锁，但是在Java API中有很多方法都是加了同步的，那么此时JVM会判断这段代码是否需要加锁。如果数据并不会逃逸，则会进行锁消除。比如如下操作：在操作String类型数据时，由于String是一个不可变类，对字符串的连接操作总是通过生成的新的String对象来进行的。因此Javac编译器会对String连接做自动优化。在JDK 1.5之前会使用StringBuffer对象的连续append()操作，在JDK 1.5及以后的版本中，会转化为StringBuidler对象的连续append()操作。

```java
public static String test03(String s1, String s2, String s3) {
    String s = s1 + s2 + s3;
    return s;
}
```

上述代码使用javap 编译结果

![[09_Attachments/Java/并发编程/Java_Synchronized_005.png]]

众所周知，StringBuilder不是安全同步的，但是在上述代码中，JVM判断该段代码并不会逃逸，则将该代码带默认为线程独有的资源，并不需要同步，所以执行了锁消除操作。(还有Vector中的各种操作也可实现锁消除。在没有逃逸出数据安全防卫内)

### 锁粗化

​原则上，我们都知道在加同步锁时，尽可能的将同步块的作用范围限制到尽量小的范围(只在共享数据的实际作用域中才进行同步，这样是为了使得需要同步的操作数量尽可能变小。在存在锁同步竞争中，也可以使得等待锁的线程尽早的拿到锁)。

​大部分上述情况是完美正确的，但是如果存在连串的一系列操作都对同一个对象反复加锁和解锁，甚至加锁操作时出现在循环体中的，那即使没有线程竞争，频繁的进行互斥同步操作也会导致不必要的性能操作。

这里贴上根据上述Javap 编译的情况编写的实例java类

```java
public static String test04(String s1, String s2, String s3) {
    StringBuffer sb = new StringBuffer();
    sb.append(s1);
    sb.append(s2);
    sb.append(s3);
    return sb.toString();
}
```

​在上述的连续append()操作中就属于这类情况。JVM会检测到这样一连串的操作都是对同一个对象加锁，那么JVM会将加锁同步的范围扩展(粗化)到整个一系列操作的 外部，使整个一连串的append()操作只需要加锁一次就可以了。
  
## 小结  
  
记住 synchronized 的三种应用方式，指令重排情况分析，以及 synchronized 的可重入性，通过今天的学习，你基本可以掌握 synchronized 的使用姿势了。  
  
同步会带来一定的性能开销，因此需要合理使用。不应将整个方法或者更大范围的代码块做同步，而应尽可能地缩小同步范围。  
  
在 JVM 的早期版本中，synchronized 是重量级的，因为线程阻塞和唤醒需要操作系统的介入。但在 JVM 的后续版本中，对 synchronized 进行了大量优化，如偏向锁、轻量级锁和适应性自旋等，所以现在的 synchronized 并不一定是重量级的，其性能在许多情况下都很好，可以大胆地用。  
  
- Java 中的每一个对象都可以作为一个锁，Java 中的锁都是基于对象的。  
- synchronized 关键字可以用来修饰方法和代码块，它可以保证在同一时刻最多只有一个线程执行该段代码。  
- synchronized 关键字在修饰方法时，锁为当前实例对象；在修饰静态方法时，锁为当前 Class 对象；在修饰代码块时，锁为括号里面的对象。  
- Java 6 为了减少获得锁和释放锁带来的性能消耗，引入了“偏向锁”和“轻量级锁“。在 Java 6 以前，所有的锁都是”重量级“锁。所以在 Java 6 及其以后，一个对象其实有四种锁状态，它们级别由低到高依次是：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态。  
- 偏向锁会偏向于第一个访问锁的线程，如果在接下来的运行过程中，该锁没有被其他的线程访问，则持有偏向锁的线程将永远不需要触发同步。也就是说，偏向锁在资源无竞争情况下消除了同步语句，连 CAS 操作都不做了，提高了程序的运行性能。  
- 轻量级锁是通过 CAS 操作和自旋来实现的，如果自旋失败，则会升级为重量级锁。  
- 重量级锁依赖于操作系统的互斥量（mutex） 实现的，而操作系统中线程间状态的转换需要相对较长的时间，所以重量级锁效率很低，但被阻塞的线程不会消耗 CPU。  
  
----