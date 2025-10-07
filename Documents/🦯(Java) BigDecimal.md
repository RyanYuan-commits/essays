---
finished: "true"
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
```java
BigDecimal bd = new BigDecimal("123.4567");
System.out.println(bd.multiply(bd)); // 15241.55677489
```

`BigDecimal` 可以表示一个任意大小, 且精度完全准确的浮点数.

---

```java
BigDecimal d1 = new BigDecimal("123.45");
BigDecimal d2 = new BigDecimal("123.4500");
BigDecimal d3 = new BigDecimal("1234500");
System.out.println(d1.scale()); // 2,两位小数
System.out.println(d2.scale()); // 4
System.out.println(d3.scale()); // 0
```

`BigDecimal` 由两个部分组成:

- `unscaledValue`: 一个整数, 表示没有小数点的数字;
- `scale`: 一个整数, 表示小数点后有多少位

---

```java
BigDecimal d1 = new BigDecimal("123.4500");
BigDecimal d2 = d1.stripTrailingZeros();
System.out.println(d1.scale()); // 4
System.out.println(d2.scale()); // 2,因为去掉了00

BigDecimal d3 = new BigDecimal("1234500");
BigDecimal d4 = d3.stripTrailingZeros();
System.out.println(d3.scale()); // 0
System.out.println(d4.scale()); // -2
```

`stripTrailingZeros()` 会调整上面提到的两个部分, 来移除尾部的 0:

- 对于整数, 比如 12300, 经过这个方法后, `unscaledValue` 变为 123, 而 `scale` 变为 -2;
- 对于小数, 比如 1.10, `unscaledValue` 从 110 变为 11, `scale` 从 2 变为 1.

---

```java
BigDecimal d1 = new BigDecimal("123.456789");
BigDecimal d2 = d1.setScale(4, RoundingMode.HALF_UP); // 四舍五入，123.4568
BigDecimal d3 = d1.setScale(4, RoundingMode.DOWN); // 直接截断，123.4567
System.out.println(d2);
System.out.println(d3);
```

可以设置 `BigDecimal` 的 `scale`, 如果精度比原始值低, 可以指定进行四舍五入或者直接截断.

---

```java
BigDecimal d1 = new BigDecimal("123.456");
BigDecimal d2 = new BigDecimal("23.456789");
BigDecimal d3 = d1.divide(d2, 10, RoundingMode.HALF_UP); // 保留10位小数并四舍五入
BigDecimal d4 = d1.divide(d2); // 报错：ArithmeticException，因为除不尽
```

对 `BigDecimal` 做加减乘的时候, 精度不会丢失, 但是做除法的时候, 存在除不尽的情况, 这时候就必须指定精度和如何截断.

---

```java
BigDecimal n = new BigDecimal("12.345");
BigDecimal m = new BigDecimal("0.12");
BigDecimal[] dr = n.divideAndRemainder(m);
System.out.println(dr[0]); // 102
System.out.println(dr[1]); // 0.105
```

还支持做除法的同时求余数.

---

```java
BigDecimal d1 = new BigDecimal("123.456");
BigDecimal d2 = new BigDecimal("123.45600");
System.out.println(d1.equals(d2)); // false,因为scale不同
System.out.println(d1.equals(d2.stripTrailingZeros())); // true,因为d2去除尾部0后scale变为3
System.out.println(d1.compareTo(d2)); // 0 = 相等, -1 = d1 < d2, 1 = d1 > d2
```

如果使用 `equals()` 方法来比较两个 `BigDecimal`, 不仅值要相同, `scale` 也要相同;

应该使用 `compare()` 方法来比较, 0 = 相等, -1 = d1 < d2, 1 = d1 > d2.
