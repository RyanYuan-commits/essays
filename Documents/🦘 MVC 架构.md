>不管是 MVC 还是 DDD，都可以归类为软件架构模式，当我们需要构建大型的应用的时候，为了代码的可维护性和可拓展性，我们需要一种结构化的方法来组织代码和组件，我们可以将软件的架构理解为一个超级大的设计模式，在设计模式中我们关注的是如何更优雅的来组织我们的代码，而在架构模式中，我们更关注的是如何更好的来组织我们的项目。
# MVC
![[MVC]]
MVC（Model-View-Controller），是一种目前运用最广泛的一种设计模式，它可以将大型的应用程序，分割为一些特定的部分，每个部分都有自己的职能和目的。
它的名称就直接解释了它的架构层次，分别为：**模型**、**视图** 和 **控制器**。

模型就是我们整个程序的核心功能与数据逻辑，它通常与数据库交互，并包含应用程序的核心功能。
视图负责呈现数据。视图是用户界面部分，显示模型中的数据，并将用户的输入传递给控制器。
控制器则负责响应用户输入并调用模型和视图来执行相应的操作。控制器接收用户输入，处理用户请求，并决定调用哪个模型和视图来实现用户请求。
# 架构案例代码
源码地址：[https://gitcode.net/KnowledgePlanet/road-map/xfg-frame-mvc](https://gitcode.net/KnowledgePlanet/road-map/xfg-frame-mvc)
代码结构
```
.
├── docs
│   └── mvc.drawio - 架构文档
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── cn
│   │   │       └── bugstack
│   │   │           └── xfg
│   │   │               └── frame
│   │   │                   ├── Application.java
│   │   │                   ├── common
│   │   │                   │   ├── Constants.java
│   │   │                   │   └── Result.java
│   │   │                   ├── controller
│   │   │                   │   └── UserController.java
│   │   │                   ├── dao
│   │   │                   │   └── IUserDao.java
│   │   │                   ├── domain
│   │   │                   │   ├── po
│   │   │                   │   │   └── User.java
│   │   │                   │   ├── req
│   │   │                   │   │   └── UserReq.java
│   │   │                   │   ├── res
│   │   │                   │   │   └── UserRes.java
│   │   │                   │   └── vo
│   │   │                   │       └── UserInfo.java
│   │   │                   └── service
│   │   │                       ├── IUserService.java
│   │   │                       └── impl
│   │   │                           └── UserServiceImpl.java
│   │   └── resources
│   │       ├── application.yml
│   │       └── mybatis
│   │           ├── config
│   │           │   └── mybatis-config.xml
│   │           └── mapper
│   │               └── User_Mapper.xml
│   └── test
│       └── java
│           └── cn
│               └── bugstack
│                   └── xfg
│                       └── frame
│                           └── test
│                               └── ApiTest.java
└── road-map.sql
```
整个工程由 SpringBoot 驱动。
- Application.java 是启动程序的 SpringBoot 应用
- common 是额外添加的一个层，用于定义通用的类
- controller 控制层，提供接口实现。
- dao 数据库操作层
- domain 对象定义层
- service 服务实现层