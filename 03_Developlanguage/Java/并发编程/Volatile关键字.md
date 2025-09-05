---
Title: Java 并发编程-Volatile关键字
Category: Java
tags:
  - java
  - 并发编程
---
## volatile 可见性实现
- 当写一个 volatile 变量时，JMM 会把该线程在本地内存中的变量强制刷新到主内存中去；  
- 这个写操作会导致其他线程中的 volatile 变量缓存无效。 

> volatile 变量的内存可见性是基于内存屏障(Memory Barrier)实现:

- 内存屏障，又称内存栅栏，是一个 CPU 指令。
- 在程序运行时，为了提高执行性能，编译器和处理器会对指令进行重排序，JMM 为了保证在不同的编译器和 CPU 上有相同的结果，通过插入特定类型的内存屏障来禁止+ 特定类型的编译器重排序和处理器重排序，插入一条内存屏障会告诉编译器和 CPU：不管什么指令都不能和这条 Memory Barrier 指令重排序。

写一段简单的 Java 代码，声明一个 volatile 变量，并赋值。

```
public class Test {
    private volatile int a;
    public void update() {
        a = 1;
    }
    public static void main(String[] args) {
        Test test = new Test();
        test.update();
    }
}
```

通过 hsdis 和 jitwatch 工具可以得到编译后的汇编代码:

```
......
  0x0000000002951563: and    $0xffffffffffffff87,%rdi
  0x0000000002951567: je     0x00000000029515f8
  0x000000000295156d: test   $0x7,%rdi
  0x0000000002951574: jne    0x00000000029515bd
  0x0000000002951576: test   $0x300,%rdi
  0x000000000295157d: jne    0x000000000295159c
  0x000000000295157f: and    $0x37f,%rax
  0x0000000002951586: mov    %rax,%rdi
  0x0000000002951589: or     %r15,%rdi
  0x000000000295158c: lock cmpxchg %rdi,(%rdx)  //在 volatile 修饰的共享变量进行写操作的时候会多出 lock 前缀的指令
  0x0000000002951591: jne    0x0000000002951a15
  0x0000000002951597: jmpq   0x00000000029515f8
  0x000000000295159c: mov    0x8(%rdx),%edi
  0x000000000295159f: shl    $0x3,%rdi
  0x00000000029515a3: mov    0xa8(%rdi),%rdi
  0x00000000029515aa: or     %r15,%rdi
......
```

lock 前缀的指令在多核处理器下会引发两件事情:

- 将当前处理器缓存行的数据写回到系统内存。
- 写回内存的操作会使在其他 CPU 里缓存了该内存地址的数据无效。

为了提高处理速度，处理器不直接和内存进行通信，而是先将系统内存的数据读到内部缓存(L1，L2 或其他)后再进行操作，但操作完不知道何时会写到内存。

如果对声明了 volatile 的变量进行写操作，JVM 就会向处理器发送一条 lock 前缀的指令，将这个变量所在缓存行的数据写回到系统内存。

为了保证各个处理器的缓存是一致的，实现了缓存一致性协议(MESI)，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。

所有多核处理器下还会完成：当处理器发现本地缓存失效后，就会从内存中重读该变量数据，即可以获取当前最新值。

volatile 变量通过这样的机制就使得每个线程都能获得该变量的最新值。

#### lock 指令

在 Pentium 和早期的 IA-32 处理器中，lock 前缀会使处理器执行当前指令时产生一个 LOCK# 信号，会对总线进行锁定，其它 CPU 对内存的读写请求都会被阻塞，直到锁释放。 后来的处理器，加锁操作是由高速缓存锁代替总线锁来处理。 因为锁总线的开销比较大，锁总线期间其他 CPU 没法访问内存。 这种场景多缓存的数据一致通过缓存一致性协议(MESI)来保证。

#### 缓存一致性

缓存是分段(line)的，一个段对应一块存储空间，称之为缓存行，它是 CPU 缓存中可分配的最小存储单元，大小 32 字节、64 字节、128 字节不等，这与 CPU 架构有关，通常来说是 64 字节。 LOCK# 因为锁总线效率太低，因此使用了多组缓存。 为了使其行为看起来如同一组缓存那样。因而设计了 缓存一致性协议。 缓存一致性协议有多种，但是日常处理的大多数计算机设备都属于 " 嗅探(snooping)" 协议。 所有内存的传输都发生在一条共享的总线上，而所有的处理器都能看到这条总线。 缓存本身是独立的，但是内存是共享资源，所有的内存访问都要经过仲裁(同一个指令周期中，只有一个 CPU 缓存可以读写内存)。 CPU 缓存不仅仅在做内存传输的时候才与总线打交道，而是不停在嗅探总线上发生的数据交换，跟踪其他缓存在做什么。 当一个缓存代表它所属的处理器去读写内存时，其它处理器都会得到通知，它们以此来使自己的缓存保持同步。 只要某个处理器写内存，其它处理器马上知道这块内存在它们的缓存段中已经失效。 
  
## volatile 会禁止指令重排  
  
在讲 JMM 的时候，我们提到了指令重排，相信大家都还有印象，我们来回顾一下重排序需要遵守的规则：  
  
- 重排序不会对存在数据依赖关系的操作进行重排序。比如：`a=1;b=a;` 这个指令序列，因为第二个操作依赖于第一个操作，所以在编译时和处理器运行时这两个操作不会被重排序。  
- 重排序是为了优化性能，但是不管怎么重排序，单线程下程序的执行结果不能被改变。比如：`a=1;b=2;c=a+b` 这三个操作，第一步 (a=1) 和第二步 (b=2) 由于不存在数据依赖关系，所以可能会发生重排序，但是 c=a+b 这个操作是不会被重排序的，因为需要保证最终的结果一定是 c=a+b=3。  
  
使用 volatile 关键字修饰共享变量可以禁止这种重排序。怎么做到的呢？  
  
当我们使用 volatile 关键字来修饰一个变量时，Java 内存模型会插入内存屏障（一个处理器指令，可以对 CPU 或编译器重排序做出约束）来确保以下两点：  
  
- 写屏障（Write Barrier）：当一个 volatile 变量被写入时，写屏障确保在该屏障之前的所有变量的写入操作都提交到主内存。  
- 读屏障（Read Barrier）：当读取一个 volatile 变量时，读屏障确保在该屏障之后的所有读操作都从主内存中读取。  
  
换句话说：  
  
- 当程序执行到 volatile 变量的读操作或者写操作时，在其前面操作的更改肯定已经全部进行，且结果对后面的操作可见；在其后面的操作肯定还没有进行；  
- 在进行指令优化时，不能将 volatile 变量的语句放在其后面执行，也不能把 volatile 变量后面的语句放到其前面执行。  
  
也就是说，执行到 volatile 变量时，其前面的所有语句都必须执行完，后面所有得语句都未执行。且前面语句的结果对 volatile 变量及其后面语句可见。  
  
先看下面未使用 volatile 的代码：  
  
```java  
class ReorderExample {
  int a = 0;
  boolean flag = false;
  public void writer() {
      a = 1;                   //1
      flag = true;             //2
  }
  public void reader() {
      if (flag) {                //3
          int i = a * a;         //4
          System.out.println(i);
      }
  }
}
```  
  
因为重排序影响，所以最终的输出可能是 0，重排序请参考上一篇 JMM 的介绍，如果引入 volatile，我们再看一下代码：  
  
```java  
class ReorderExample {
  int a = 0;
  boolean volatile flag = false;
  public void writer() {
      a = 1;                   //1
      flag = true;             //2
  }
  public void reader() {
      if (flag) {                //3
          int i = a * a;         //4
          System.out.println(i);
      }
  }
}
```  
  
这时候，volatile 会禁止指令重排序，这个过程建立在 happens before 关系的基础上：  
  
1.  根据程序次序规则，1 happens before 2; 3 happens before 4。  
2.  根据 volatile 规则，2 happens before 3。  
3.  根据 happens before 的传递性规则，1 happens before 4。  
  
上述 happens before 关系的图形化表现形式如下：  
  
![[09_Attachments/Java/并发编程/Java_volatile_001.png]]  
  
在上图中，每一个箭头链接的两个节点，代表了一个 happens before 关系:  
  
- 黑色箭头表示程序顺序规则；  
- 橙色箭头表示 volatile 规则；  
- 蓝色箭头表示组合这些规则后提供的 happens before 保证。  
  
这里 A 线程写一个 volatile 变量后，B 线程读同一个 volatile 变量。A 线程在写 volatile 变量之前所有可见的共享变量，在 B 线程读同一个 volatile 变量后，将立即变得对 B 线程可见。  
  
## volatile 不适用的场景  
  
下面是变量自加的示例：  
  
```java  
public class volatileTest {
    public volatile int inc = 0;
    public void increase() {
        inc++;
    }
    public static void main(String[] args) {
        final volatileTest test = new volatileTest();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println("inc output:" + test.inc);
    }
} 
```  
  
测试输出：  
  
```  
inc output:8182  
```  
  
因为 inc++不是一个原子性操作，由读取、加、赋值 3 步组成，所以结果并不能达到 10000。  
  
怎么解决呢？  
  
01、采用 synchronized，把 `inc++` 拎出来单独加 synchronized 关键字：  
  
```java  
public class volatileTest1 {
    public int inc = 0;
    public synchronized void increase() {
        inc++;
    }
    public static void main(String[] args) {
        final volatileTest1 test = new volatileTest1();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println("add synchronized, inc output:" + test.inc);
    }
}
```  
  
02、采用 Lock，通过重入锁 ReentrantLock 对 `inc++` 加锁：  
  
```java  
public class volatileTest2 {
    public int inc = 0;
    Lock lock = new ReentrantLock();
    public void increase() {
        lock.lock();
        inc++;
        lock.unlock();
    }
    public static void main(String[] args) {
        final volatileTest2 test = new volatileTest2();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<1000;j++)
                        test.increase();
                };
            }.start();
        }
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println("add lock, inc output:" + test.inc);
    }
}
```  
  
03、采用原子类 AtomicInteger 来实现：  
  
```java  
public class volatileTest3 {
    public AtomicInteger inc = new AtomicInteger();
    public void increase() {
        inc.getAndIncrement();
    }
    public static void main(String[] args) {
        final volatileTest3 test = new volatileTest3();
        for(int i=0;i<10;i++){
            new Thread(){
                public void run() {
                    for(int j=0;j<100;j++)
                        test.increase();
                };
            }.start();
        }
        while(Thread.activeCount()>1)  //保证前面的线程都执行完
            Thread.yield();
        System.out.println("add AtomicInteger, inc output:" + test.inc);
    }
}
```  
  
三者输出都是 1000，如下：  
  
```  
add synchronized, inc output:1000  
add lock, inc output:1000  
add AtomicInteger, inc output:1000  
```  
  
## volatile 实现单例模式的双重锁  
  
下面是一个使用"双重检查锁定"（double-checked locking）实现的单例模式（Singleton Pattern）的例子。  
  
```java  
public class Penguin {
    private static volatile Penguin m_penguin = null;

    // 一个成员变量 money
    private int money = 10000;

    // 避免通过 new 初始化对象，构造方法应为 private
    private Penguin() {}

    public void beating() {
        System.out.println("打豆豆" + money);
    }

    public static Penguin getInstance() {
        if (m_penguin == null) {
            synchronized (Penguin.class) {
                if (m_penguin == null) {
                    m_penguin = new Penguin();
                }
            }
        }
        return m_penguin;
    }
}
```  
  
在这个例子中，Penguin 类只能被实例化一次。来看代码解释：  
  
- 声明了一个类型为 Penguin 的 volatile 变量 m_penguin，它是类的静态变量，用来存储 Penguin 类的唯一实例。  
- `Penguin()` 构造方法被声明为 private，这样就阻止了外部代码使用 new 来创建 Penguin 实例，保证了只能通过 `getInstance()` 方法获取实例。  
- `getInstance()` 方法是获取 Penguin 类唯一实例的公共静态方法。  
- 第一次 `if (null == m_penguin)` 检查是否已经存在 Penguin 实例。如果不存在，才进入同步代码块。  
- `synchronized(penguin.class)` 对类的 Class 对象加锁，确保在多线程环境下，同时只能有一个线程进入同步代码块。在同步代码块中，再次执行 `if (null == m_penguin)` 检查实例是否已经存在，如果不存在，则创建新的实例。这就是所谓的“双重检查锁定”，一共两次。  
- 最后返回 m_penguin，也就是 Penguin 的唯一实例。  
  
其中，使用 volatile 关键字是为了防止 `m_penguin = new Penguin()` 这一步被指令重排序。因为实际上，`new Penguin()` 这一行代码分为三个子步骤：  
  
- 步骤 1：为 Penguin 对象分配足够的内存空间，伪代码 `memory = allocate()`。  
- 步骤 2：调用 Penguin 的构造方法，初始化对象的成员变量，伪代码 `ctorInstanc(memory)`。  
- 步骤 3：将内存地址赋值给 m_penguin 变量，使其指向新创建的对象，伪代码 `instance = memory`。  
  
如果不使用 volatile 关键字，JVM 可能会对这三个子步骤进行指令重排。  
  
- 为 Penguin 对象分配内存  
- 将对象赋值给引用 m_penguin- 调用构造方法初始化成员变量  
  
这种重排序会导致 m_penguin 引用在对象完全初始化之前就被其他线程访问到。具体来说，如果一个线程执行到步骤 2 并设置了 m_penguin 的引用，但尚未完成对象的初始化，这时另一个线程可能会看到一个“半初始化”的 Penguin 对象。  
  
假如此时有两个线程 A 和 B，要执行 `getInstance()` 方法：  
  
```java  
public static Penguin getInstance() {
    if (m_penguin == null) {
        synchronized (Penguin.class) {
            if (m_penguin == null) {
                m_penguin = new Penguin();
            }
        }
    }
    return m_penguin;
}
```  
  
- 线程 A 执行到 `if (m_penguin == null)`，判断为 true，进入同步块。  
- 线程 B 执行到 `if (m_penguin == null)`，判断为 true，进入同步块。  
  
如果线程 A 执行 `m_penguin = new Penguin()` 时发生指令重排序：  
  
- 线程 A 分配内存并设置引用，但尚未调用构造方法完成初始化。  
- 线程 B 此时判断 `m_penguin != null`，直接返回这个“半初始化”的对象。  
  
这样就会导致线程 B 拿到一个不完整的 Penguin 对象，可能会出现空指针异常或者其他问题。  
  
于是，我们可以为 m_penguin 变量添加 volatile 关键字，来禁止指令重排序，确保对象的初始化完成后再将其赋值给 m_penguin。  
  
## 小结  
  
volatile 可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在 JVM 底层 volatile 是采用“内存屏障”来实现的。  
  
观察加入 volatile 关键字和没有加入 volatile 关键字时所生成的汇编代码就能发现，加入 volatile 关键字时，会多出一个 lock 前缀指令，lock 前缀指令实际上相当于一个内存屏障（也称内存栅栏），内存屏障会提供 3 个功能：  
  
- 它确保指令重排序时不会把其后面的指令排到内存屏障之前的位置，也不会把前面的指令排到内存屏障的后面；即在执行到内存屏障这句指令时，在它前面的操作已经全部完成；  
- 它会强制将对缓存的修改操作立即写入主存；  
- 如果是写操作，它会导致其他 CPU 中对应的缓存行无效。  
  
最后，我们学习了 volatile 不适用的场景，以及解决的方法，并解释了双重检查锁定实现的单例模式为何需要使用 volatile。  
