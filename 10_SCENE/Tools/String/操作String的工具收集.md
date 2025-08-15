### 1.    需求：XXXX_AAA_BB 得到XXXX
<mark style="background: #FF5582A6;">方式一：</mark>`split()`是最高效简洁的方式，通过分隔符 `_`将字符串拆分为数组，并取第一个元素：
```java
String input = "XXXX_AAA_BB";
String[] parts = input.split("_"); // 以 "_" 为分隔符拆分
String result = parts[0];          // 取拆分后的第一部分
```
---
<mark style="background: #FF5582A6;">方式二：</mark>**`substring()` + `indexOf()`**
```java
String input = "XXXX_AAA_BB";
int firstUnderscoreIndex = input.indexOf("_"); // 定位第一个 "_" 的索引
if (firstUnderscoreIndex != -1) {
    String result = input.substring(0, firstUnderscoreIndex); // 截取开头到 "_" 前的子串
    System.out.println(result); // 输出: XXXX
} else {
    System.out.println("未找到分隔符"); // 处理无下划线的情况
}
```
**性能考量**​：
- • `split()` 适用于简单场景，但频繁操作时可能产生临时数组开销。
- • ` substring() ` + ` indexOf()` 更节省内存，适合大字符串处理。