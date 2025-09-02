---
Title: Java 基础核心要点-基本数据类型-boolean
Category: Java
tags:
  - java
  - 基本数据类型
---
布尔（boolean）仅用于存储两个值：true 和 false，也就是真和假，通常用于条件的判断。代码示例：  
  
```java  
boolean hasMoney = true;  
boolean hasGirlFriend = false;  
```  
  
根据 Java 语言规范，boolean 类型只有两个值 true 和 false，但在语言层面，Java 没有明确规定 boolean 类型的大小。  
  
那经过我的调查，发现有两种论调。  
  
我们先来看论调一。  
  
对于单独使用的 boolean 类型，JVM 并没有提供专用的字节码指令，而是使用 int 相关的指令 istore 来处理，那么 int 明确是 4 个字节，所以此时的 boolean 也占用 4 个字节。  
  
对于作为数组来使用的 boolean 类型，JVM 会按照 byte 的指令来处理（bastore），那么已知 byte 类型占用 1 个字节，所以此时的 boolean 也占用 1 个字节。  
  
论调二，布尔具体占用的大小是不确定的，取决于 JVM 的具体实现。  
  
>boolean: The boolean data type has only two possible values: true and false. Use this data type for simple flags that track true/false conditions. This data type represents one bit of information, but its "size" isn't something that's precisely defined.  
  
可以通过 JOL 工具打印出对象的内存布局，展示 boolean 单独使用和作为数组使用时在内存中的实际占用大小。  
  
```java  
public class BooleanSizeExample {  
    public static void main(String[] args) {        boolean singleBoolean = true;        boolean[] booleanArray = new boolean[10];        // 分析内存占用，可以使用第三方工具如 JOL（Java Object Layout）  
        System.out.println("Size of single boolean: " + org.openjdk.jol.info.ClassLayout.parseInstance(singleBoolean).toPrintable());        System.out.println("Size of boolean array: " + org.openjdk.jol.info.ClassLayout.parseInstance(booleanArray).toPrintable());    }}  
```  
  
运行结果如下（64 操作系统 JDK 8）：  
 ```
Size of single boolean: java. lang. Boolean object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                               VALUE
      0     4           (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4           (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4           (object header)                           dd 20 00 f 8 (11011101 00100000 00000000 11111000) (-134209315)
     12     1   boolean Boolean. value                             true
     13     3           (loss due to the next object alignment)
Instance size: 16 bytes
Space losses: 0 bytes internal + 3 bytes external = 3 bytes total

Size of boolean array: [Z object internals:
 OFFSET  SIZE      TYPE DESCRIPTION                               VALUE
      0     4           (object header)                           01 00 00 00 (00000001 00000000 00000000 00000000) (1)
      4     4           (object header)                           00 00 00 00 (00000000 00000000 00000000 00000000) (0)
      8     4           (object header)                           05 00 00 f 8 (00000101 00000000 00000000 11111000) (-134217723)
     12     4           (object header)                           0 a 00 00 00 (00001010 00000000 00000000 00000000) (10)
     16    10   boolean [Z.<elements>                             N/A
     26     6           (loss due to the next object alignment)
Instance size: 32 bytes
Space losses: 0 bytes internal + 6 bytes external = 6 bytes total
```

对于单个 boolean 变量来说：  
  
①、**对象头（Object Header）** 占用了 12 个字节：  
  
- **OFFSET 0 - 4**：对象头的一部分，包含对象的标记字段（Mark Word），用于存储对象的哈希码、GC 状态等。  
- **OFFSET 4 - 8**：对象头的另一部分，通常是指向类元数据的指针（Class Pointer）。  
- **OFFSET 8 - 12**：对象头的最后一部分，包含锁状态或其他信息。  
  
②、实际的 `boolean` 值占用 1 个字节，也就是**OFFSET 12 - 13**。  
  
③、为了满足 8 字节的对齐要求（HotSpot JVM 默认的对象对齐方式），有 3 个字节的填充。**OFFSET 13 - 16**。  
  
也就是说，尽管 `boolean` 值本身只需要 1 个字节，但由于对象头和对齐要求，一个 `boolean` 在内存中占用 16 字节。  
  
对于 `boolean` 数组来说：  
  
①、**对象头（Object Header）** 占用了 12 个字节：  
  
- **OFFSET 0 - 4**：对象头的一部分，包含对象的标记字段（Mark Word）。  
- **OFFSET 4 - 8**：对象头的另一部分，包含指向类元数据的指针（Class Pointer）。  
- **OFFSET 8 - 12**：对象头的最后一部分，通常包含数组的长度信息。  
  
②、**数组长度** 占用了 4 个字节，此处是 10，**OFFSET 12 - 16**。  
  
③、实际的 `boolean` 数组元素，每个 `boolean` 值占用 1 个字节，总共 10 个字节，**OFFSET 16 - 26**。  
  
④、为了满足 8 字节对齐要求，有 6 个字节的填充，**OFFSET 26 - 32**。  
  
也就是说，每个 `boolean` 数组元素占用 1 个字节，加上对象头、对齐填充和数组长度，包含 10 个元素的 `boolean` 数组占用 32 字节。