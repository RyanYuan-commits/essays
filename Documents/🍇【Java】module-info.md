在 Java 的模块化中, 一个模块的定义文件需要命名为 `module-info.java`, 这个文件要放在模块的根目录中.

文件中使用一系列的关键字来定义模块的属性和依赖关系.

一个 `module-info` 的简单模板:

```java
module com.example.myawesomeapp {

    // 1. 声明依赖

    // 依赖于其他模块

    requires com.example.database;

    requires java.sql;

    requires transitive com.example.core; // 传递性依赖

  

    // 2. 暴露包

    // 将模块中的某个包对外公开，其他模块才能访问

    exports com.example.app.service;

    exports com.example.app.api to com.example.client; // 限定性导出

  

    // 3. 开放包（用于反射）

    // 将模块中的某个包开放给反射访问（即使不对外暴露）

    opens com.example.app.model;

    opens com.example.app.util to com.example.test; // 限定性开放

  

    // 4. 服务提供

    // 声明本模块提供了一个服务接口的实现

    provides com.example.app.service.MyService with com.example.app.service.MyServiceImpl;

  

    // 5. 服务消费

    // 声明本模块需要使用一个服务接口

    uses com.example.app.service.MyService;

}
```
