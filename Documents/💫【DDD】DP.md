## 平平无奇的案例
>[!case] 比如现在有一个这样的简单需求，地推员输入用户的姓名和手机号，系统根据客户的手机号的归属地和所属运行商，分配给响应的销售组，由销售组来进行后续的业务。

```Java
public class User {
    Long userId;
    String name;
    String phone;
    Long repId;
}
public class RegistrationServiceImpl implements RegistrationService {
    private SalesRepRepository salesRepRepo;
    private UserRepository userRepo;
    
    public User register(String name, String phone)
            throws ValidationException {
        // 参数校验
        if (name == null || name.length() == 0) {
            throw new ValidationException("name");
        }
        if (phone == null || !isValidPhoneNumber(phone)) {
            throw new ValidationException("phone");
        }
        // 获取手机号归属地编号和运营商编号 然后通过编号找到区域内的SalesRep
        String areaCode = getAreaCode(phone);
        String operatorCode = getOperatorCode(phone);
        SalesRep rep = salesRepRepo.findRep(areaCode, operatorCode);
        // 最后创建用户，落盘，然后返回
        User user = new User();
        user.name = name;
        user.phone = phone;
        if (rep != null) {
            user.repId = rep.repId;
        }
        return userRepo.save(user);
    }
    private boolean isValidPhoneNumber(String phone) {
        String pattern = "^0[1-9]{2,3}-?\\d{8}$";
        return phone.matches(pattern);
    }
    private String getAreaCode(String phone) {
        //...
    }
    private String getOperatorCode(String phone) {
       //...
    }
}
```
上面代码的流程是这样的：
1. 首先对入参进行参数校验
2. 然后获取手机号的归属地编号和运营商编号，找到对应的销售组
3. 构建用户对象，并且将上面获取到的用户组 id 存入用户的相应字段
4. 存储用户数据

==上面的代码存在哪些问题呢？==
1. 方法的入参是两个 String 类型，在方法被编译引用后，我们并不能通过方法的参数类型来确定我们的参数顺序，这就导致了接口语义的不明确，或者有可能因为开发者的纰漏导致了入参的填写错误。
2. 当出现后续需要拓展的情况，比如我们希望拓展通过身份证号来构造用户，就需要修改接口来适配这个拓展，从而继续引发上面的问题。
3. 当我们的接口被大量方法实现的时候，如果每个方法开头都需要对这样的含有特殊格式的数据
	- 也就意味着在每个方法之前都需要进行一次检测，势必会造成大量重复的代码
	- 而如果在类中封装单独的方法，也会对后续的维护产生一些影响
		- 如果封装成单独的工具类，业务方法调用工具类，如果参数出现出现问题之后，最简单的处理方法就是抛出异常，这样就会导致在我们的业务代码之中，既要处理业务异常，而又要处理参数异常。
		- 一旦参数越来越多，后续工具类也需要不断修改，也不利于后续的维护。

综上所述，我们希望我们的接口可以
1. 接口的语意明确、拓展性强、最好还带有自检性
2. 参数校验的逻辑可以实现服用
3. 参数教研异常和业务异常可以解耦
## 案例优化 
避免接口语义混乱，封装基本的校验逻辑逻辑
>[!info] PhoneNumber.java
```Java
public class PhoneNumber {
    private final String number;
    private final String pattern = "^0?[1-9]{2,3}-?\\d{8}$";
    
    public String getNumber() {
        return number;
    }
    // 仅存在含参构造器
    public PhoneNumber(String number) {
        if (number == null) {
            throw new ValidationException("number不能为空");
        } else if (isValid(number)) {
            throw new ValidationException("number格式错误");
        }
        this.number = number;
    }
    private boolean isValid(String number) {
        return number.matches(pattern);
    }
    public String getAreaCode() {
        //...
    }
    public String getOperatorCode(PhoneNumber phone) {
       //...
    }
}
```

>[!info] User.java、RegistrationServiceImpl.java
```Java
public class User {
    Long userId;
    String name;
    PhoneNumber phone;
    Long repId;
}

public class RegistrationServiceImpl implements RegistrationService {
    private SalesRepRepository salesRepRepo;
    private UserRepository userRepo;
    public User register(String name, PhoneNumber phone) {
        // 获取手机号归属地编号和运营商编号，然后通过编号找到区域内的SalesRep
        String areaCode = getAreaCode(phone);
        String operatorCode = getOperatorCode(phone);
        SalesRep rep = salesRepRepo.findRep(areaCode, operatorCode);
        // 最后创建用户，落盘，然后返回
        User user = new User();
        user.name = name;
        user.phone = phone;
        if (rep != null) {
            user.repId = rep.repId;
        }
        return userRepo.save(user);
    }
    private String getAreaCode(PhoneNumber phone) {
        //...
    }
    private String getOperatorCode(PhoneNumber phone) {
       //...
    }
}
```
这样就解决了上面的问题，而我们观察这段代码：
```java
// 获取手机号归属地编号和运营商编号，然后通过编号找到区域内的SalesRep
String areaCode = getAreaCode(phone);
String operatorCode = getOperatorCode(phone);
SalesRep rep = salesRepRepo.findRep(areaCode, operatorCode);
```
这个逻辑是否应该存放在 **注册** 功能中呢？
我们可以修改 `findRep()` 接口方法的入参：
```java
public SalesRep findRep(PhoneNumber phone);
```
这样就是比较合理的；而如果我们无法修改这个入参；也可以将获取手机归属地和运营商编号的方法放在这个手机号类之中，这样也是合理的；经过这样的改造，我们最终将会获得一个非常纯粹的注册方法，也就是获取用户的信息，然后将用户的数据存储到数据库。
```Java
public class PhoneNumber {

    private final String number;
    private final String pattern = "^0?[1-9]{2,3}-?\\d{8}$";
    
    public String getNumber() {
        return number;
    }

    // 仅存在含参构造器
    public PhoneNumber(String number) {
        if (number == null) {
            throw new ValidationException("number不能为空");
        } else if (isValid(number)) {
            throw new ValidationException("number格式错误");
        }
        this.number = number;
    }

    private boolean isValid(String number) {
        return number.matches(pattern);
    }
    
    public String getAreaCode() {
        //...
    }
    
    public String getOperatorCode(PhoneNumber phone) {
       //...
    }
}
```

```Java
public class User {
    Long userId;
    String name;
    PhoneNumber phone;
    Long repId;
}

public class RegistrationServiceImpl implements RegistrationService {

    private SalesRepRepository salesRepRepo;
    private UserRepository userRepo;

    public User register(String name, PhoneNumber phone) {
    
        // 获取用户信息
        SalesRep rep = salesRepRepo.findRep(phone.getAreaCode(), phone.getOperatorCode());

        // 存储用户信息
        User user = new User();
        user.name = name;
        user.phone = phone;

        if (rep != null) {
            user.repId = rep.repId;
        }

        return userRepo.save(user);
    }
}
```

我们可以将上面这个 PhoneNumber 类称为一个 DP（Domain Primitive），正如 String、Integer 是语言的基础，DP 是 DDD 架构中一切模型、方法和架构的基础；它是在特定的领域，拥有精确的定义、可以进行自我验证，拥有行为的对象，可以认为是领域的最小组成部分。

>[!question] 思考
> 目前 PhoneNumber 中，封装的是一些基础的字符串校验逻辑，没有外部的依赖；
> 但是如果需要对手机号的注册信息和用户名做一致性的校验，让 PhoneNumber 内部依赖查询服务显然是不合适的，应该怎么办呢？
