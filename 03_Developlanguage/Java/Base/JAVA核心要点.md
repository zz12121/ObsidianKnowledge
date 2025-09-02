---
Title: Java 基础核心要点
Category: Java
Tags:
Head:
  - - meta
    - Name: keywords
      Content: Integer 缓存, BigDecimal, String, Object, 数组, hashcode, equals, 哈希冲突, final, static, trancient, 值传递, 引用传递, 序列化和反序列化, LocalDate
  - - meta
    - Name: description
      Content: 全网最全的 Java 知识点总结，让天下没有难学的八股文！
---
## Integer 缓存池

这里引申出一个经典问题，看下面代码

```java
Integer a = 100;

Integer b = 100;
System.out.println(a == b);//true

Integer c = 200;
Integer d = 200;
System.out.println(c == d);//false
```

为什么第一个输出的是 true，第二个输出的是 false？

Integer a = 100 的这种直接赋值操作，是调⽤Integer.ValueOf (100) 方法，从 Integer.ValueOf () 源码可以看到，返回的是 Integer 对象，但这里的实现并不是简单的 new Integer，而是先判断 i 这个值是否在 IntegerCache 范围内，如果在，直接返回 IntegerCache 中的值，如果不在则 new Integer

```java
public static Integer valueOf(int i) {
    if (i >= IntegerCache.low && i <= IntegerCache.high)
        return IntegerCache.cache[i + (-IntegerCache.low)];
    return new Integer(i);
}

 private static class IntegerCache {
    static final int low = -128;
    static final int high;
    static final Integer cache[];

    static {
        // high value may be configured by property
        int h = 127;
        String integerCacheHighPropValue =
            sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
        if (integerCacheHighPropValue != null) {
            try {
                int i = parseInt(integerCacheHighPropValue);
                i = Math.max(i, 127);
                // Maximum array size is Integer.MAX_VALUE
                h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
            } catch( NumberFormatException nfe) {
                // If the property cannot be parsed into an int, ignore it.
            }
        }
        high = h;

        cache = new Integer[(high - low) + 1];
        int j = low;
        for(int k = 0; k < cache.length; k++)
            cache[k] = new Integer(j++);

        // range [-128, 127] must be interned (JLS7 5.1.7)
        assert IntegerCache.high >= 127;
    }

    private IntegerCache() {}
}
```

从源码可以看到，默认 Integer cache 的下限是-128，上限默认 127。当赋值 100 给 Integer 时，刚好在这个范围内，所以从 cache 中取对应的 Integer 并返回，所以 a 和 b 返回的是同一个对象，所以比较是相等的，当赋值 200 给 Integer 时，不在 cache 的范围内，所以会 new Integer 并返回，当然比较的结果是不相等的。

> 扩展：Byte，Short，Integer，Long 这 4 种包装类默认创建了数值 [-128，127] 的相应类型的缓存数据，Character 创建了数值在 [0,127] 范围的缓存数据，Boolean 直接返回 True or False

```java
System.out.println(Integer.valueOf(-128) == Integer.valueOf(-128));//1.true
System.out.println(Integer.valueOf(127) == Integer.valueOf(127));//2.true
System.out.println(Integer.valueOf(128) == Integer.valueOf(128));//3.false
System.out.println(Integer.parseInt("128") == Integer.valueOf(128));//4.true
```

1、2、3 都好理解，缓存范围是 [-128，127]，1、2 都在范围内，返回的是缓存中的对象，因此输出 true，3 不在范围内，返回的是新 new 的 Integer，因此输出 false。

那为什么 4 输出的是 true 呢？ 128 在缓存范围外，按道理会 new 出一个 Integer 对象，为什么输出 true 呢？

- 首先 Integer. ParseInt 方法返回的是 int 基本数据类型，不是对象，也就是说 Integer.ParseInt ("128") = 128
  ```java
  public static int parseInt(String s) throws NumberFormatException {
      return parseInt(s,10);
  }
  ```

- 当进行比较 == 运算时，会进行自动拆箱，也就是说 Integer.ValueOf (128) 生成的 Integer 会自动拆箱成 128，那么比较两个相等的额数值自然是 true 的

**注意**：使用 == 运算符时，需要一边是基本数据类型才会自动拆箱，如果两边都是引用数据类型，是不会自动拆箱的。

> 当基础类型与它们的包装类有如下几种情况时，编译器会自动进行装箱或拆箱：
>
> - 赋值操作（装箱或拆箱）
>- 进行加减乘除混合运算 （拆箱）
> - 进行>,<,>=,<=,== 比较运算（拆箱）
>- 调用 equals 进行比较（装箱）
> - ArrayList、HashMap 等集合类添加基础类型数据时（装箱）

注意：三目运算符 condition ? 表达式 1：表达式 2 中，高度注意表达式 1 和 2 在类型对齐时，可能抛出因自动拆箱导致的 NPE 异常

1. 表达式 1 或表达式 2 的值只要有一个是原始类型。
2. 表达式 1 或表达式 2 的值的类型不一致，会强制拆箱升级成表示范围更大的那个类型。

```java
Integer a = 1;
Integer b = 2;
Integer c = null;
Boolean flag = false;
// a*b 的结果是 int 类型，那么 c 会强制拆箱成 int 类型，抛出 NPE 异常
Integer result = (flag ? a * b : c);
```

缓存机制存在的原因：将频繁被使用的对象缓存起来，可以提升读取的效率，这是一个典型的用空间换时间的例子（其实缓存机制都是这个原理），而 Java 开发者认为[-128，127]是比较常使用的范围。


## BigDecimal

[阿里巴巴 Java 开发手册 (黄山版)](10_SCENE/Norms/阿里巴巴%20Java%20开发手册%20(黄山版).md) 中提到：“浮点数之间的等值判断，基本数据类型不能用 == 来比较，包装数据类型不能用 equals 来判断”。“为了避免精度丢失，可以使用 `BigDecimal` 来进行浮点数的运算”。

浮点数的运算竟然还会有精度丢失的风险吗？确实会！

示例代码：

```java
float a = 2.0f - 1.9f;
float b = 1.8f - 1.7f;
System.out.println(a);// 0.100000024
System.out.println(b);// 0.099999905
System.out.println(a == b);// false
```

**为什么浮点数 `float` 或 `double` 运算的时候会有精度丢失的风险呢？**

这个和计算机保存浮点数的机制有很大关系。我们知道计算机是二进制的，而且计算机在表示一个数字时，宽度是有限的，无限循环的小数存储在计算机时，只能被截断，所以就会导致小数精度发生损失的情况。这也就是解释了为什么浮点数没有办法用二进制精确表示。

就比如说十进制下的 0.2 就没办法精确转换成二进制小数：

```java
// 0.2 转换为二进制数的过程为，不断乘以 2，直到不存在小数为止，
// 在这个计算过程中，得到的整数部分从上到下排列就是二进制的结果。
0.2 * 2 = 0.4 -> 0
0.4 * 2 = 0.8 -> 0
0.8 * 2 = 1.6 -> 1
0.6 * 2 = 1.2 -> 1
0.2 * 2 = 0.4 -> 0（发生循环）
...
```

关于浮点数的更多内容，建议看一下[计算机系统基础（四）浮点数](http://kaito-kidd.com/2018/08/08/computer-system-float-point/)这篇文章。

### BigDecimal 介绍

`BigDecimal` 可以实现对浮点数的运算，不会造成精度丢失。

通常情况下，大部分需要浮点数精确运算结果的业务场景（比如涉及到钱的场景）都是通过 `BigDecimal` 来做的。

想要解决浮点数运算精度丢失这个问题，可以直接使用 `BigDecimal` 来定义浮点数的值，然后再进行浮点数的运算操作即可。

```java
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("0.9");
BigDecimal c = BigDecimal.valueOf(0.8);

BigDecimal x = a.subtract(b);
BigDecimal y = b.subtract(c);

System.out.println(x.compareTo(y));// 0
```

### BigDecimal 常见方法

#### 创建

在使用 `BigDecimal` 时，为了防止精度丢失，推荐使用它的 `BigDecimal(String val)` 构造方法或者 `BigDecimal.valueOf(double val)` 静态方法来创建对象。

《阿里巴巴 Java 开发手册》对这部分内容也有提到，如下图所示。

![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202406202139919.png)

```java
public static BigDecimal valueOf(double val) {
    return new BigDecimal(Double.toString(val));
}
```



#### 加减乘除

- `add` 方法：两个 `BigDecimal` 对象相加
- `subtract` 方法：两个 `BigDecimal` 对象相减
- `multiply` 方法：两个 `BigDecimal` 对象相乘
- `divide` 方法：两个 `BigDecimal` 对象相除。

```java
BigDecimal a = new BigDecimal("1.0");
BigDecimal b = new BigDecimal("0.9");
System.out.println(a.add(b));// 1.9
System.out.println(a.subtract(b));// 0.1
System.out.println(a.multiply(b));// 0.90
System.out.println(a.divide(b));// 无法除尽，抛出 ArithmeticException 异常
System.out.println(a.divide(b, 2, RoundingMode.HALF_UP));// 1.11
```

这里需要注意的是，在使用 `divide` 方法的时候尽量使用 3 个参数版本，并且 `RoundingMode` 不要选择 `UNNECESSARY`，否则很可能会遇到 `ArithmeticException`（无法除尽出现无限循环小数的时候），其中 `scale` 表示要保留几位小数，`roundingMode` 代表保留规则。

```java
public BigDecimal divide(BigDecimal divisor, int scale, RoundingMode roundingMode) {
    return divide(divisor, scale, roundingMode.oldMode);
}
```



#### 保留几位小数 setScale

通过 `setScale` 方法设置保留几位小数以及保留规则。保留规则如上，不需要记，IDEA 会提示。

```java
BigDecimal m = new BigDecimal("1.255433");
BigDecimal n = m.setScale(3, RoundingMode.HALF_DOWN);
System.out.println(n);// 1.255
```

保留规则非常多，这里列举几种:

```java
public enum RoundingMode {
   // 2.5 -> 3 , 1.6 -> 2
   // -1.6 -> -2 , -2.5 -> -3
   UP(BigDecimal.ROUND_UP),//远离零方向舍入，无论正负
   // 2.5 -> 2 , 1.6 -> 1
   // -1.6 -> -1 , -2.5 -> -2
   DOWN(BigDecimal.ROUND_DOWN),//向零方向舍入，直接去掉小数部分。
   // 2.5 -> 3 , 1.6 -> 2
   // -1.6 -> -1 , -2.5 -> -2
   CEILING(BigDecimal.ROUND_CEILING),//向正无穷方向舍入。
   // 2.5 -> 2 , 1.6 -> 1
   // -1.6 -> -2 , -2.5 -> -3
   FLOOR(BigDecimal.ROUND_FLOOR),//向负无穷方向舍入。
   // 2.5 -> 3 , 1.6 -> 2
   // -1.6 -> -2 , -2.5 -> -3
   HALF_UP(BigDecimal.ROUND_HALF_UP),//四舍五入，小数部分 >= 0.5 向上，否则向下。
   //......
}
```



#### 等值比较问题

《阿里巴巴 Java 开发手册》中提到：

![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202406202139954.png)

`BigDecimal` 使用 `equals()` 方法进行等值比较出现问题的代码示例：

```java
BigDecimal a = new BigDecimal("1");
BigDecimal b = new BigDecimal("1.0");
System.out.println(a.equals(b));//false
```

这是因为 BigDecimal 的 `equals()` 方法不仅仅会比较值的大小（value）还会比较精度（scale），而 `compareTo()` 方法比较的时候会忽略精度。

1.0 的 scale 是 1，1 的 scale 是 0，因此 `a.equals(b)` 的结果是 false。

![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202406202139936.png)

`compareTo()` 方法可以比较两个 `BigDecimal` 的值：
`a.compareTo(b)` : 返回 -1 表示 `a` 小于 `b`，0 表示 `a` 等于 `b` ， 1 表示 `a` 大于 `b`。

```java
BigDecimal a = new BigDecimal("1");
BigDecimal b = new BigDecimal("1.0");
System.out.println(a.compareTo(b));//0
```



### BigDecimal 存在的性能问题

由于其精确性和灵活性，`BigDecimal` 在某些场景下同样可能会带来性能问题。

 BigDecimal 的性能问题主要源于以下几点：

1. 内存占用：BigDecimal 对象的内存占用较大，尤其是在处理大数字时。每个 BigDecimal 实例都需要维护其精度和标度等信息，这会导致内存开销增加。
2. 不可变性：BigDecimal 是不可变类，每次进行运算或修改值时都会生成一个新的 BigDecimal 实例。这意味着频繁的操作可能会导致大量的对象创建和垃圾回收，对性能造成一定的影响。
3. 运算复杂性：由于 BigDecimal 要求精确计算，它在执行加、减、乘、除等运算时会比较复杂。这些运算需要更多的计算和处理时间，相比原生的基本类型，会带来一定的性能损耗。



性能问题验证：

```java
@Slf4j
public class BigDecimalEfficiency {
 
    //执行次数
    public static int REPEAT_TIMES = 10000000;
 
    // 转BigDecimal 类型计算
    public static double computeByBigDecimal(double a, double b) {
        BigDecimal result = BigDecimal.valueOf(0);
        BigDecimal decimalA = BigDecimal.valueOf(a);
        BigDecimal decimalB = BigDecimal.valueOf(b);
        for (int i = 0; i < REPEAT_TIMES; i++) {
            result = result.add(decimalA.multiply(decimalB));
        }
        return result.doubleValue();
    }
 
    // 转double 类型计算
    public static double computeByDouble(double a, double b) {
        double result = 0;
        for (int i = 0; i < REPEAT_TIMES; i++) {
            result += a * b;
        }
        return result;
    }
 
    public static void main(String[] args) {
        long start1 = System.nanoTime();
        double result1 = computeByBigDecimal(0.120001110034, 11.22);
        long end1 = System.nanoTime();
        long start2 = System.nanoTime();
        double result2 = computeByDouble(0.120001110034, 11.22);
        long end2 = System.nanoTime();
 
        long timeUsed1 = (end1 - start1);
        long timeUsed2 = (end2 - start2);
        log.info("result by BigDecimal:{},time used:{}", result1, timeUsed1);
        log.info("result by Double:{},time used:{}", result2, timeUsed2);
        log.info("timeUsed1/timeUsed2=" + timeUsed1 / timeUsed2);
    }
}
```

运行结果：

![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202408142322787.png)



#### 性能优化策略

`BigDecimal` 性能问题优化策略，可以考虑以下几点优化策略：

1. 避免频繁的对象创建：尽量复用 BigDecimal 对象，而不是每次运算都创建新的实例。可以使用 BigDecimal 的 setScale () 方法设置精度和舍入模式，而不是每次都创建新的对象。
2. 使用原生类型替代：对于一些不需要精确计算的场景，可以使用原生类型（如 int、double、long）来进行运算，以提高性能。只在最后需要精确结果时再转换为 BigDecimal。
3. 使用适当的缓存策略：对于频繁使用的 BigDecimal 对象，可以考虑使用缓存来避免重复创建和销毁。例如，使用对象池或缓存来管理常用的 BigDecimal 对象，以减少对象创建和垃圾回收的开销。
4. 考虑并行计算：对于大规模的计算任务，可以考虑使用并行计算来提高性能。Java 8 提供了 Stream API 和并行流（parallel stream），可以方便地实现并行计算。

需要根据具体的应用场景和需求来权衡精确性和性能，选择合适的处理方式。在对性能要求较高的场景下，可以考虑使用其他更适合的数据类型或算法来替代 BigDecimal。在需要精度计算的情况下，也不能因为 BigDecimal 存在一定的性能问题二选择弃用，顾此失彼。




### BigDecimal 工具类分享

网上有一个使用人数比较多的 `BigDecimal` 工具类，提供了多个静态方法来简化 `BigDecimal` 的操作。源码：

```java
public class BigDecimalUtil {

    /**
     * 默认除法运算精度
     */
    private static final int DEF_DIV_SCALE = 10;

    private BigDecimalUtil() {
    }

    /**
     * 提供精确的加法运算。
     *
     * @param v1 被加数
     * @param v2 加数
     * @return 两个参数的和
     */
    public static double add(double v1, double v2) {
        BigDecimal b1 = BigDecimal.valueOf(v1);
        BigDecimal b2 = BigDecimal.valueOf(v2);
        return b1.add(b2).doubleValue();
    }

    /**
     * 提供精确的减法运算。
     *
     * @param v1 被减数
     * @param v2 减数
     * @return 两个参数的差
     */
    public static double subtract(double v1, double v2) {
        BigDecimal b1 = BigDecimal.valueOf(v1);
        BigDecimal b2 = BigDecimal.valueOf(v2);
        return b1.subtract(b2).doubleValue();
    }

    /**
     * 提供精确的乘法运算。
     *
     * @param v1 被乘数
     * @param v2 乘数
     * @return 两个参数的积
     */
    public static double multiply(double v1, double v2) {
        BigDecimal b1 = BigDecimal.valueOf(v1);
        BigDecimal b2 = BigDecimal.valueOf(v2);
        return b1.multiply(b2).doubleValue();
    }

    /**
     * 提供（相对）精确的除法运算，当发生除不尽的情况时，精确到小数点以后10位，以后的数字四舍五入。
     *
     * @param v1 被除数
     * @param v2 除数
     * @return 两个参数的商
     */
    public static double divide(double v1, double v2) {
        return divide(v1, v2, DEF_DIV_SCALE);
    }

    /**
     * 提供（相对）精确的除法运算。当发生除不尽的情况时，由scale参数指定精度，以后的数字四舍五入。
     *
     * @param v1    被除数
     * @param v2    除数
     * @param scale 表示表示需要精确到小数点以后几位。
     * @return 两个参数的商
     */
    public static double divide(double v1, double v2, int scale) {
        if (scale < 0) {
            throw new IllegalArgumentException(
                    "The scale must be a positive integer or zero");
        }
        BigDecimal b1 = BigDecimal.valueOf(v1);
        BigDecimal b2 = BigDecimal.valueOf(v2);
        return b1.divide(b2, scale, RoundingMode.HALF_EVEN).doubleValue();
    }

    /**
     * 提供精确的小数位四舍五入处理。
     *
     * @param v     需要四舍五入的数字
     * @param scale 小数点后保留几位
     * @return 四舍五入后的结果
     */
    public static double round(double v, int scale) {
        if (scale < 0) {
            throw new IllegalArgumentException(
                    "The scale must be a positive integer or zero");
        }
        BigDecimal b = BigDecimal.valueOf(v);
        BigDecimal one = new BigDecimal("1");
        return b.divide(one, scale, RoundingMode.HALF_UP).doubleValue();
    }

    /**
     * 提供精确的类型转换(Float)
     *
     * @param v 需要被转换的数字
     * @return 返回转换结果
     */
    public static float convertToFloat(double v) {
        BigDecimal b = new BigDecimal(v);
        return b.floatValue();
    }

    /**
     * 提供精确的类型转换(Int)不进行四舍五入
     *
     * @param v 需要被转换的数字
     * @return 返回转换结果
     */
    public static int convertsToInt(double v) {
        BigDecimal b = new BigDecimal(v);
        return b.intValue();
    }

    /**
     * 提供精确的类型转换(Long)
     *
     * @param v 需要被转换的数字
     * @return 返回转换结果
     */
    public static long convertsToLong(double v) {
        BigDecimal b = new BigDecimal(v);
        return b.longValue();
    }

    /**
     * 返回两个数中大的一个值
     *
     * @param v1 需要被对比的第一个数
     * @param v2 需要被对比的第二个数
     * @return 返回两个数中大的一个值
     */
    public static double returnMax(double v1, double v2) {
        BigDecimal b1 = new BigDecimal(v1);
        BigDecimal b2 = new BigDecimal(v2);
        return b1.max(b2).doubleValue();
    }

    /**
     * 返回两个数中小的一个值
     *
     * @param v1 需要被对比的第一个数
     * @param v2 需要被对比的第二个数
     * @return 返回两个数中小的一个值
     */
    public static double returnMin(double v1, double v2) {
        BigDecimal b1 = new BigDecimal(v1);
        BigDecimal b2 = new BigDecimal(v2);
        return b1.min(b2).doubleValue();
    }

    /**
     * 精确对比两个数字
     *
     * @param v1 需要被对比的第一个数
     * @param v2 需要被对比的第二个数
     * @return 如果两个数一样则返回0，如果第一个数比第二个数大则返回1，反之返回-1
     */
    public static int compareTo(double v1, double v2) {
        BigDecimal b1 = BigDecimal.valueOf(v1);
        BigDecimal b2 = BigDecimal.valueOf(v2);
        return b1.compareTo(b2);
    }

}
```

### 小结

浮点数没有办法用二进制精确表示，因此存在精度丢失的风险。不过，Java 提供了 `BigDecimal` 来操作浮点数。`BigDecimal` 的实现利用到了 `BigInteger` （用来操作大整数）, 所不同的是 `BigDecimal` 加入了小数位的概念。



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



### String.Intern ()

调用字符串对象的 intern 方法，会将该字符串对象尝试放入到串池中

- 如果串池中没有该字符串对象，则放入成功

- 如果有该字符串对象，则放入失败

无论放入是否成功，都会返回串池中的字符串对象

注意：此时如果调用 intern 方法成功，堆内存与串池中的字符串对象是同一个对象；如果失败，则不是同一个对象

#### 例 1：

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

#### 例 2：

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



### 自定义一个 String 类

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



## 数组是不是对象

什么是对象？
对象是类的一个实例，有状态和行为  

Java 对象： 

- 软件的对象也有行为和状态 
- 软件对象的状态称之为属性 
- 方法操作对象内部状态的改变，对象的相互调用也是通过方法来完成  

而 java 中的数组具有 java 中其他对象的一些基本特点。比如封装了一些数据，可以访问属性，也可以调用方法。因此，数组是对象



### 证明

可以通过代码验证数组是对象的事实
```java
Class clz = int[].class;
System.out.println(clz.getSuperclass().getName());//java.lang.Object
```

显然，数组继承与 Object，是对象


同理，二维数组也是对象
```java
int[][] arr = new int[2][];
System.out.println(arr.getClass().getSuperclass().getName());//java.lang.Object
```



### 为什么使用 Arrays. Sort 时不能自定义比较器

Arrays.Sort () 默认是升序排序，如果要降序排序，需要自定义比较器
```java
int[] arr = new int[]{1, 2, 3, 4};
Arrays.sort(arr, (a, b) -> Integer.compare(b,a));//报错
```
报错显示：需要的是 int 类型，但提供的是 T 类型的
![](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202408031537390.png)

这是因为 `Arrays.sort` 方法有多个重载版本，其中针对基本类型数组（如 `int[]`）的版本不接受自定义比较器。你尝试传入一个自定义比较器给 `int[]` 数组的 `Arrays.sort` 方法，因此会导致编译错误。

具体来说，`Arrays.sort` 有以下几种主要的重载方法：

1. `Arrays.sort(int[] arr)`：用于排序 `int` 数组，按自然顺序排序，不接受比较器。
2. `Arrays.sort(T[] arr, Comparator<? super T> c)`：用于排序泛型对象数组，按自定义比较器排序。

因此如果试图将一个自定义比较器传入 `int` 数组的 `Arrays.sort` 方法，这是不被允许的，因为基本类型数组的排序方法不接受比较器。



一维数组自定义排序可以用如下方法：

```java
arr = Arrays.stream(arr)
                .boxed()
                .sorted((a,b) -> b-a)
                .mapToInt(Integer::intValue)
                .toArray();
```



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

重写了 hashCode 后，其是对属性值的 hash，p 1 和 p 2 的属性值一致，因此 p 1.HashCode () == p 2.HashCode () 为 true，再进行 equals 方法的判断也为 true，认为是一个对象，因此 set 集合中只有一个对象数据。

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

 

## 关键字

### Final

1. 数据：声明数据为常量，可以是编译时常量，也可以是在运行时被初始化后不能被改变的常量。

   - 对于基本类型，final 使数值不变；

   - 对于引用类型，final 使引用不变，也就不能引用其它对象，但是被引用的对象本身是可以修改的。

2. 方法：声明方法不能被子类重写。
   - Private 方法隐式地被指定为 final，如果在子类中定义的方法和基类中的一个 private 方法签名相同，此时子类的方法不是重写基类方法，而是在子类中定义了一个新的方法。

3. 类：声明类不允许被继承。

### Static

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



###  transient

Java 语言的关键字，变量修饰符，如果用 transient 声明一个实例变量，当对象存储时，它的值不需要维持。

也就是说被 transient 修饰的成员变量，在序列化的时候其值会被忽略，在被反序列化后， transient 变量的值被设为初始值，如 int 型的是 0，对象型的是 null。



## Java 为什么是值传递？

### 形参&实参

方法的定义可能会用到 **参数**（有参的方法），参数在程序语言中分为：

- **实参（实际参数，Arguments）**：用于传递给函数/方法的参数，必须有确定的值。
- **形参（形式参数，Parameters）**：用于定义函数/方法，接收实参，不需要有确定的值。

```java
String hello = "Hello!";
// hello 为实参
SayHello (hello);
// str 为形参
Void sayHello (String str) {
    System.Out.Println (str);
}
```



### 值传递&引用传递

程序设计语言将实参传递给方法（或函数）的方式分为两种：

- **值传递**：方法接收的是实参值的拷贝，会创建副本。
- **引用传递**：方法接收的直接是实参所引用的对象在堆中的地址，不会创建副本，对形参的修改将影响到实参。

很多程序设计语言（比如 C++、 Pascal ) 提供了两种参数传递的方式，不过，在 Java 中只有值传递。



### 为什么 Java 只有值传递？

**为什么说 Java 只有值传递呢？** 通过 3 个例子来给大家证明。

#### 案例 1：传递基本类型参数

代码：

```java
Public static void main (String[] args) {
    Int num 1 = 10;
    Int num 2 = 20;
    Swap (num 1, num 2);
    System.Out.Println ("num 1 = " + num 1);
    System.Out.Println ("num 2 = " + num 2);
}

Public static void swap (int a, int b) {
    Int temp = a;
    A = b;
    B = temp;
    System.Out.Println ("a = " + a);
    System.Out.Println ("b = " + b);
}
```

输出：

```plain
A = 20
B = 10
Num 1 = 10
Num 2 = 20
```

解析：在 `swap ()` 方法中，`a`、`b` 的值进行交换，并不会影响到 `num 1`、`num 2`。因为，`a`、`b` 的值，只是从 `num 1`、`num 2` 的复制过来的。也就是说，a、b 相当于 `num 1`、`num 2` 的副本，副本的内容无论怎么修改，都不会影响到原件本身。

通过上面例子，我们已经知道了一个方法不能修改一个基本数据类型的参数，而对象引用作为参数就不一样，请看案例 2。



#### 案例 2：传递引用类型参数 1

代码：

```java
  Public static void main (String[] args) {
      Int[] arr = { 1, 2, 3, 4, 5 };
      System.Out.Println (arr[0]);
      Change (arr);
      System.Out.Println (arr[0]);
  }

  Public static void change (int[] array) {
      // 将数组的第一个元素变为 0
      Array[0] = 0;
  }
```

输出：

```plain
1
0
```

看了这个案例很多人肯定觉得 Java 对引用类型的参数采用的是引用传递。实际上，并不是的，这里传递的还是值，不过，这个值是实参的地址罢了！

也就是说 `change` 方法的参数拷贝的是 `arr` （实参）的地址，因此，它和 `arr` 指向的是同一个数组对象。这也就说明了为什么方法内部对形参的修改会影响到实参。

为了更强有力地反驳 Java 对引用类型的参数采用的不是引用传递，我们再来看下面这个案例！

#### 案例 3：传递引用类型参数 2

```java
Public class Person {
    Private String name;
   // 省略构造函数、Getter&Setter 方法
}

Public static void main (String[] args) {
    Person xiaoZhang = new Person ("小张");
    Person xiaoLi = new Person ("小李");
    Swap (xiaoZhang, xiaoLi);
    System.Out.Println ("xiaoZhang: " + xiaoZhang.GetName ());
    System.Out.Println ("xiaoLi: " + xiaoLi.GetName ());
}

Public static void swap (Person person 1, Person person 2) {
    Person temp = person 1;
    Person 1 = person 2;
    Person 2 = temp;
    System.Out.Println ("person 1: " + person 1.GetName ());
    System.Out.Println ("person 2: " + person 2.GetName ());
}
```

输出:

```plain
Person 1: 小李
Person 2: 小张
XiaoZhang: 小张
XiaoLi: 小李
```

解析：`swap` 方法的参数 `person 1` 和 `person 2` 只是拷贝的实参 `xiaoZhang` 和 `xiaoLi` 的地址。因此， `person 1` 和 `person 2` 的互换只是拷贝的两个地址的互换罢了，并不会影响到实参 `xiaoZhang` 和 `xiaoLi` 。



### 引用传递是怎么样的？

看到这里，相信你已经知道了 Java 中只有值传递，是没有引用传递的。
但是，引用传递到底长什么样呢？下面以 `C++` 的代码为例，让你看一下引用传递的庐山真面目。

```C++
#include <iostream>

Void incr (int& num)
{
    Std:: cout << "incr before: " << num << "\n";
    Num++;
    Std:: cout << "incr after: " << num << "\n";
}

Int main ()
{
    Int age = 10;
    Std:: cout << "invoke before: " << age << "\n";
    Incr (age);
    Std:: cout << "invoke after: " << age << "\n";
}
```

输出结果：

```plain
Invoke before: 10
Incr before: 10
Incr after: 11
Invoke after: 11
```

分析：可以看到，在 `incr` 函数中对形参的修改，可以影响到实参的值。要注意：这里的 `incr` 形参的数据类型用的是 `int&` 才为引用传递，如果是用 `int` 的话还是值传递哦！



### 为什么 Java 不引入引用传递呢？

引用传递看似很好，能在方法内就直接把实参的值修改了，但是，为什么 Java 不引入引用传递呢？

**注意：以下为个人观点看法，并非来自于 Java 官方：**

1. 出于安全考虑，方法内部对值进行的操作，对于调用者都是未知的（把方法定义为接口，调用方不关心具体实现）。你也想象一下，如果拿着银行卡去取钱，取的是 100，扣的是 200，是不是很可怕。
2. Java 之父 James Gosling 在设计之初就看到了 C、C++ 的许多弊端，所以才想着去设计一门新的语言 Java。在他设计 Java 的时候就遵循了简单易用的原则，摒弃了许多开发者一不留意就会造成问题的“特性”，语言本身的东西少了，开发者要学习的东西也少了。

### 小结

Java 中将实参传递给方法（或函数）的方式是 **值传递**：

- 如果参数是基本类型的话，很简单，传递的就是基本类型的字面量值的拷贝，会创建副本。
- 如果参数是引用类型，传递的就是实参所引用的对象在堆中地址值的拷贝，同样也会创建副本。



##  序列化和反序列化

- 序列化：把对象转换为字节序列的过程称为对象的序列化.
- 反序列化：把字节序列恢复为对象的过程称为对象的反序列化.



### 什么时候会用到

当只在本地 JVM 里运行下 Java 实例，这个时候是不需要什么序列化和反序列化的，但当出现以下场景时，就需要序列化和反序列化了：

- 当需要将内存中的对象持久化到磁盘，数据库中时
- 当需要与浏览器进行交互时
- 当需要实现 RPC 时

但是当我们在与浏览器交互时，还有将内存中的对象持久化到数据库中时，好像都没有去进行序列化和反序列化，因为我们都没有实现 Serializable 接口，但一直正常运行？

先给出结论：**只要我们对内存中的对象进行持久化或网络传输，这个时候都需要序列化和反序列化.**

理由：服务器与浏览器交互时真的没有用到 Serializable 接口吗? **JSON 格式实际上就是将一个对象转化为字符串**，所以服务器与浏览器交互时的数据格式其实是字符串，我们来看来 String 类型的源码:

```java
public final class String implements java. Io. Serializable，Comparable<String>，CharSequence {
    /\*\* The value is used for character storage. \*/
    Private final char value\[\];

    /\*\* Cache the hash code for the string \*/
    Private int hash; // Default to 0

    /\*\* use serialVersionUID from JDK 1.0.2 for interoperability \*/
    Private static final long serialVersionUID = -6849794470754667710 L;

    ......
}
```

String 类型实现了 Serializable 接口，并显示指定 serialVersionUID 的值.

然后再来看对象持久化到数据库中时的情况，Mybatis 数据库映射文件里的 insert 代码:

```text
<insert id="insertUser" parameterType="org.tyshawn.bean.User">
    INSERT INTO t\_user (name，age) VALUES (#{name}，#{age})
</insert>
```

实际上并不是将整个对象持久化到数据库中，而是将对象中的属性持久化到数据库中，而这些属性（如 Date/String）都实现了 Serializable 接口。



### 为什么要实现 Serializable 接口?

在 Java 中实现了 Serializable 接口后， JVM 在类加载的时候就会发现我们实现了这个接口，然后在初始化实例对象的时候就会在底层实现序列化和反序列化。如果被写对象类型不是 String、数组、Enum，并且没有实现 Serializable 接口，那么在进行序列化的时候，将抛出 NotSerializableException。源码如下：

```java
// remaining cases
If (obj instanceof String) {
    WriteString ((String) obj, unshared);
} else if (cl.IsArray ()) {
    WriteArray (obj, desc, unshared);
} else if (obj instanceof Enum) {
    writeEnum ((Enum<?>) obj, desc, unshared);
} else if (obj instanceof Serializable) {
    WriteOrdinaryObject (obj, desc, unshared);
} else {
    If (extendedDebugInfo) {
        Throw new NotSerializableException (
            Cl.GetName () + "\n" + debugInfoStack.ToString ());
    } else {
        Throw new NotSerializableException (cl.GetName ());
    }
}
```



### 为什么要显示指定 serialVersionUID 的值?

如果不显示指定 serialVersionUID，JVM 在序列化时会根据属性自动生成一个 serialVersionUID，然后与属性一起序列化，再进行持久化或网络传输. 在反序列化时，JVM 会再根据属性自动生成一个新版 serialVersionUID，然后将这个新版 serialVersionUID 与序列化时生成的旧版 serialVersionUID 进行比较，如果相同则反序列化成功，否则报错.

如果显示指定了 serialVersionUID，JVM 在序列化和反序列化时仍然都会生成一个 serialVersionUID，但值为显示指定的值，这样在反序列化时新旧版本的 serialVersionUID 就一致了.

当然了，如果类写完后不再修改，那么不指定 serialVersionUID，不会有问题，但这在实际开发中是不可能的，类会不断迭代，一旦类被修改了，那旧对象反序列化就会报错。所以在实际开发中，都会显示指定一个 serialVersionUID。



### Static 属性为什么不会被序列化?

因为序列化是针对对象而言的，而 static 属性优先于对象存在，随着类的加载而加载，所以不会被序列化.

看到这个结论，是不是有人会问，serialVersionUID 也被 static 修饰，为什么 serialVersionUID 会被序列化? 其实 serialVersionUID 属性并没有被序列化，JVM 在序列化对象时会自动生成一个 serialVersionUID，然后将显示指定的 serialVersionUID 属性值赋给自动生成的 serialVersionUID。



### 常见序列化的方式

序列化只是定义了拆解对象的具体规则，那这种规则肯定也是多种多样的，比如现在常见的序列化方式有：JDK 原生、JSON、ProtoBuf、Hessian、Kryo 等。

- JDK 原生

作为一个成熟的编程语言，JDK 自带了序列化方法。只需要类实现了`Serializable`接口，就可以通过`ObjectOutputStream`类将对象变成 byte[]字节数组。

JDK 序列化会把对象类的描述信息和所有的属性以及继承的元数据都序列化为字节流，所以会导致生成的字节流相对比较大。

另外，这种序列化方式是 JDK 自带的，因此不支持跨语言。

简单总结一下：JDK 原生的序列化方式生成的字节流比较大，也不支持跨语言，因此在实际项目和框架中用的都比较少。



- ProtoBuf

谷歌推出的，是一种语言无关、平台无关、可扩展的序列化结构数据的方法，它可用于通信协议、数据存储等。序列化后体积小，一般用于对传输性能有较高要求的系统。



- Hessian

Hessian 是一个轻量级的二进制 web service 协议，主要用于传输二进制数据。

在传输数据前 Hessian 支持将对象序列化成二进制流，相对于 JDK 原生序列化，Hessian 序列化之后体积更小，性能更优。



- Kryo

Kryo 是一个 Java 序列化框架，号称 Java 最快的序列化框架。Kryo 在序列化速度上很有优势，底层依赖于字节码生成机制。

由于只能限定在 JVM 语言上，所以 Kryo 不支持跨语言使用。



- JSON

上面讲的几种序列化方式都是直接将对象变成二进制，也就是 byte[]字节数组，这些方式都可以叫二进制方式。

JSON 序列化方式生成的是一串有规则的字符串，在可读性上要优于上面几种方式，但是在体积上就没什么优势了。

另外 JSON 是有规则的字符串，不跟任何编程语言绑定，天然上就具备了跨平台。

**总结一下：JSON 可读性强，支持跨平台，体积稍微逊色。**

JSON 序列化常见的框架有：`fastJSON`、`Jackson`、`Gson` 等。



### 序列化技术的选型

上面列举的这些序列化技术各有优缺点，不能简单地说哪一种就是最好的，不然也不会有这么多序列化技术共存了。

既然有这么多序列化技术可供选择，那在实际项目中如何选型呢？

我认为需要结合具体的项目来看，比较技术是服务于业务的。你可以从下面这几个因素来考虑：

- 协议是否支持跨平台：如果一个大的系统有好多种语言进行混合开发，那么就肯定不适合用有语言局限性的序列化协议，比如 JDK 原生、Kryo 这些只能用在 Java 语言范围下，你用 JDK 原生方式进行序列化，用其他语言是无法反序列化的。

- 序列化的速度：如果序列化的频率非常高，那么选择序列化速度快的协议会为你的系统性能提升不少。

- 序列化生成的体积：如果频繁的在网络中传输的数据那就需要数据越小越好，小的数据传输快，也不占带宽，也能整体提升系统的性能，因此序列化生成的体积就很关键了。





## 时间类库相关

在 Java 中，处理日期和时间的方式经历了演变。在 Java 8 之前，主要使用 `java. Util. Date` 类来表示日期和时间，但它存在一些问题，如不可变性、线程安全性等。Java 8 引入了新的日期时间 API，位于 `java. Time` 包中，提供了更加强大、易用和安全的日期时间处理方式。



### LocalDate、LocalTime、LocalDateTime

![|662x438](https://seven97-blog.oss-cn-hangzhou.aliyuncs.com/imgs/202407121801562.gif)



```java
LocalDate now = LocalDate.Now ();
System.Out.Println (now.GetYear ());//2024
System.Out.Println (now.GetMonthValue ());//7
System.Out.Println (now.GetDayOfMonth ());//12
System.Out.Println ("------------");

LocalTime nowTime = LocalTime.Now ();
System.Out.Println (nowTime.GetHour ());//18
System.Out.Println (nowTime.GetMinute ());//0
System.Out.Println (nowTime.GetSecond ());//48
System.Out.Println ("------------");
        
LocalDateTime localDateTime = LocalDateTime.Now ();
System.Out.Println (localDateTime.GetYear ());//2024
System.Out.Println (localDateTime.GetMonthValue ());//7
System.Out.Println (localDateTime.GetDayOfMonth ());//12
System.Out.Println (localDateTime.GetHour ());//18
System.Out.Println (localDateTime.GetMinute ());//0
System.Out.Println (localDateTime.GetSecond ());//48
```



> 说个题外话：如果在国际化应用中使用 `LocalDate` 时，需要明确理解其不包含时区信息的特点。这意味着，如果直接使用 `LocalDate.Now ()` 来获取“当前日期”，实际上会使用系统默认的时区来确定当前的日期。这对于那些严格依赖于用户所在地的具体日期的应用来说，可能会引入一些问题。
>
> 例如，假设服务器位于美国东部时间区（EST），而用户位于新西兰（NZST）。当美国东部时间是 4 月 1 日的晚上 11 点时，在新西兰已经是 4 月 2 日的下午 3 点。使用 `LocalDate.Now ()` 得到的日期将基于服务器的时区，而不是用户的时区，这在某些情况下可能不是期望的行为。



### Instant 时间戳

Instant 类是为了方便计算机理解的而设计的，它表示一个持续时间段上某个点的单一大整型数，实际上它是以 Unix 元年时间（传统的设定为 UTC 时区 1970 年 1 月 1 日午夜时分）开始所经历的秒数进行计算（最小计算单位为纳秒）。



### Duration 与 Period

 Duration 是用于比较两个 LocalTime 对象或者两个 Instant 之间的时间差值。
