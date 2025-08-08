### 上一部分的原始代码
```Java
public class RegistrationServiceImpl implements RegistrationService {
    private SalesRepMapper salesRepDAO;
    private UserMapper userDAO;
    public UserDO register(String name, PhoneNumber phone) {
        // 获取用户信息
        SalesRepDO repDO = salesRepDAO.select(phone.getAreaCode(), phone.getOperatorCode());
        // 存储用户信息
        UserDO userDO = new UserDO();
        userDO.name = name;
        userDO.phone = phone;
        if (repDO != null) {
            userDO.repId = repDO.repId;
        }
        return userDAO.insert(userDO);
    }
}
```
### 迭代需求
```Java
1. 上期留下的问题，需要对手机号进行实名校验，实名信息通过调用外部服务获得。（假设目前由中国电信提供该服务）
2. 根据外部服务返回的实名信息，按照一定逻辑计算出用户标签，记录在用户账号中。
3. 根据用户标签为该用户开通相应等级的新客福利。
```
最终，经过需求梳理，可以得到这样的流程图：
![[DDD 架构案例流程图.png|600]]

### 平平无奇的写法

```java
public class RegistrationServiceImpl implements RegistrationService {

    private SalesRepMapper salesRepDAO;
    private UserMapper userDAO;
    private RewardMapper rewardDAO;
    private TelecomRealnameService telecomService;
    private RiskControlService riskControlService;

    public UserDO register(String name, PhoneNumber phone) {
        // 参数合法性校验已在PhoneNumber中处理
        // 参数一致性校验
        TelecomInfoDTO rnInfoDTO = telecomService.getRealnameInfo(phone.getNumber());
        if (!name.equals(rnInfoDTO.getName())) {
            throw new InvalidRealnameException();
        }
       
        // 计算用户标签
        String label = getLabel(rnInfoDTO);
        // 计算销售组
        String salesRepId = getSalesRepId(phone);
        
        // 构造User对象和Reward对象
        String idCard = rnInfoDTO.getIdCard();
        UserDO userDO = new UserDO(idCard, name, phone.getNumber(), label, salesRepId);
        RewardDO rewardDO = RewardDO(idCard, label);
        
        // 检查风控
        if(!riskControlService.check(idCard, label)) {
            userDO.setNew(true);
            rewardDO.setAvailable(false);
        } else {
            userDO.setNew(false);
            rewardDO.setAvailable(true);
        }
        
        // 存储信息
        rewardDAO.insert(rewardDO);
        return userDAO.insert(userDO);
    }
    
    private String getLabel(TelecomInfoDTO dto) {
        // 本地逻辑处理
    }
    
    private String getSalesRepId(PhoneNumber phone) {
        SalesRepDO repDO = salesRepDAO.select(phone.getAreaCode(), phone.getOperatorCode());
        if (repDO != null) {
            return repDO.getRepId();
        }
        return null;
    }
}
```

^ef9952

#### 代码存在的外部依赖问题

上述的代码至少存在这样一些依赖问题：

>[!warning] （1）对数据库 schema 的依赖
> 也就是对表的结构具有很强的依赖性，上述代码中大量的使用了 UserDO 对象，甚至返回值也是 UserDO 对象，当数据库的表结构进行了改动，势必会引起 UserDO 对象的改动，可能导致代码被改的面目全非。

>[!warning] （2）对数据库框架的依赖
>上述的代码中使用了 ORM 框架作为查询数据库的工具，当后续对数据库框架进行改动的时候，会引起较大的变动。

>[!warning] （3）RPC 的依赖
>这里使用了电信的手机号查询服务（TelecomRealnameService），当电信在后续的更新中将接口的入参修改，或者我们的服务需要更换运营商的时候，对代码进行改动也会非常麻烦。

#### 处理 RPC 的依赖
也就是处理这一部分的代码：
```java
public class RegistrationServiceImpl implements RegistrationService {
    private TelecomRealnameService telecomService;
    public UserDO register(String name, PhoneNumber phone) {
        TelecomInfoDTO rnInfoDTO = telecomService.getRealnameInfo(phone.getNumber());
        if (!name.equals(rnInfoDTO.getName())) {
            throw new InvalidRealnameException();
        }
        // 构造User对象和Reward对象
        String idCard = rnInfoDTO.getIdCard();
    }
}
```
我们可以利用一个 `RealnameService` 将上面的代码将 `TelecomRealnameService` 隔离开来，并且，将返回的数据从 `TelecomInfoDTO` 改为 `RealnameInfo`；这样，就实现了与电信服务的解耦。
```Java
public interface RealnameService {
    RealnameInfo get(PhoneNumber phone);
}
public class TelecomRealnameService implements RealnameService {
    @Override
    public RealnameInfo get(PhoneNumber phone){
        // RPC调用，并将返回结果封装为RealnameInfo
        // RealnameInfo是DP
    }
}
```
#### 处理数据访问的依赖

^756c07

>[!info]
>这里有两个部分需要修改，分别是 UserDo 作为数据库的直接映射对象，不应该暴露给业务代码去使用；还有对 ORM 对外部依赖问题。

>[!note] User.java
```Java
// User Entity
public class User {
    // 用户id，DP
    private UserId userId;
    // 用户手机号，DP
    private PhoneNumber phone;
    // 用户标签，DP
    private Label label;
    // 绑定销售组ID，DP
    private SalesRepId salesRepId;
    private Boolean fresh = false;
    private SalesRepRepository salesRepRepository;
    // 构造方法
    public User(RealnameInfo info, name, PhoneNumber phone) {
        // 参数一致性校验，若校验失败，则check内抛出异常（DP的优点）
        info.check(name);
        initId(info);
        labelledAs(info);
        // 查询
        SalesRep salesRep = salesRepRepository.find(phone);
        this.salesRepId = salesRep.getRepId();
    }
    // 对this.userId赋值
    private void initId(RealnameInfo info) {}
    // 对this.label赋值
    private void labelledAs(RealnameInfo info) {
        // 本地处理逻辑
    }
    public void fresh() {
        this.fresh = true;
    }
}
```

>[!note] UserRepository.java、UserRepositoryImpl.java
```Java
public interface UserRepository {
    User find(UserId id);
    User find(PhoneNumber phone);
    User save(User user);
}
public class UserRepositoryImpl implements UserRepository {
    private UserMapper userDAO;
    private UserBuilder userBuilder;
    @Override
    public User find(UserId id) {
        UserDO userDO = userDAO.selectById(id.value());
        return userBuilder.parseUser(userDO);
    }
    @Override
    public User find(PhoneNumber phone) {
        UserDO userDO = userDAO.selectByPhone(phone.getNumber());
        return userBuilder.parseUser(userDO);
    }
    @Override
    public User save(User user) {
        UserDO userDO = userBuilder.fromUser(user);
        if (userDO.getId() == null) {
            userDAO.insert(userDO);
        } else {
            userDAO.update(userDO);
        }
        return userBuilder.parseUser(userDO);
    }
}
```

>[!note] RegistrationServiceImpl.java
```Java
public class RegistrationServiceImpl implements RegistrationService {
    private UserRepository userRepository;
    private RewardRepository rewardRepository;
    private RealnameService realnameService;
    private RiskControlService riskControlService;
    public UserDO register(String name, PhoneNumber phone) {
        // 查询实名信息
        RealnameInfo realnameInfo = realnameService.get(phone);
        // 构造对象
        User user = new User(realnameInfo, phone);
        Reward reward = Reward(user);
        // 检查风控
        if(!riskControlService.check(user)) {
            user.fresh();
            reward.inavailable();
        }
        // 存储信息
        rewardRepository.save(reward);
        return UserRepository.save(user);
    }
}
```
上面的代码中，发奖的逻辑和注册逻辑还是耦合在了一起，我们可以将发奖的行为，理解为对用户的一拓展和更新操作；
这样，我们将可以将上面的注册逻辑抽象为这样的三个步骤：
- 获取用户信息
- 对用户进行检查和更新
- 存储用户数据

综合改造后：
>[!note] RegistrationServiceImpl.java
```Java
public class RegistrationServiceImpl implements RegistrationService {
    private UserRepository userRepository;
    private RealnameService realnameService;
    private CheckUserService checkUserService;
    public UserDO register(String name, PhoneNumber phone) {
        // 查询信息
        RealnameInfo realnameInfo = realnameService.get(phone);
        // 构造对象
        User user = new User(realnameInfo, phone);
        // 检查并更新对象
        checkUserService.check(user);
        // 存储信息
        return userRepository.save(user);
    }
}
```

>[!note] CheckUserService.java、CheckAndUpdateUserServiceImpl.java
> 这种涉及到多个 Domain 的修改操作的 Service 被称为 Domain Service，主要用于封装多 Entity 或者跨业务域的逻辑。
```Java
public interface CheckUserService(User user) {
    void check(user);
}
public class CheckAndUpdateUserServiceImpl(User user) {
    private RiskControlService riskControlService;
    private RewardRepository rewardRepository;
    
    @Override
    public void check(User user) {
        rewardCheck(user);
        // ...
        // 可能存在的其他逻辑
    }
    private void rewardCheck(User user) {
        Reward reward = Reward(user);
        // 检查风控
        if(!riskControlService.check(user)) {
            user.fresh();
            reward.inavailable();
        }
        rewardRepository.save(reward);
    }
}
```
### 要点总结
`DP`：抽象并封装自检和一些隐性属性的计算逻辑，且这些属性是无状态的。
`Entity`：抽象并封装单对象有状态的逻辑
`Domain Service`：抽象并封装多对象的有状态的逻辑
`Repository`：抽象并封装外部对象的访问逻辑
### 尝试
我们使用 DDD 来对自己的项目进行组织的时候，一般是采用这样的步骤：
1. 首先需要对需要处理对业务进行一个总览
2. 对领域对象（Domain）进行一个划分，明确每个领域对象包含的信息和职责边界，并进行跨对象、多对象的逻辑组织（Domain Service）
3. 在上层的应用中根据业务去编排 Entity 和 Domain Service
4. 然后再去做一些下水道工作，去对下层的数据访问、RPC 的调用做一些具体的实现。
由这个步骤，我们来尝试以 DDD 的方式来思考一下上面的需求应该如何思考：

>[!note] （1）首先需要对需要处理对业务进行一个总览，这里再来从头梳理一下需求
>
```java

原始需求：
地推员输入用户的姓名和手机号，系统根据客户的手机号的归属地和所属运行商，分配给响应的销售组，由销售组来进行后续的业务。

拓展点：
1. 我们需要对手机号进行实名校验，实名信息通过调用外部服务获得。（假设目前由中国电信提供该服务）
2. 根据外部服务返回的实名信息，按照一定逻辑计算出用户标签，记录在用户账号中。
3. 根据用户标签为该用户开通相应等级的新客福利。
```
![[DDD 架构案例流程图.png|600]]
>[!node] （2）对领域对象（Domain）进行一个划分，明确每个领域对象包含的信息和职责边界，并进行跨对象、多对象的逻辑组织（Domain Service）
>Domain Primitive 指的是：在特定的领域，拥有精确的定义、可以进行自我验证，拥有行为的对象
在上面的业务中，我们可以拆分出这样一些对象，手机号：`PhoneNumber`、








