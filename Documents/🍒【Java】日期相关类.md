## java.util.Date
### 简单介绍

java.util.Date 是 Java 中用于表示特定瞬间（时间点）的类。

>[!tips]
从 Java 8 开始，该类在很大程度上已被 `java.time` 包下的新日期和时间 API 所取代，但在旧的 Java 代码和一些遗留系统中仍然广泛使用。下面将从基本介绍、常用构造方法、常用方法、局限性以及与新日期时间 API 的转换等方面详细介绍 `java.util.Date` 类。

优点：简单易用，直接存储日期和时间
缺点：没有提供日期和时间的格式化和解析功能
### 常用构造方法
直接获取当前时间
```java
Date date = new Date();
```

根据时间戳构建
```java
long milliseconds = 1609459200000L; // 2021-01-01 00:00:00 的毫秒数
Date specificDate = new Date(milliseconds);
```
### 常用方法
**`long getTime()`**: 返回自 1970 年 1 月 1 日 00:00:00 GMT 以来此 `Date` 对象表示的毫秒数。
**`void setTime(long time)`**： 根据时间戳设置 `Date` 对象表示的时间。
**`boolean after(Date when)`**: 测试此 `Date` 对象表示的时间是否在指定 `Date` 对象表示的时间之后。
**`boolean before(Date when)`**: 测试此 `Date` 对象表示的时间是否在指定 `Date` 对象表示的时间之前。
### 局限性
- **可变性**：`java.util.Date` 类是可变的，这意味着在多线程环境下使用时可能会出现线程安全问题。一旦创建了 `Date` 对象，就可以使用 `setTime()` 方法改变其表示的时间。
- **缺乏时区支持**：`Date` 类本身并不包含时区信息，它只是简单地存储自纪元以来的毫秒数。这使得在处理不同时区的日期和时间时变得复杂，需要结合 `java.util.TimeZone` 类来处理。
- **设计缺陷**：`Date` 类的一些方法（如 `getYear()`、`getMonth()` 等）已经被标记为过时，因为它们的设计存在问题，容易引起混淆。例如，`getYear()` 返回的是从 1900 年开始计算的年份偏移量，而不是实际的年份。
## java.util.Calendar
### 基本介绍
`ava.util.Calendar` 是 Java 中用于处理日期和时间的抽象类，它提供了比 `java.util.Date` 更丰富的日期和时间操作功能。
由于 `Date` 类存在一些设计缺陷，`Calendar` 类弥补了这些不足，使得日期和时间的处理更加灵活和方便。
优点：提供了丰富的日期和时间操作功能，可以处理复杂的日期计算
缺点：API 较为复杂，不如 java.time 使用起来简洁
```java
import java.util.Calendar;

public class Main {
    public static void main(String[] args) {
        Calendar calendar = Calendar.getInstance();
        System.out.println(calendar.getTime());
    }
}
```
### 常用字段
`Calendar` 类定义了许多静态常量字段，用于表示日期和时间的各个部分，常见的有：
- `Calendar.YEAR`：表示年份。
- `Calendar.MONTH`：表示月份，从 0 开始计数，即 0 表示一月，11 表示十二月。
- `Calendar.DAY_OF_MONTH`：表示一个月中的第几天。
- `Calendar.HOUR_OF_DAY`：表示 24 小时制的小时数。
- `Calendar.MINUTE`：表示分钟数。
- `Calendar.SECOND`：表示秒数。
- `Calendar.DAY_OF_WEEK`：表示一周中的第几天，1 表示星期日，2 表示星期一，以此类推。
### 常用构造和静态方法
**`static Calendar getInstance()`**：返回一个默认时区和语言环境的 `Calendar` 实例，通常使用这个方法来获取 `Calendar` 对象。
**`int get(int field)`**：返回指定日历字段的值。
**`void set(int field, int value)`**：将指定的日历字段设置为给定的值。
```java
import java.util.Calendar;

public class CalendarExample {
    public static void main(String[] args) {
        Calendar calendar = Calendar.getInstance();
        calendar.set(Calendar.YEAR, 2024); // 设置年份为 2024
        calendar.set(Calendar.MONTH, Calendar.JANUARY); // 设置月份为一月
        calendar.set(Calendar.DAY_OF_MONTH, 1); // 设置日期为 1 号
        System.out.println(calendar.getTime());
    }
}
```
**`void add(int field, int amount)`**：根据日历的规则，为指定的日历字段添加或减去指定的时间量。
```java
import java.util.Calendar;

public class CalendarExample {
    public static void main(String[] args) {
        Calendar calendar = Calendar.getInstance();
        calendar.add(Calendar.DAY_OF_MONTH, 7); // 增加 7 天
        System.out.println(calendar.getTime());
    }
}
```
**`Date getTime()`**：返回一个表示此 `Calendar` 时间值（从纪元至现在的毫秒偏移量）的 `Date` 对象。
**`void setTime(Date date)`**：使用给定的 `Date` 对象设置此 `Calendar` 的时间。
### 局限性
- **可变性**：`Calendar` 类是可变的，这在多线程环境下使用时可能会导致线程安全问题。
- **月份索引问题**：`Calendar` 类中月份是从 0 开始计数的，这容易引起混淆，在编写代码时需要特别注意。
- **API 设计不够简洁**：`Calendar` 类的方法和字段较多，使用起来不够直观和简洁，增加了学习和使用的成本。
### java.time.LocalDate
`java.time.LocalDate` 是 Java 8 引入的 `java.time` 包中的一个类，用于表示不带时区的日期，例如 `2024-12-31`。它是不可变且线程安全的，相比旧的 `java.util.Date` 和 `java.util.Calendar` 类，提供了更简洁、更直观的日期处理方式。
## 创建 `LocalDate` 实例
### 使用 `now()` 方法获取当前日期
```java
import java.time.LocalDate;

public class LocalDateExample {
    public static void main(String[] args) {
        LocalDate currentDate = LocalDate.now();
        System.out.println("当前日期: " + currentDate);
    }
}
```
### 使用 `of()` 方法指定日期
```java
import java.time.LocalDate;

public class LocalDateExample {
    public static void main(String[] args) {
        LocalDate specificDate = LocalDate.of(2024, 10, 15);
        System.out.println("指定日期: " + specificDate);
    }
}
```
`of()` 方法接收年、月、日作为参数，创建指定的日期实例。
### 使用 `parse()` 方法从字符串解析日期
```java
import java.time.LocalDate;

public class LocalDateExample {
    public static void main(String[] args) {
        String dateStr = "2024-10-15";
        LocalDate parsedDate = LocalDate.parse(dateStr);
        System.out.println("解析后的日期: " + parsedDate);
    }
}
```
`parse()` 方法默认按照 `ISO_LOCAL_DATE` 格式（`yyyy-MM-dd`）解析字符串。
## 常用方法
### 获取日期信息
```java
import java.time.LocalDate;

public class LocalDateExample {
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 10, 15);
        int year = date.getYear();
        int month = date.getMonthValue();
        int day = date.getDayOfMonth();
        System.out.printf("年份: %d, 月份: %d, 日期: %d%n", year, month, day);
    }
}
```
`getYear()`、`getMonthValue()` 和 `getDayOfMonth()` 分别用于获取年份、月份和日期。
### 判断日期关系
```java
import java.time.LocalDate;

public class LocalDateExample {
    public static void main(String[] args) {
        LocalDate date1 = LocalDate.of(2024, 10, 15);
        LocalDate date2 = LocalDate.of(2024, 10, 20);
        boolean isBefore = date1.isBefore(date2);
        boolean isAfter = date1.isAfter(date2);
        boolean isEqual = date1.isEqual(date2);
        System.out.printf("date1 在 date2 之前: %b%n", isBefore);
        System.out.printf("date1 在 date2 之后: %b%n", isAfter);
        System.out.printf("date1 等于 date2: %b%n", isEqual);
    }
}
```
`isBefore()`、`isAfter()` 和 `isEqual()` 用于判断日期之间的先后关系和是否相等。
### 日期计算
#### 增加或减少日期
```java
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

public class LocalDateExample {
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 10, 15);
        LocalDate nextDay = date.plusDays(1);
        LocalDate previousMonth = date.minusMonths(1);
        System.out.println("增加一天后的日期: " + nextDay);
        System.out.println("减少一个月后的日期: " + previousMonth);

        // 使用 ChronoUnit 进行计算
        LocalDate futureDate = date.plus(2, ChronoUnit.WEEKS);
        System.out.println("增加两周后的日期: " + futureDate);
    }
}
```
`plusXxx()` 和 `minusXxx()` 方法用于增加或减少指定的时间单位，也可以使用 `plus()` 和 `minus()` 结合 `ChronoUnit` 进行更灵活的计算。
#### 计算两个日期之间的差值
```java
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

public class LocalDateExample {
    public static void main(String[] args) {
        LocalDate startDate = LocalDate.of(2024, 10, 1);
        LocalDate endDate = LocalDate.of(2024, 10, 15);
        long daysBetween = ChronoUnit.DAYS.between(startDate, endDate);
        System.out.println("两个日期之间相差的天数: " + daysBetween);
    }
}
```
`ChronoUnit` 的 `between()` 方法用于计算两个日期之间的差值。
### 格式化与解析
#### 格式化日期
```java
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

public class LocalDateExample {
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 10, 15);
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy/MM/dd");
        String formattedDate = date.format(formatter);
        System.out.println("格式化后的日期: " + formattedDate);
    }
}
```
使用 `DateTimeFormatter` 可以将 `LocalDate` 格式化为指定格式的字符串。
#### 解析自定义格式的日期字符串
```java
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

public class LocalDateExample {
    public static void main(String[] args) {
        String dateStr = "2024/10/15";
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy/MM/dd");
        LocalDate parsedDate = LocalDate.parse(dateStr, formatter);
        System.out.println("解析后的日期: " + parsedDate);
    }
}
```
同样使用 `DateTimeFormatter` 可以解析自定义格式的日期字符串为 `LocalDate` 实例。
## java.time.LocalDateTime
`LocalDateTime` 是 Java 8 引入的 `java.time` 包中的一个类，用于表示不带时区信息的日期和时间，它结合了 `LocalDate` 和 `LocalTime` 的功能，可以精确到纳秒级别。`LocalDateTime` 是不可变且线程安全的，提供了简洁、高效且功能强大的日期时间处理方式。

### 创建 `LocalDateTime` 实例
#### 1.1 使用 `now()` 方法获取当前日期和时间
时区可以通过 TimeZone.setDefault 设置
```java
import java.time.LocalDateTime;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        LocalDateTime now = LocalDateTime.now();
        System.out.println("当前日期和时间: " + now);
    }
}
```
#### 1.2 使用 `of()` 方法指定日期和时间
```java
import java.time.LocalDateTime;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        // 指定年、月、日、时、分、秒
        LocalDateTime specificDateTime = LocalDateTime.of(2024, 10, 15, 14, 30, 0);
        System.out.println("指定的日期和时间: " + specificDateTime);
    }
}
```
#### 1.3 结合 `LocalDate` 和 `LocalTime` 创建
```java
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        LocalDate date = LocalDate.of(2024, 10, 15);
        LocalTime time = LocalTime.of(14, 30);
        LocalDateTime dateTime = LocalDateTime.of(date, time);
        System.out.println("结合日期和时间创建的实例: " + dateTime);
    }
}
```
#### 1.4 使用 `parse()` 方法从字符串解析
```java
import java.time.LocalDateTime;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        String dateTimeStr = "2024-10-15T14:30:00";
        LocalDateTime parsedDateTime = LocalDateTime.parse(dateTimeStr);
        System.out.println("解析后的日期和时间: " + parsedDateTime);
    }
}
```
默认按照 `ISO_LOCAL_DATE_TIME` 格式（`yyyy-MM-ddTHH:mm:ss`）解析，`T` 是日期和时间的分隔符。
## 常用方法
### 2.1 获取日期和时间信息
```java
import java.time.LocalDateTime;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        LocalDateTime dateTime = LocalDateTime.of(2024, 10, 15, 14, 30, 0);
        int year = dateTime.getYear();
        int month = dateTime.getMonthValue();
        int day = dateTime.getDayOfMonth();
        int hour = dateTime.getHour();
        int minute = dateTime.getMinute();
        int second = dateTime.getSecond();
        System.out.printf("年: %d, 月: %d, 日: %d, 时: %d, 分: %d, 秒: %d%n", year, month, day, hour, minute, second);
    }
}
```
### 2.2 判断日期和时间关系
```java
import java.time.LocalDateTime;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        LocalDateTime dateTime1 = LocalDateTime.of(2024, 10, 15, 14, 30, 0);
        LocalDateTime dateTime2 = LocalDateTime.of(2024, 10, 16, 10, 0, 0);
        boolean isBefore = dateTime1.isBefore(dateTime2);
        boolean isAfter = dateTime1.isAfter(dateTime2);
        boolean isEqual = dateTime1.isEqual(dateTime2);
        System.out.printf("dateTime1 在 dateTime2 之前: %b%n", isBefore);
        System.out.printf("dateTime1 在 dateTime2 之后: %b%n", isAfter);
        System.out.printf("dateTime1 等于 dateTime2: %b%n", isEqual);
    }
}
```
## 日期和时间计算
### 3.1 增加或减少时间
```java
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        LocalDateTime dateTime = LocalDateTime.of(2024, 10, 15, 14, 30, 0);
        LocalDateTime nextHour = dateTime.plusHours(1);
        LocalDateTime previousDay = dateTime.minusDays(1);
        System.out.println("增加一小时后的日期和时间: " + nextHour);
        System.out.println("减少一天后的日期和时间: " + previousDay);

        // 使用 ChronoUnit 进行计算
        LocalDateTime futureDateTime = dateTime.plus(2, ChronoUnit.WEEKS);
        System.out.println("增加两周后的日期和时间: " + futureDateTime);
    }
}
```
### 3.2 计算两个 `LocalDateTime` 之间的差值
```java
import java.time.Duration;
import java.time.LocalDateTime;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        LocalDateTime startDateTime = LocalDateTime.of(2024, 10, 15, 10, 0, 0);
        LocalDateTime endDateTime = LocalDateTime.of(2024, 10, 15, 14, 30, 0);
        Duration duration = Duration.between(startDateTime, endDateTime);
        long hours = duration.toHours();
        long minutes = duration.toMinutesPart();
        System.out.printf("时间间隔: %d 小时 %d 分钟%n", hours, minutes);
    }
}
```
## 格式化与解析
### 4.1 格式化日期和时间
```java
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        LocalDateTime dateTime = LocalDateTime.of(2024, 10, 15, 14, 30, 0);
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        String formattedDateTime = dateTime.format(formatter);
        System.out.println("格式化后的日期和时间: " + formattedDateTime);
    }
}
```
### 4.2 解析自定义格式的日期和时间字符串
```java
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        String dateTimeStr = "2024-10-15 14:30:00";
        DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd HH:mm:ss");
        LocalDateTime parsedDateTime = LocalDateTime.parse(dateTimeStr, formatter);
        System.out.println("解析后的日期和时间: " + parsedDateTime);
    }
}
```
## 与其他时间类的转换

### 5.1 转换为 `LocalDate` 和 `LocalTime`
```java
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.time.LocalTime;

public class LocalDateTimeExample {
    public static void main(String[] args) {
        LocalDateTime dateTime = LocalDateTime.of(2024, 10, 15, 14, 30, 0);
        LocalDate date = dateTime.toLocalDate();
        LocalTime time = dateTime.toLocalTime();
        System.out.println("转换后的日期: " + date);
        System.out.println("转换后的时间: " + time);
    }
}
```
`LocalDateTime` 类为处理日期和时间提供了丰富且便捷的功能，在 Java 8 及后续版本的开发中，是处理日期和时间的理想选择。 