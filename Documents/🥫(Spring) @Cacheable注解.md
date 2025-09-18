用于将方法的返回值缓存下来, 当方法被调用的时候, 会首先检查缓存中是否已有该方法的返回值.

## 1 @Cacheable 主要属性

`value/cacheNames`: 指定缓存的名称, 类似库的概念;

`key`: 定义缓存的键, Spring 会基于方法参数生成一个默认键, 但是可以用 `key` 属性来定义, 它支持使用 Spring Expression Language (SpEL) 来动态生成键, 例如 `@Cacheable(value="users", key="#uid")`.

`condition`: 定义一个 SpEL 表达式, 当表达式为 `true` 的时候缓存才会生效, 如 `@Cacheable(value = "users", condition = "#id > 10")`.

`unless`: 定义一个 SpEL 表达式, 如果其为 `true`, 则不将方法的缓存值放入缓存, 通常用于处理异常和空值.

## 2 @Cacheable 案例

```java
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    @Cacheable(value = "users", key = "#id")
    public User findUserById(Long id) {
        System.out.println("Finding user with ID: " + id);
        // 模拟一个耗时的数据库查询
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        return new User(id, "User-" + id);
    }
}

// 假设 User 是一个简单的 JavaBean
class User {
    private Long id;
    private String name;
    
    // 构造函数、Getter和Setter...
    public User(Long id, String name) {
        this.id = id;
        this.name = name;
    }

    public Long getId() {
        return id;
    }

    public String getName() {
        return name;
    }
}
```
