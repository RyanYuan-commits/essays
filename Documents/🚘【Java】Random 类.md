[# Random 类
在 Java 中，Random 类位于 java.util 包中，用于生成伪随机数。它提供了多种方法来生成不同类型的随机值，例如整数、浮点数、布尔值等。
## Random 类的核心特点
**伪随机数**：生成的随机数基于种子（seed），是可预测的。如果使用相同的种子初始化，则会生成相同的随机序列。
**线程不安全**：Random 类不是线程安全的。如果在多线程环境下使用，需要显式加锁或考虑使用 ThreadLocalRandom。
**范围支持**：可生成不同范围的随机值（整数、浮点数等）。
## 使用示例
```java
import java.util.Random;
public class RandomExample {
    public static void main(String[] args) {
        Random random = new Random();
        // 生成随机整数
        System.out.println("Random Integer: " + random.nextInt());
        // 生成 0-99 范围内的随机整数
        System.out.println("Random Integer [0-99]: " + random.nextInt(100));
        // 生成随机布尔值
        System.out.println("Random Boolean: " + random.nextBoolean());
        // 生成随机浮点数
        System.out.println("Random Float: " + random.nextFloat());
        // 生成随机双精度数
        System.out.println("Random Double: " + random.nextDouble());
        // 生成正态分布随机数
        System.out.println("Random Gaussian: " + random.nextGaussian());
    }

}
```
