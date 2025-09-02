---
Title: Java 基础核心要点-Bigdecimal相关
Category: Java
tags:
  - java
  - 数据类型
---
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

