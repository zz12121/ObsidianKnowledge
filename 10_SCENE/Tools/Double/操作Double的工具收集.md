#tools
### 1 .需求：解决double转string 有.0 
在 Java 中，当 `double` 类型数值的小数部分为 0（如 `123.0`）时，转换为 `String` 会保留 `.0`（例如 `"123.0"`），同时保留有效小数位（如 `3.23423` 不受影响）
<mark style="background: #FF5582A6;">方式：</mark> 使用 `DecimalFormat` 模式匹配
通过 `"0.##"` 格式模式，自动省略整数的小数部分 `.0`，保留非零小数位：
```java
double num1 = 123.0;    // 整数
double num2 = 3.23423;   // 含有效小数

DecimalFormat df = new DecimalFormat("0.##");
String result1 = df.format(num1); // 输出 "123"
String result2 = df.format(num2); // 输出 "3.23423"
```
**原理**​：
`#`：可选数字位，小数部分为零时省略小数点及零。
`0`：强制显示数字位（整数部分至少一位，避免 `.123` 格式）。
**局限性**：
若返回要求不能是科学计数法（如 `1.23E5`），优先用 `BigDecimal` 的 `toPlainString()` 转换。
解决方式：[操作Bigdecimal的工具收集](10_SCENE/Tools/Bigdecimal/操作Bigdecimal的工具收集.md)

