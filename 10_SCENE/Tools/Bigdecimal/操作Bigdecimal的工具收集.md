### 1.需求：高精度的数值转String，返回不是科学计数法
将 `double` 转换为 `String` 时去除整数结尾的 `.0`，同时保留有效小数位（如 `3.23423` 不受影响），要求数值不能返回科学计数法（如 `1.23E5`）
<mark style="background: #FF5582A6;">方式：</mark>利用 `BigDecimal` 的精度控制方法，彻底移除末尾零
```java
double num = 100.0;
BigDecimal bd = new BigDecimal(String.valueOf(num));
String result = bd.stripTrailingZeros().toPlainString(); // 输出 "100"
```
**关键方法**​：
`stripTrailingZeros()`：移除小数部分末尾的零（如 `100.00`→ `100`）。
`toPlainString()`：避免返回科学计数法（如 `1E+2` → `100`）