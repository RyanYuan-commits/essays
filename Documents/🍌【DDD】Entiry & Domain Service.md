上一篇 [[💫【DDD】DP]]

上期留下的问题
```
假设现在在做一个简单的数据统计系统，市场专员输入客户的姓名和手机号，根据客户手机号的归属地和所属运营商，将客户群体分组，分配给相应销售专员，由销售专员跟进后续的业务。
```

## 需求梳理
```
业务方法的入参是客户姓名和手机号，首先使用手机号去调用外部服务查询实名信息，校对是否和入参中的姓名一致，如果一致，则通过。

然后，然后根据得到的实名信息，按照一定逻辑计算得出该用户的标签，该标签将作为用户的一个属性。

接着，根据手机号的归属地和所属运营商，查询得到关联的销售组信息，该销售组ID将作为用户的一个属性。

此时，根据用户信息，构建用户对象和福利对象，并查询风控是否通过。

若通过，用户失去新客身份，且可以查询到福利信息，数据落库。
若不通过，用户保持新客身份，但查询不到福利信息，数据落库。

上述逻辑默认在同一个事务中处理。
```

## 平平无奇的写法
>[!question] 怎样定义外部依赖?
> 一切不属于当前域中的设施或者服务，比如数据库、数据库 Schema、RPC 等等

最终目的: 将外部依赖变动，对代码变动的影响降到最低。
```Java
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
        }else {
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
上面的代码存在的外部依赖：
- 对 UserDO 的依赖
- 对 MyBatis 的依赖
- 对电信提供的 RPC SDK 的依赖
怎么办？面向具体实现编程 => 面向**抽象接口**编程
## 改动方法
### RPC 调用抽象改造
```java
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

```java
public class RegistrationServiceImpl implements RegistrationService {

    private SalesRepMapper salesRepDAO;
    private UserMapper userDAO;
    private RewardMapper rewardDAO;
    private RealnameService realnameService;
    private RiskControlService riskControlService;

    public UserDO register(String name, PhoneNumber phone) {
        // 一致性校验
        RealnameInfo realnameInfo = realnameService.get(phone);
        realnameInfo.check(name);
        
        // ......
}
```
### 数据访问抽象改造
- DO 直接映射数据表 Schema，不应该直接暴露给业务逻辑。
- DAO 作为访问中的具体实现，也不应该直接暴露给外部逻辑。
最终目标：业务代码 面向 领域实体

```java
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
    private void initId(RealnameInfo info) {
    
    }
    
    // 对this.label赋值
    private void labelledAs(RealnameInfo info) {
        // 本地处理逻辑
    }
    
    public void fresh() {
        this.fresh = true;
    }
}
```

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
### 最终改造
回归注册最基本的逻辑：构造用户 + 检查并更新用户 + 存储信息
上面的业务中，抽奖逻辑为什么要在注册功能中呢？
检查并更新用户 中 存在发奖的衍生行为，将这部分需要连接多个域的功能拆分出来，构筑新的 Service。
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

---

总结：
我们的代码中会存在哪些问题？

对于外部依赖的耦合性过高，比如对于 Do（强依赖与数据库 Schema）、对于 Mapper（强依赖于持久层框架）、对于 RPC 服务（强依赖于框架或外部 SDK）等等。
=> 构筑防腐层，本质就是用接口来隔离功能提供方和功能使用方的变化，两边都只需要对接口负责。

代码中会耦合其他无关的业务，比如说，注册功能中，耦合了对于新用户发奖等功能。
=> 将这部分逻辑抽象成注册功能中的一个模块，比如抽奖功能可以抽象为检查用户状态，将这部分单独拆分出来，构建一个 Domain Service，在这个复杂 check 的 Domain Service 去书写与注册无关的逻辑。


