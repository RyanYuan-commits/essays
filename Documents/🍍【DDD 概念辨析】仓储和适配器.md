在 DDD 的设计方法中，领域层做到了只关心领域服务实现，最能体现这样设计的就是仓库和适配器的设计。
通常在 Service + 数据模型的设计中，会在 Service 中引入 Redis、RPC、配置中心等各类其他外部服务。但在 DDD 中，通过仓储和适配器以及基础设施层的定义，解耦了这部分内容。

![[仓储和适配器.png|600]]
特征：
1. 封装持久化操作：`Repository` 负责封装所有与数据源交互的操作，如创建、读取、更新和删除（CRUD）操作。这样，领域层的代码就可以避免直接处理数据库或其他存储机制的复杂性。
2. 领域对象的集合管理：`Repository` 通常被视为领域对象的集合，提供了查询和过滤这些对象的方法，使得领域对象的获取和管理更加方便。
3. 抽象接口：`Repository` 定义了一个与持久化机制无关的接口，这使得领域层的代码可以在不同的持久化机制之间切换，而不需要修改业务逻辑。
用途：
1. 数据访问抽象：`Repository` 为领域层提供了一个清晰的数据访问接口，使得领域对象可以专注于业务逻辑的实现，而不是数据访问的细节。
2. 领域对象的查询和管理：`Repository` 使得对领域对象的查询和管理变得更加方便和灵活，支持复杂的查询逻辑。
3. 领域逻辑与数据存储分离：通过 `Repository` 模式，领域逻辑与数据存储逻辑分离，提高了领域模型的纯粹性和可测试性。
4. 优化数据访问：`Repository` 实现可以包含数据访问的优化策略，如缓存、批处理操作等，以提高应用程序的性能。
实现手段：
1. 定义 `Repository` 接口：在领域层定义一个或多个Repository接口，这些接口声明了所需的数据访问方法。
2. 实现 `Repository` 接口：在基础设施层或数据访问层实现这些接口，具体实现可能是使用ORM（对象关系映射）框架，如MyBatis、Hibernate等，或者直接使用数据库访问API，如JDBC等。
3. 依赖注入：在应用程序中使用依赖注入（DI）来将具体的 `Repository` 实现注入到需要它们的领域服务或应用服务中。这样做可以进一步解耦领域层和数据访问层，同时也便于单元测试。
4. 使用规范模式（Specification Pattern）：有时候，为了构建复杂的查询，可以结合使用规范模式，这是一种允许将业务规则封装为单独的业务逻辑单元的模式，这些单元可以被Repository用来构建查询。

Repository 模式是DDD（领域驱动设计）中的一个核心概念，它有助于保持领域模型的聚焦和清晰，同时提供了灵活、可测试和可维护的数据访问策略。
仓储解耦的手段使用了依赖倒置的设计，所有领域需要的外部服务，不在直接引入外部的服务，而是通过定义接口的方式，让基础设施层实现领域层接口（仓储/适配器）的方式来处理。
那么也就是基础设置层负责原则对接数据库、缓存、配置中心、RPC接口、HTTP接口、MQ推送等各项资源，并承接领域服务的接口调用各项服务为领域层提供数据能力。
同时这也会体现出，领域层的实现是具有业务语义的，而到了基础设置层则没有业务语义，都是原子的方法。通过原子方法的组合为领域业务语义提供支撑。

假设我们有一个 Order 聚合根，我们可以为它创建一个仓储接口：
```java
public interface OrderRepository {
    Order findById(String orderId);
    void save(Order order);
    void delete(Order order);
}
```

然后，我们可以有一个具体的实现，比如使用 JPA：
```java
import org.springframework.stereotype.Repository;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
@Repository
public class JpaOrderRepository implements OrderRepository {
    @PersistenceContext
    private EntityManager entityManager;
    @Override
    public Order findById(String orderId) {
        return entityManager.find(Order.class, orderId);
    }
    @Override
    public void save(Order order) {
        entityManager.persist(order);
    }
    @Override
    public void delete(Order order) {
        entityManager.remove(order);
    }
}
```

假设我们有一个服务需要通过 REST API 暴露 Order 的功能，我们可以创建一个控制器作为输入适配器：
```java
import org.springframework.web.bind.annotation.*;
@RestController
@RequestMapping("/orders")
public class OrderController {
    private final OrderService orderService;
    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }
    @GetMapping("/{orderId}")
    public Order getOrder(@PathVariable String orderId) {
        return orderService.getOrderById(orderId);
    }
    @PostMapping
    public void createOrder(@RequestBody Order order) {
        orderService.createOrder(order);
    }
}
```

也可以参考 [[🐏 Entity、DomainService、Repository 案例#^756c07]]
