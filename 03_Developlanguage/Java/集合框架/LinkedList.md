---
Title: Java 集合框架-LinkedList
Category: Java
tags:
  - java
  - 集合框架
---
## 概述

LinkedList 同时实现了 List 接口和 Deque 接口，也就是说它既可以看作一个顺序容器，又可以看作一个队列(_Queue_)，同时又可以看作一个栈(_Stack_)。这样看来， LinkedList 简直就是个全能冠军。当你需要使用栈或者队列时，可以考虑使用 LinkedList ，一方面是因为Java官方已经声明不建议使用 Stack 类，更遗憾的是，Java里根本没有一个叫做 Queue 的类(它是个接口名字)。关于栈或队列，现在的首选是 ArrayDeque ，它有着比 LinkedList (当作栈或队列使用时)有着更好的性能。

LinkedList 的实现方式决定了所有跟下标相关的操作都是线性时间，而在首段或者末尾删除元素只需要常数时间。为追求效率_LinkedList_没有实现同步(synchronized)，如果需要多个线程并发访问，可以先采用 `Collections.synchronizedList()` 方法对其进行包装。

## LinkedList实现

### 底层数据结构

LinkedList底层**通过双向链表实现**，本节将着重讲解插入和删除元素时双向链表的维护过程，也即是跟 List 接口相关的函数，而将 Queue 和 Stack 以及 Deque 相关的知识放在下一节讲。双向链表的每个节点用内部类 Node 表示。 LinkedList 通过 `first` 和 `last` 引用分别指向链表的第一个和最后一个元素。注意这里没有所谓的哑元，当链表为空的时候 `first` 和 `last` 都指向 `null`。

```java
    transient int size = 0;

    /**
     * Pointer to first node.
     * Invariant: (first == null && last == null) ||
     *            (first.prev == null && first.item != null)
     */
    transient Node<E> first;

    /**
     * Pointer to last node.
     * Invariant: (first == null && last == null) ||
     *            (last.next == null && last.item != null)
     */
    transient Node<E> last;
```

其中Node是私有的内部类:

```java
    private static class Node<E> {
        E item;
        Node<E> next;
        Node<E> prev;

        Node(Node<E> prev, E element, Node<E> next) {
            this.item = element;
            this.next = next;
            this.prev = prev;
        }
    }
```

### 构造函数

```java
    /**
     * Constructs an empty list.
     */
    public LinkedList() {
    }

    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param  c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public LinkedList(Collection<? extends E> c) {
        this();
        addAll(c);
    }
```

### 增删改查方法
  
#### **1）增**  
  
可以调用 add 方法添加元素：  
  
```java  
list.add("呵呵1");  
list.add("呵呵2");  
list.add("呵呵3");  
```  
  
add 方法内部其实调用的是 linkLast 方法：  
  
```java  
/**
 * 将指定的元素添加到列表的尾部。
 *
 * @param e 要添加到列表的元素
 * @return 始终为 true（根据 Java 集合框架规范）
 */
public boolean add(E e) {
    linkLast(e); // 在列表的尾部添加元素
    return true; // 添加元素成功，返回 true
}
```  
  
linkLast，顾名思义，就是在链表的尾部添加元素：  
  
```java  
/**
 * 在列表的尾部添加指定的元素。
 *
 * @param e 要添加到列表的元素
 */
void linkLast(E e) {
    final Node<E> l = last; // 获取链表的最后一个节点
    final Node<E> newNode = new Node<>(l, e, null); // 创建一个新的节点，并将其设置为链表的最后一个节点
    last = newNode; // 将新的节点设置为链表的最后一个节点
    if (l == null) // 如果链表为空，则将新节点设置为头节点
        first = newNode;
    else
        l.next = newNode; // 否则将新节点链接到链表的尾部
    size++; // 增加链表的元素个数
}
```  
  
- 添加第一个元素的时候，first 和 last 都为 null。  
- 然后新建一个节点 newNode，它的 prev 和 next 也为 null。  
- 然后把 last 和 first 都赋值为 newNode。  
  
此时还不能称之为链表，因为前后节点都是断裂的。    
  
- 添加第二个元素的时候，first 和 last 都指向的是第一个节点。  
- 然后新建一个节点 newNode，它的 prev 指向的是第一个节点，next 为 null。  
- 然后把第一个节点的 next 赋值为 newNode。  
  
此时的链表还不完整。    

- 添加第三个元素的时候，first 指向的是第一个节点，last 指向的是最后一个节点。  
- 然后新建一个节点 newNode，它的 prev 指向的是第二个节点，next 为 null。  
- 然后把第二个节点的 next 赋值为 newNode。  
  
此时的链表已经完整了。    
  
增还可以演化成另外两个版本：  
  
- `addFirst()` 方法将元素添加到第一位；  
- `addLast()` 方法将元素添加到末尾。  
  
addFirst 内部其实调用的是 linkFirst：  
  
```java  
/**  
/**
 * 在列表的开头添加指定的元素。
 *
 * @param e 要添加到列表的元素
 */
public void addFirst(E e) {
    linkFirst(e); // 在列表的开头添加元素
}
```  
  
linkFirst 负责把新的节点设为 first，并将新的 first 的 next 更新为之前的 first。  
  
```java  
/**
 * 在列表的开头添加指定的元素。
 *
 * @param e 要添加到列表的元素
 */
private void linkFirst(E e) {
    final Node<E> f = first; // 获取链表的第一个节点
    final Node<E> newNode = new Node<>(null, e, f); // 创建一个新的节点，并将其设置为链表的第一个节点
    first = newNode; // 将新的节点设置为链表的第一个节点
    if (f == null) // 如果链表为空，则将新节点设置为尾节点
        last = newNode;
    else
        f.prev = newNode; // 否则将新节点链接到链表的头部
    size++; // 增加链表的元素个数
}
```  
  
addLast 的内核其实和 addFirst 差不多，内部调用的是 linkLast 方法，前面分析过了。  
  
```java  
/**
 * 在列表的尾部添加指定的元素。
 *
 * @param e 要添加到列表的元素
 * @return 始终为 true（根据 Java 集合框架规范）
 */
public boolean addLast(E e) {
    linkLast(e); // 在列表的尾部添加元素
    return true; // 添加元素成功，返回 true
}
```  
  
  
#### **2）删**  
  
我这个删的招式还挺多的：  
  
- `remove()`：删除第一个节点  
- `remove(int)`：删除指定位置的节点  
- `remove(Object)`：删除指定元素的节点  
- `removeFirst()`：删除第一个节点  
- `removeLast()`：删除最后一个节点  
  
`remove()` 内部调用的是 `removeFirst()`，所以这两个招式的功效一样。  
  
`remove(int)` 内部其实调用的是 unlink 方法。  
  
```java  
/**
 * 删除指定位置上的元素。
 *
 * @param index 要删除的元素的索引
 * @return 从列表中删除的元素
 * @throws IndexOutOfBoundsException 如果索引越界（index &lt; 0 || index &gt;= size()）
 */
public E remove(int index) {
    checkElementIndex(index); // 检查索引是否越界
    return unlink(node(index)); // 删除指定位置的节点，并返回节点的元素
}
```  
  
unlink 方法其实很好理解，就是更新当前节点的 next 和 prev，然后把当前节点上的元素设为 null。  
  
```java  
/**
 * 从链表中删除指定节点。
 *
 * @param x 要删除的节点
 * @return 从链表中删除的节点的元素
 */
E unlink(Node<E> x) {
    final E element = x.item; // 获取要删除节点的元素
    final Node<E> next = x.next; // 获取要删除节点的下一个节点
    final Node<E> prev = x.prev; // 获取要删除节点的上一个节点

    if (prev == null) { // 如果要删除节点是第一个节点
        first = next; // 将链表的头节点设置为要删除节点的下一个节点
    } else {
        prev.next = next; // 将要删除节点的上一个节点指向要删除节点的下一个节点
        x.prev = null; // 将要删除节点的上一个节点设置为空
    }

    if (next == null) { // 如果要删除节点是最后一个节点
        last = prev; // 将链表的尾节点设置为要删除节点的上一个节点
    } else {
        next.prev = prev; // 将要删除节点的下一个节点指向要删除节点的上一个节点
        x.next = null; // 将要删除节点的下一个节点设置为空
    }

    x.item = null; // 将要删除节点的元素设置为空
    size--; // 减少链表的元素个数
    return element; // 返回被删除节点的元素
}
```  
  
remove(Object) 内部也调用了 unlink 方法，只不过在此之前要先找到元素所在的节点：  
  
```java  
/**
 * 从链表中删除指定元素。
 *
 * @param o 要从链表中删除的元素
 * @return 如果链表包含指定元素，则返回 true；否则返回 false
 */
public boolean remove(Object o) {
    if (o == null) { // 如果要删除的元素为 null
        for (Node<E> x = first; x != null; x = x.next) { // 遍历链表
            if (x.item == null) { // 如果节点的元素为 null
                unlink(x); // 删除节点
                return true; // 返回 true 表示删除成功
            }
        }
    } else { // 如果要删除的元素不为 null
        for (Node<E> x = first; x != null; x = x.next) { // 遍历链表
            if (o.equals(x.item)) { // 如果节点的元素等于要删除的元素
                unlink(x); // 删除节点
                return true; // 返回 true 表示删除成功
            }
        }
    }
    return false; // 如果链表中不包含要删除的元素，则返回 false 表示删除失败
}
```  
  
元素为 null 的时候，必须使用 == 来判断；元素为非 null 的时候，要使用 equals 来判断。  
  
removeFirst 内部调用的是 unlinkFirst 方法：  
  
```java  
/**
 * 从链表中删除第一个元素并返回它。
 * 如果链表为空，则抛出 NoSuchElementException 异常。
 *
 * @return 从链表中删除的第一个元素
 * @throws NoSuchElementException 如果链表为空
 */
public E removeFirst() {
    final Node<E> f = first; // 获取链表的第一个节点
    if (f == null) // 如果链表为空
        throw new NoSuchElementException(); // 抛出 NoSuchElementException 异常
    return unlinkFirst(f); // 调用 unlinkFirst 方法删除第一个节点并返回它的元素
}
```  
  
unlinkFirst 负责的就是把第一个节点毁尸灭迹，并且捎带把后一个节点的 prev 设为 null。  
  
```java  
/**
 * 删除链表中的第一个节点并返回它的元素。
 *
 * @param f 要删除的第一个节点
 * @return 被删除节点的元素
 */
private E unlinkFirst(Node<E> f) {
    final E element = f.item; // 获取要删除的节点的元素
    final Node<E> next = f.next; // 获取要删除的节点的下一个节点
    f.item = null; // 将要删除的节点的元素设置为 null
    f.next = null; // 将要删除的节点的下一个节点设置为 null
    first = next; // 将链表的头节点设置为要删除的节点的下一个节点
    if (next == null) // 如果链表只有一个节点
        last = null; // 将链表的尾节点设置为 null
    else
        next.prev = null; // 将要删除节点的下一个节点的前驱设置为 null
    size--; // 减少链表的大小
    return element; // 返回被删除节点的元素
} 
```  
  
#### **3）改**  
  
可以调用 `set()` 方法来更新元素：  
  
```java  
list.set(0, "呵呵4");  
```  
  
来看一下 `set()` 方法：  
  
```java  
/**
 * 将链表中指定位置的元素替换为指定元素，并返回原来的元素。
 *
 * @param index 要替换元素的位置（从 0 开始）
 * @param element 要插入的元素
 * @return 替换前的元素
 * @throws IndexOutOfBoundsException 如果索引超出范围（index < 0 || index >= size()）
 */
public E set(int index, E element) {
    checkElementIndex(index); // 检查索引是否超出范围
    Node<E> x = node(index); // 获取要替换的节点
    E oldVal = x.item; // 获取要替换节点的元素
    x.item = element; // 将要替换的节点的元素设置为指定元素
    return oldVal; // 返回替换前的元素
}
```  
  
来看一下node方法：  
  
```java  
/**
 * 获取链表中指定位置的节点。
 *
 * @param index 节点的位置（从 0 开始）
 * @return 指定位置的节点
 * @throws IndexOutOfBoundsException 如果索引超出范围（index < 0 || index >= size()）
 */
Node<E> node(int index) {
    if (index < (size >> 1)) { // 如果索引在链表的前半部分
        Node<E> x = first;
        for (int i = 0; i < index; i++) // 从头节点开始向后遍历链表，直到找到指定位置的节点
            x = x.next;
        return x; // 返回指定位置的节点
    } else { // 如果索引在链表的后半部分
        Node<E> x = last;
        for (int i = size - 1; i > index; i--) // 从尾节点开始向前遍历链表，直到找到指定位置的节点
            x = x.prev;
        return x; // 返回指定位置的节点
    }
}
```  
  
`size >> 1`：也就是右移一位，相当于除以 2。对于计算机来说，移位比除法运算效率更高，因为数据在计算机内部都是以二进制存储的。  
  
换句话说，node 方法会对下标进行一个初步判断，如果靠近前半截，就从下标 0 开始遍历；如果靠近后半截，就从末尾开始遍历，这样可以提高效率，最大能提高一半的效率。  
  
找到指定下标的节点就简单了，直接把原有节点的元素替换成新的节点就 OK 了，prev 和 next 都不用改动。  
  
#### **4）查**  
  
查可以分为两种：  
  
- indexOf(Object)：查找某个元素所在的位置  
- get(int)：查找某个位置上的元素  
  
来看一下 indexOf 方法的源码。  
  
```java  
/**
 * 返回链表中首次出现指定元素的位置，如果不存在该元素则返回 -1。
 *
 * @param o 要查找的元素
 * @return 首次出现指定元素的位置，如果不存在该元素则返回 -1
 */
public int indexOf(Object o) {
    int index = 0; // 初始化索引为 0
    if (o == null) { // 如果要查找的元素为 null
        for (Node<E> x = first; x != null; x = x.next) { // 从头节点开始向后遍历链表
            if (x.item == null) // 如果找到了要查找的元素
                return index; // 返回该元素的索引
            index++; // 索引加 1
        }
    } else { // 如果要查找的元素不为 null
        for (Node<E> x = first; x != null; x = x.next) { // 从头节点开始向后遍历链表
            if (o.equals(x.item)) // 如果找到了要查找的元素
                return index; // 返回该元素的索引
            index++; // 索引加 1
        }
    }
    return -1; // 如果没有找到要查找的元素，则返回 -1
}
```  
  
get 方法的内核其实还是 node 方法，node 方法之前已经说明过了，这里略过。  
  
```java  
public E get(int index) {
    checkElementIndex(index);
    return node(index).item;
} 
```  
  
其实，查还可以演化为其他的一些，比如说：  
  
- `getFirst()` 方法用于获取第一个元素；  
- `getLast()` 方法用于获取最后一个元素；  
- `poll()` 和 `pollFirst()` 方法用于删除并返回第一个元素（两个方法尽管名字不同，但方法体是完全相同的）；  
- `pollLast()` 方法用于删除并返回最后一个元素；  
- `peekFirst()` 方法用于返回但不删除第一个元素。