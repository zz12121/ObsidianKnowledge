---
Title: Java 集合框架-ArrayList
Category: Java
tags:
  - java
  - 集合框架
---
从名字就可以看得出来，ArrayList 实现了 List 接口，并且是基于数组实现的。  
  
数组的大小是固定的，一旦创建的时候指定了大小，就不能再调整了。也就是说，如果数组满了，就不能再添加任何元素了。ArrayList 在数组的基础上实现了自动扩容，并且提供了比数组更丰富的预定义方法（各种增删改查），非常灵活。   

ArrayList 在JDK1.8 前后的实现区别：

- JDK1.7：像饿汉式，直接创建一个初始容量为10的数组
- JDK1.8：像懒汉式，一开始创建一个长度为0的数组，当添加add第一个元素时再创建一个初始容量为10的数组

size(), isEmpty(), get(), set()方法均能在常数时间内完成，add()方法的时间开销跟插入位置有关，addAll()方法的时间开销跟添加元素的个数成正比。其余方法大都是线性时间。

为追求效率，ArrayList没有实现同步(synchronized)，如果需要多个线程并发访问，用户可以手动同步，也可使用Vector替代
  
### 01、创建 ArrayList  
  
```java  
ArrayList<String> alist = new ArrayList<String>();  
```  
  
可以通过上面的语句来创建一个字符串类型的 ArrayList（通过尖括号来限定 ArrayList 中元素的类型，如果尝试添加其他类型的元素，将会产生编译错误），更简化的写法如下：  
  
```java  
List<String> alist = new ArrayList<>();  
```  
  
由于 ArrayList 实现了 List 接口，所以 alist 变量的类型可以是 List 类型；new 关键字声明后的尖括号中可以不再指定元素的类型，因为编译器可以通过前面尖括号中的类型进行智能推断。  
  
此时会调用无参构造方法（见下面的代码）创建一个空的数组，常量DEFAULTCAPACITY_EMPTY_ELEMENTDATA的值为 `{}`。  
  
```java  
public ArrayList() {  
    this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;}  
```  
  
如果非常确定 ArrayList 中元素的个数，在创建的时候还可以指定初始大小。  
  
```java  
List<String> alist = new ArrayList<>(20);  
```  
  
这样做的好处是，可以有效地避免在添加新的元素时进行不必要的扩容。  
  
### 02、向 ArrayList 中添加元素  

可以通过 `add()` 方法向 ArrayList 中添加一个元素。  
  
```java  
alist.add("呵呵");  
```  
  
我们来跟一下源码，看看 add 方法到底执行了哪些操作。跟的过程中，我们也可以偷师到 Java 源码的作者（大师级程序员）是如何优雅地写代码的。  
  
我先给个结论，全当抛砖引玉。  
  
```
堆栈过程图示：  
add(element)  
└── if (size == elementData.length) // 判断是否需要扩容  
    ├── grow(minCapacity) // 扩容  
    │   └── newCapacity = oldCapacity + (oldCapacity >> 1) // 计算新的数组容量  
    │   └── Arrays.copyOf(elementData, newCapacity) // 创建新的数组  
    ├── elementData[size++] = element; // 添加新元素  
    └── return true; // 添加成功  
```  
  
来具体看一下，先是 `add()` 方法的源码（已添加好详细地注释）  
  
```java  
/**  
 * 将指定元素添加到 ArrayList 的末尾  
 * @param e 要添加的元素  
 * @return 添加成功返回 true */
 public boolean add(E e) {  
    ensureCapacityInternal(size + 1);  // 确保 ArrayList 能够容纳新的元素  
    elementData[size++] = e; // 在 ArrayList 的末尾添加指定元素  
    return true;}  
```  
  
参数 e 为要添加的元素，此时的值为“呵呵”，size 为 ArrayList 的长度，此时为 0。  
  
继续跟下去，来看看 `ensureCapacityInternal()`方法：  
  
```java  
/**
 * 确保 ArrayList 能够容纳指定容量的元素
 * @param minCapacity 指定容量的最小值
 */
private void ensureCapacityInternal(int minCapacity) {
    if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) { // 如果 elementData 还是默认的空数组
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity); // 使用 DEFAULT_CAPACITY 和指定容量的最小值中的较大值
    }

    ensureExplicitCapacity(minCapacity); // 确保容量能够容纳指定容量的元素
}
```  
  
此时：  
  
- 参数 minCapacity 为 1（size+1 传过来的）  
- elementData 为存放 ArrayList 元素的底层数组，前面声明 ArrayList 的时候讲过了，此时为空 `{}`  
- DEFAULTCAPACITY_EMPTY_ELEMENTDATA 前面也讲过了，为 `{}`  
  
所以，if 条件此时为 true，if 语句`minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity)`要执行。  
  
DEFAULT_CAPACITY 为 10（见下面的代码），所以执行完这行代码后，minCapacity 为 10，`Math.max()` 方法的作用是取两个当中最大的那个。  
  
```java  
private static final int DEFAULT_CAPACITY = 10;  
```  
  
接下来执行 `ensureExplicitCapacity()` 方法，来看一下源码：  
  
```java  
/**
 * 检查并确保集合容量足够，如果需要则增加集合容量。
 *
 * @param minCapacity 所需最小容量
 */
private void ensureExplicitCapacity(int minCapacity) {
    // 检查是否超出了数组范围，确保不会溢出
    if (minCapacity - elementData.length > 0)
        // 如果需要增加容量，则调用 grow 方法
        grow(minCapacity);
}
```  
  
此时：  
  
- 参数 minCapacity 为 10  
- elementData.length 为 0（数组为空）  
  
所以 10-0>0，if 条件为 true，进入 if 语句执行 `grow()` 方法，来看源码：  
  
```java  
/**
 * 扩容 ArrayList 的方法，确保能够容纳指定容量的元素
 * @param minCapacity 指定容量的最小值
 */
private void grow(int minCapacity) {
    // 检查是否会导致溢出，oldCapacity 为当前数组长度
    int oldCapacity = elementData.length;
    int newCapacity = oldCapacity + (oldCapacity >> 1); // 扩容至原来的1.5倍
    if (newCapacity - minCapacity < 0) // 如果还是小于指定容量的最小值
        newCapacity = minCapacity; // 直接扩容至指定容量的最小值
    if (newCapacity - MAX_ARRAY_SIZE > 0) // 如果超出了数组的最大长度
        newCapacity = hugeCapacity(minCapacity); // 扩容至数组的最大长度
    // 将当前数组复制到一个新数组中，长度为 newCapacity
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```  
  
此时：  
  
- 参数 minCapacity 为 10  
- 变量 oldCapacity 为 0  
  
所以 newCapacity 也为 0，于是 `newCapacity - minCapacity` 等于 -10 小于 0，于是第一个 if 条件为 true，执行第一个 if 语句 `newCapacity = minCapacity`，然后 newCapacity 为 10。  
  
紧接着执行 `elementData = Arrays.copyOf(elementData, newCapacity);`，也就是进行数组的第一次扩容，长度为 10。  
  
回到 `add()` 方法：  
  
```java  
public boolean add(E e) {  
    ensureCapacityInternal(size + 1);    
    elementData[size++] = e;    
    return true;
}  
```  
  
执行 `elementData[size++] = e`。  
  
此时：  
  
- size 为 0  
- e 为 “呵呵”  
  
所以数组的第一个元素（下标为 0） 被赋值为“沉默王二”，接着返回 true，第一次 add 方法执行完毕。  
  
PS：add 过程中会遇到一个令新手感到困惑的右移操作符 `>>`，借这个机会来解释一下。  
  
ArrayList 在第一次执行 add 后会扩容为 10，那 ArrayList 第二次扩容发生在什么时候呢？  
  
答案是添加第 11 个元素时，大家可以尝试分析一下这个过程。  
  
### 03、右移操作符  
  
“oldCapacity 等于 10，`oldCapacity >> 1` 这个表达式等于多少呢？

`>>` 是右移运算符，`oldCapacity >> 1` 相当于 oldCapacity 除以 2。在计算机内部，都是按照二进制存储的，10 的二进制就是 1010，也就是 `0*2^0 + 1*2^1 + 0*2^2 + 1*2^3`=0+2+0+8=10 。。。。。。”  
  
先从位权的含义说起，平常我们使用的是十进制数，比如说 39，并不是简单的 3 和 9，3 表示的是 `3*10 = 30`，9 表示的是 `9*1 = 9`，和 3 相乘的 10，和 9 相乘的 1，就是**位权**。位数不同，位权就不同，第 1 位是 10 的 0 次方（也就是 `10^0=1`），第 2 位是 10 的 1 次方（`10^1=10`），第 3 位是 10 的 2 次方（`10^2=100`），最右边的是第一位，依次类推。  
  
位权这个概念同样适用于二进制，第 1 位是 2 的 0 次方（也就是 `2^0=1`），第 2 位是 2 的 1 次方（`2^1=2`），第 3 位是 2 的 2 次方（`2^2=4`），第 4 位是 2 的 3 次方（`2^3=8`）。  
  
十进制的情况下，10 是基数，二进制的情况下，2 是基数。  
  
10 在十进制的表示法是 `0*10^0+1*10^1`=0+10=10。  
  
10 的二进制数是 1010，也就是 `0*2^0 + 1*2^1 + 0*2^2 + 1*2^3`=0+2+0+8=10。  
  
然后是**移位运算**，移位分为左移和右移，在 Java 中，左移的运算符是 `<<`，右移的运算符 `>>`。  
  
拿 `oldCapacity >> 1` 来说吧，`>>` 左边的是被移位的值，此时是 10，也就是二进制 `1010`；`>>` 右边的是要移位的位数，此时是 1。  
  
1010 向右移一位就是 101，空出来的最高位此时要补 0，也就是 0101。  
  
那为什么不补 1 呢？
  
因为是算术右移，并且是正数，所以最高位补 0；如果表示的是负数，就需要补 1。0101 的十进制就刚好是 `1*2^0 + 0*2^1 + 1*2^2 + 0*2^3`=1+0+4+0=5，如果多移几个数来找规律的话，就会发现，右移 1 位是原来的 1/2，右移 2 位是原来的 1/4，诸如此类。”
  
也就是说，ArrayList 的大小会扩容为原来的大小+原来大小/2，也就是 1.5 倍。  
  
可以通过在 ArrayList 中添加第 11 个元素来 debug 验证一下。  
  
![[09_Attachments/Java/集合框架/Java_ArrayList_001.png]]  
  
### 04、向 ArrayList 的指定位置添加元素  
  
除了 `add(E e)` 方法，还可以通过 `add(int index, E element)` 方法把元素添加到 ArrayList 的指定位置：  
  
```java  
alist.add(0, "呵呵");  
```  
  
`add(int index, E element)` 方法的源码如下：  
  
```java  
/**
 * 在指定位置插入一个元素。
 *
 * @param index   要插入元素的位置
 * @param element 要插入的元素
 * @throws IndexOutOfBoundsException 如果索引超出范围，则抛出此异常
 */
public void add(int index, E element) {
    rangeCheckForAdd(index); // 检查索引是否越界

    ensureCapacityInternal(size + 1);  // 确保容量足够，如果需要扩容就扩容
    System.arraycopy(elementData, index, elementData, index + 1,
            size - index); // 将 index 及其后面的元素向后移动一位
    elementData[index] = element; // 将元素插入到指定位置
    size++; // 元素个数加一
}
```  
  
`add(int index, E element)`方法会调用到一个非常重要的本地方法 `System.arraycopy()`，它会对数组进行复制（要插入位置上的元素往后复制）。  
  
来细品一下。  
  
这是 arraycopy() 的语法：  
  
```java  
System.arraycopy(Object src, int srcPos, Object dest, int destPos, int length);  
```  
  
在 `ArrayList.add(int index, E element)` 方法中，具体用法如下：  
  
```java  
System.arraycopy(elementData, index, elementData, index + 1, size - index);  
```  
  
- elementData：表示要复制的源数组，即 ArrayList 中的元素数组。  
- index：表示源数组中要复制的起始位置，即需要将 index 及其后面的元素向后移动一位。  
- elementData：表示要复制到的目标数组，即 ArrayList 中的元素数组。  
- index + 1：表示目标数组中复制的起始位置，即将 index 及其后面的元素向后移动一位后，应该插入到的位置。  
- size - index：表示要复制的元素个数，即需要将 index 及其后面的元素向后移动一位，需要移动的元素个数为 size - index。  
  
![[09_Attachments/Java/集合框架/Java_ArrayList_002.png]]

addAll()方法能够一次添加多个元素，根据位置不同也有两个版本：

- 在末尾添加的addAll(Collection< ? extends E> c) 方法，
- 从指定位置开始插入的addAll(int index, Collection< ? extends E> c)方法

跟add()方法类似，在插入之前也需要进行空间检查，如果需要则自动扩容；如果从指定位置插入，也会存在移动元素的情况。 addAll()的时间复杂度不仅跟插入元素的多少有关，也跟插入的位置相关。

### 05、更新 ArrayList 中的元素  
  
可以使用 `set()` 方法来更改 ArrayList 中的元素，需要提供下标和新元素。  
  
```java  
alist.set(0, "呵呵2");  
```  
  
假设原来 0 位置上的元素为“呵呵”，现在可以将其更新为“呵呵2”。  
  
来看一下 `set()` 方法的源码：  
  
```java  
/**
 * 用指定元素替换指定位置的元素。
 *
 * @param index   要替换的元素的索引
 * @param element 要存储在指定位置的元素
 * @return 先前在指定位置的元素
 * @throws IndexOutOfBoundsException 如果索引超出范围，则抛出此异常
 */
public E set(int index, E element) {
    rangeCheck(index); // 检查索引是否越界

    E oldValue = elementData(index); // 获取原来在指定位置上的元素
    elementData[index] = element; // 将新元素替换到指定位置上
    return oldValue; // 返回原来在指定位置上的元素
} 
```  
  
该方法会先对指定的下标进行检查，看是否越界，然后替换新值并返回旧值。  
  
### 06、删除 ArrayList 中的元素  
  
`remove(int index)` 方法用于删除指定下标位置上的元素，`remove(Object o)` 方法用于删除指定值的元素。  
  
```java  
alist.remove(1);  
alist.remove("呵呵2");  
```  
  
先来看 `remove(int index)` 方法的源码：  
  
```java  
/**
 * 删除指定位置的元素。
 *
 * @param index 要删除的元素的索引
 * @return 先前在指定位置的元素
 * @throws IndexOutOfBoundsException 如果索引超出范围，则抛出此异常
 */
public E remove(int index) {
    rangeCheck(index); // 检查索引是否越界

    E oldValue = elementData(index); // 获取要删除的元素

    int numMoved = size - index - 1; // 计算需要移动的元素个数
    if (numMoved > 0) // 如果需要移动元素，就用 System.arraycopy 方法实现
        System.arraycopy(elementData, index+1, elementData, index,
                numMoved);
    elementData[--size] = null; // 将数组末尾的元素置为 null，让 GC 回收该元素占用的空间

    return oldValue; // 返回被删除的元素
} 
```  
  
需要注意的是，在 ArrayList 中，删除元素时，需要将删除位置后面的元素向前移动一位，以填补删除位置留下的空缺。如果需要移动元素，则需要使用 System.arraycopy 方法将删除位置后面的元素向前移动一位。最后，将数组末尾的元素置为 null，以便让垃圾回收机制回收该元素占用的空间。  
  
再来看 `remove(Object o)` 方法的源码：  
  
```java  
/**
 * 删除列表中第一次出现的指定元素（如果存在）。
 *
 * @param o 要删除的元素
 * @return 如果列表包含指定元素，则返回 true；否则返回 false
 */
public boolean remove(Object o) {
    if (o == null) { // 如果要删除的元素是 null
        for (int index = 0; index < size; index++) // 遍历列表
            if (elementData[index] == null) { // 如果找到了 null 元素
                fastRemove(index); // 调用 fastRemove 方法快速删除元素
                return true; // 返回 true，表示成功删除元素
            }
    } else { // 如果要删除的元素不是 null
        for (int index = 0; index < size; index++) // 遍历列表
            if (o.equals(elementData[index])) { // 如果找到了要删除的元素
                fastRemove(index); // 调用 fastRemove 方法快速删除元素
                return true; // 返回 true，表示成功删除元素
            }
    }
    return false; // 如果找不到要删除的元素，则返回 false
}
```  
  
该方法通过遍历的方式找到要删除的元素，null 的时候使用 == 操作符判断，非 null 的时候使用 `equals()` 方法，然后调用 `fastRemove()` 方法。  
  
注意：  
  
- 有相同元素时，只会删除第一个。  
- 判断两个元素是否相等，可以参考Java如何判断两个字符串是否相等  
  
继续往后面跟，来看一下 `fastRemove()` 方法：  
  
```java  
/**
 * 快速删除指定位置的元素。
 *
 * @param index 要删除的元素的索引
 */
private void fastRemove(int index) {
    int numMoved = size - index - 1; // 计算需要移动的元素个数
    if (numMoved > 0) // 如果需要移动元素，就用 System.arraycopy 方法实现
        System.arraycopy(elementData, index+1, elementData, index,
                numMoved);
    elementData[--size] = null; // 将数组末尾的元素置为 null，让 GC 回收该元素占用的空间
}
```  
  
同样是调用 `System.arraycopy()` 方法对数组进行复制和移动。  
  
![[09_Attachments/Java/集合框架/Java_ArrayList_003.png]]  
  
### 07、查找 ArrayList 中的元素  
  
如果要正序查找一个元素，可以使用 `indexOf()` 方法；如果要倒序查找一个元素，可以使用 `lastIndexOf()` 方法。  
  
```java  
alist.indexOf("呵呵");  
alist.lastIndexOf("呵呵");  
```  
  
来看一下 `indexOf()` 方法的源码：  
  
```java  
/**
 * 返回指定元素在列表中第一次出现的位置。
 * 如果列表不包含该元素，则返回 -1。
 *
 * @param o 要查找的元素
 * @return 指定元素在列表中第一次出现的位置；如果列表不包含该元素，则返回 -1
 */
public int indexOf(Object o) {
    if (o == null) { // 如果要查找的元素是 null
        for (int i = 0; i < size; i++) // 遍历列表
            if (elementData[i]==null) // 如果找到了 null 元素
                return i; // 返回元素的索引
    } else { // 如果要查找的元素不是 null
        for (int i = 0; i < size; i++) // 遍历列表
            if (o.equals(elementData[i])) // 如果找到了要查找的元素
                return i; // 返回元素的索引
    }
    return -1; // 如果找不到要查找的元素，则返回 -1
}
```  
  
如果元素为 null 的时候使用“==”操作符，否则使用 `equals()` 方法。  
  
`lastIndexOf()` 方法和 `indexOf()` 方法类似，不过遍历的时候从最后开始。  
  
```java  
/**
 * 返回指定元素在列表中最后一次出现的位置。
 * 如果列表不包含该元素，则返回 -1。
 *
 * @param o 要查找的元素
 * @return 指定元素在列表中最后一次出现的位置；如果列表不包含该元素，则返回 -1
 */
public int lastIndexOf(Object o) {
    if (o == null) { // 如果要查找的元素是 null
        for (int i = size-1; i >= 0; i--) // 从后往前遍历列表
            if (elementData[i]==null) // 如果找到了 null 元素
                return i; // 返回元素的索引
    } else { // 如果要查找的元素不是 null
        for (int i = size-1; i >= 0; i--) // 从后往前遍历列表
            if (o.equals(elementData[i])) // 如果找到了要查找的元素
                return i; // 返回元素的索引
    }
    return -1; // 如果找不到要查找的元素，则返回 -1
}
```  
  
`contains()` 方法可以判断 ArrayList 中是否包含某个元素，其内部就是通过 `indexOf()` 方法实现的：  
  
```java  
public boolean contains(Object o) {
    return indexOf(o) >= 0;
} 
```  
  
### 08、二分查找法  
  
如果 ArrayList 中的元素是经过排序的，就可以使用二分查找法，效率更快。  
  
`Collections` 类的 `sort()` 方法可以对 ArrayList 进行排序，该方法会按照字母顺序对 String 类型的列表进行排序。如果是自定义类型的列表，还可以指定 Comparator 进行排序。  
  
这里先简单地了解一下，后面会详细地讲。  
  
```java  
List<String> copy = new ArrayList<>(alist);  
copy.add("a");  
copy.add("c");  
copy.add("b");  
copy.add("d");  
  
Collections.sort(copy);  
System.out.println(copy);  
```  
  
输出结果如下所示：  
  
```  
[a, b, c, d]  
```  
  
排序后就可以使用二分查找法了：  
  
```java  
int index = Collections.binarySearch(copy, "b");  
```  
  
### 09、ArrayList增删改查时的时间复杂度  
  
#### 1）查询  
  
时间复杂度为 O(1)，因为 ArrayList 内部使用数组来存储元素，所以可以直接根据索引来访问元素。  
  
```java  
/**
 * 返回列表中指定位置的元素。
 *
 * @param index 要返回的元素的索引
 * @return 列表中指定位置的元素
 * @throws IndexOutOfBoundsException 如果索引超出范围（index < 0 || index >= size()）
 */
public E get(int index) {
    rangeCheck(index); // 检查索引是否合法
    return elementData(index); // 调用 elementData 方法获取元素
}

/**
 * 返回列表中指定位置的元素。
 * 此方法不进行边界检查，因此只应由内部方法和迭代器调用。
 *
 * @param index 要返回的元素的索引
 * @return 列表中指定位置的元素
 */
E elementData(int index) {
    return (E) elementData[index]; // 返回指定索引位置上的元素
}
```  
  
#### 2）插入  
  
添加一个元素（调用 `add()` 方法时）的时间复杂度最好情况为 O(1)，最坏情况为 O(n)。  
  
- 如果在列表末尾添加元素，时间复杂度为 O(1)。  
- 如果要在列表的中间或开头插入元素，则需要将插入位置之后的元素全部向后移动一位，时间复杂度为 O(n)。  
  
#### 3）删除  
  
删除一个元素（调用 `remove(Object)` 方法时）的时间复杂度最好情况 O(1)，最坏情况 O(n)。  
  
- 如果要删除列表末尾的元素，时间复杂度为 O(1)。  
- 如果要删除列表中间或开头的元素，则需要将删除位置之后的元素全部向前移动一位，时间复杂度为 O(n)。  
  
  
#### 4）修改  
  
修改一个元素（调用 `set()`方法时）与查询操作类似，可以直接根据索引来访问元素，时间复杂度为 O(1)。  
  
```java  
/**
 * 用指定元素替换列表中指定位置的元素。
 *
 * @param index 要替换元素的索引
 * @param element 要放入列表中的元素
 * @return 原来在指定位置上的元素
 * @throws IndexOutOfBoundsException 如果索引超出范围（index < 0 || index >= size()）
 */
public E set(int index, E element) {
    rangeCheck(index); // 检查索引是否合法

    E oldValue = elementData(index); // 获取原来在指定位置上的元素
    elementData[index] = element; // 将指定位置上的元素替换为新元素
    return oldValue; // 返回原来在指定位置上的元素
}
```  
  
### 10、trimToSize()

ArrayList还给我们提供了将底层数组的容量调整为当前列表保存的实际元素的大小的功能。它可以通过trimToSize方法来实现。代码如下:

```java
    /**
     * Trims the capacity of this <tt>ArrayList</tt> instance to be the
     * list's current size.  An application can use this operation to minimize
     * the storage of an <tt>ArrayList</tt> instance.
     */
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```

### 11、indexOf(), lastIndexOf()

获取元素的第一次出现的index:

```java
/**
     * Returns the index of the first occurrence of the specified element
     * in this list, or -1 if this list does not contain the element.
     * More formally, returns the lowest index <tt>i</tt> such that
     * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>,
     * or -1 if there is no such index.
     */
    public int indexOf(Object o) {
        if (o == null) {
            for (int i = 0; i < size; i++)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = 0; i < size; i++)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

获取元素的最后一次出现的index:

```java
    /**
     * Returns the index of the last occurrence of the specified element
     * in this list, or -1 if this list does not contain the element.
     * More formally, returns the highest index <tt>i</tt> such that
     * <tt>(o==null&nbsp;?&nbsp;get(i)==null&nbsp;:&nbsp;o.equals(get(i)))</tt>,
     * or -1 if there is no such index.
     */
    public int lastIndexOf(Object o) {
        if (o == null) {
            for (int i = size-1; i >= 0; i--)
                if (elementData[i]==null)
                    return i;
        } else {
            for (int i = size-1; i >= 0; i--)
                if (o.equals(elementData[i]))
                    return i;
        }
        return -1;
    }
```

### 12、遍历时删除（添加）常见陷阱
#### for循环遍历list
删除某个元素后，list的大小发生了变化，而索引也在变化，所以会导致遍历的时候漏掉某些元素。比如当删除第1个元素后，继续根据索引访问第2个元素时，因为删除的关系后面的元素都往前移动了一位，所以实际访问的是第3个元素。因此，这种方式可以用在删除特定的一个元素时使用，但不适合循环删除多个元素时使用。

```java
for(int i=0;i<list.size();i++){
    if(list.get(i).equals("del"))
        list.remove(i);
}
```

解决办法：

//从list最后一个元素开始遍历

```java
//从list最后一个元素开始遍历
for(int i=list.size()-1;i>+0;i--){
    if(list.get(i).equals("del"))
        list.remove(i);
}
```

#### 增强for循环
删除元素后继续循环会抛异常java.util.ConcurrentModificationException，因为元素在使用的时候发生了并发的修改

```java
for(String x:list){
    if(x.equals("del"))
        list.remove(x);
}
```

解决方法：但只能删除一个"del"元素

```java
//解决：删除完毕马上使用break跳出，则不会触发报错
for(String x:list){
    if (x.equals("del")) {
         list.remove(x);
         break;
    }
}
```

#### iterator遍历
这种方式可以正常的循环及删除。但要注意的是，使用iterator的remove方法，如果用list的remove方法同样会报上面提到的ConcurrentModificationException错误。

```java
Iterator<String> it = list.iterator();
while(it.hasNext()){
    String x = it.next();
    if(x.equals("del")){
        it.remove();
    }
}
```

### 13、FailFast机制
上面提到的ConcurrentModificationException异常，都是有这个机制的存在，通过记录modCount参数来实现。在面对并发的修改时，迭代器很快就会完全失败，而不是冒着在将来某个不确定时间发生任意不确定行为的风险。

fail-fast 机制是java集合(Collection)中的一种错误机制。当多个线程对同一个集合的内容进行操作时，就可能会产生fail-fast事件。例如：当某一个线程A通过iterator去遍历某集合的过程中，若该集合的内容被其他线程所改变了；那么线程A遍历集合时，即出现expectedModCount != modCount 时，就会抛出ConcurrentModificationException异常，产生fail-fast事件。

```java
if (modCount != expectedModCount)
    throw new ConcurrentModificationException();
```

![[09_Attachments/Java/并发编程/Java_ArrayList_004.png]]
fail-fast 机制并不保证在不同步的修改下抛出异常，他只是尽最大努力去抛出，所以这种机制一般仅用于检测 bug

#### 解决 fail-fast的解决方案：
1. 在遍历过程中所有涉及到改变modCount值得地方全部加上synchronized或者直接使用Collections.synchronizedList，这样就可以解决(实际上Vector结构就是这样实现的)。但是不推荐，因为增删造成的同步锁可能会阻塞遍历操作。

```java
List<Integer> arrsyn = Collections.synchronizedList(arr);
```

2. 使用CopyOnWriteArrayList来替换ArrayList。推荐使用该方案。CopyOnWriteArrayList是兼顾了并发的线程安全

