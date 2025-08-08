## 聚合根
### 聚合根的作用
问题提出：在一个复杂的系统中，对一个实体的状态的修改，可能会引起与其关联的其他的实体的状态，如何保证其一致性？
使用聚合来描述这种存在聚合关系的引用集合，目的是：屏蔽掉内部对象之间的复杂关系，只对外暴露统一的接口，类似外观模式。
### 关键概念
关键概念：根对象 与 边界
- 根对象：唯一被外部引用的对象，暴露的接口只允许操纵根对象。
- 边界：判断哪些对象可以被放入当前聚合，比如在执行用户注销场景中，钱包、银行卡就是在聚合内部，而用户的聊天记录不需要删除，不用被归类到聚合。
![[DDD 聚合.png|800]]

聚合外部通过持有根对象的引用来操作聚合，聚合内部通过一系列逻辑来保证内部状态的一致性。
### 案例展示
可以参考下面的内容定义接口和实体等。
```Java
// 作为唯一标识的统一接口
public interface Identifier extends Serializable {}

// 实体类的统一接口
public interface Entity<T extends Identifier> {}

// 实现 Identifier 作为一个唯一标识
public class PhoneNumber implements Identifier {}

// 聚合根, 是一个 Entity
public interface AggregateRoot<T extends Identifier > extends Entity<T> {}

// WechatAccount 是一个聚合根, 它是一个以 PhoneNumber 作为唯一标识的聚合根
public class WechatAccount implements AggregateRoot<PhoneNumber> {}
```
#### 简单的转账需求
```Java
public class Balance implements Entity<BalanceNumber> {
    private BalanceNumber id;
    //...
    private BalanceNumber() {}
}

public class Wallet implements Entity<WalletNumber> {
    private WalletNumber id;
    private Balance balance; // 余额
    private Statement statement； // 账单明细
    //...
    private Wallet() {}
}

public class WechatAccount implements AggregateRoot<PhoneNumber> {
    private PhoneNumber id;
    private Nickname nickname; 
    private Wallet wallet;
    //...
    private WechatAccount() {}
}
```

Domain Service: 封装多个域的处理方法，其属于外部的对象，只能只有转账聚合的聚合根 `WechatAccount`
```Java
// domain service
public class TransferServiceImpl {
    public AccountRepository repository;
    
    @Override
    public void transfer(WechatAccount payer, WechatAccount payee, Asset asset) {
        payer.withdraw(asset);
        payee.deposit(asset);
        repository.save(payer);
        repository.save(payee);
   }
}
```
### 聚合根初体验
```Java
public class WechatAccount implements AggregateRoot<PhoneNumber> {
    private PhoneNumber id;
    private Nickname nickname; 
    private Wallet wallet;
    //...
    private WechatAccount() {}
    
    public void withdraw(Asset asset) {
        wallet.pay(asset);
    }
    
    public void deposit(Asset asset) {
        wallet.receive(asset);
    }
}

public class Wallet implements Entity<WalletNumber> {
    private WalletNumber id;
    private Balance balance; // 余额
    private Statement statement； // 账单明细
    //...
    private Wallet() {}
    
    public void pay throws BalanceException(Asset asset) {
        balance.decrease(asset.getBalance);
        statement.add(asset);
    }
    
    public void receive throws BalanceException(Asset asset) {
        balance.increase(asset.getBalance);
        statement.add(asset);
    }
    
} 
```
### 数据访问
```Java
public interface Repository<T extends AggregateRoot<ID>, ID extends Identifier> {
    save(T t);
    WechatAccount find(ID id);
}
```

```Java
public interface AccountRepository extends Repository<WechatAccount, PhoneNumber> {
    save(WechatAccount wechatAccount);
    WechatAccount find(PhoneNumber phoneNumber);
}
```

```Java
public class WeAccountRepositoryImpl implements AccountRepository {
    private WechatAccountDAO wechatAccountDAO;
    private WalletDAO  walletDAO;
    private BalanceDAO balanceDAO;
    private StatementDAO statementDAO;
    
    @Override
    public save(WechatAccount wechatAccount) {
        WechatAccountPO wechatAccountPO = WechatAccountConverter.convert(wechatAccount);
        wechatAccountDAO.save(wechatAccountPO);
        
        Wallet wallet = WalletFactory.obtain(wechatAccount);
        WalletPO walletPO = walletConverter.convert(wallet);
        walletDAO.save(walletPO);
        
        Balance balance = BalanceFactory.obtain(wallet);
        BalancePO balancePO = balanceConverter.convert(balance);
        balanceDAO.save(balancePO);
        
        Statement statement = StatementFactory.obtain(wallet);
        StatementPO statementPO = statementConverter.convert(statement);
        statementDAO.save(statementPO);
        
        //... 其他WechatAccount的属性
    }
    
    @Override
    WechatAccount find(PhoneNumber phoneNumber) {
        //...
    }
}
```
存在的问题：如果 `WeChatAccount` 中的 `Nickname nickname` 属性被修改，此时按照上面的聚合方式来说，我们是需要调用持久化层将聚合中的所有数据进行同步的，这里的优化空间非常大。
在修改对象的时候，通过代理来保证记录对象是否被修改，如果被修改再进行落库。
## 限界上下文
限界上下文是一个显式的边界，在这个边界内，领域模型的所有概念、规则和操作都是一致的、有明确语义的，并且与其他限界上下文相互隔离。它将一个大型的业务领域划分为多个相对独立的子领域，每个子领域都有自己的上下文边界，在这个边界内进行领域模型的构建和维护。
### 作用
- **划分业务边界**：随着业务的不断发展和复杂，一个系统可能包含多个不同的业务板块，如电商系统中的订单管理、库存管理、用户管理等。限界上下文可以将这些不同的业务板块清晰地划分开来，每个板块都有自己独立的限界上下文，避免不同业务概念和规则之间的混淆和冲突。
- **统一业务语言**：在每个限界上下文内部，团队成员使用统一的业务语言进行交流和协作。比如在订单管理的限界上下文中，大家对于 “订单”“订单状态”“订单金额” 等概念都有明确且一致的定义和理解，减少了因沟通不畅导致的误解和错误。
- **实现领域模型的自治**：每个限界上下文都可以独立地进行领域模型的设计、开发、测试和维护。不同限界上下文之间通过明确的接口进行交互，这样可以降低系统的耦合度，提高系统的可扩展性和可维护性。例如，库存管理限界上下文可以根据自身的业务需求和变化，独立地修改库存更新策略等，而不会影响到其他限界上下文。
- **支持团队协作**：限界上下文为团队的组织和协作提供了清晰的边界。不同的团队可以负责不同的限界上下文，专注于自己所负责的业务领域，提高团队的工作效率和专业性。同时，明确的边界也有助于团队之间的沟通和协作，清楚地知道各自的职责和接口。