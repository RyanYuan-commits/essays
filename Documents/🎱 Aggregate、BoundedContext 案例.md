# 对 DP 和 VO 的回顾
```Java
public interface Identifier extends Serializable {}
public interface Entity<T extends Identifier> {}
public class PhoneNumber implements Identifier {}
public class Red {
    private String desc；
    private String rgb;
    // ...
}
public interface AggregateRoot<T extends Identifier > extends Entity<T>{    }
public class WechatAccount implements AggregateRoot<PhoneNumber> {}
```

# 聚合（Aggregate）
## 简单的转账需求
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

## 聚合根初体验
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
        wallet. receive(asset);
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

## 数据访问
```Java
public interface Repository<T extends AggregateRoot<ID>, ID extends Identifier> {
    save(T t);
    WechatAccount find(ID id);
}
```

```Java
public interface AccountRepository extends Repository<WechatAccount, PhoneNumber> {
    save(WechatAccount wechatAccount);
    WechatAccount find(PhoneNumber phoneNumber)
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

# 边界上下文（Bound Context）
Bounded Context 是一个明确的 **边界**，在这个边界内，特定的领域模型有其独特的含义和用途。它定义了一个应用程序或子系统中某个特定领域的模型及其行为。
在大型系统中，不同的团队可能会对同一个概念有不同的理解和实现。Bounded Context 通过明确的边界来隔离这些不同的模型，确保每个上下文内的模型都是一致的。
帮助团队在复杂的系统中管理和协调不同的领域模型，避免模型之间的冲突和混淆。

在实践中，Bounded Context 通常对应于一个微服务、模块或组件。每个 Bounded Context 都有自己的数据存储、业务逻辑和 API。
不同的 Bounded Context 之间通过明确的接口进行通信，通常使用防腐层（Anti-Corruption Layer）来转换和适配不同上下文之间的数据和命令。
   - 例如，在一个电子商务系统中，"订单管理" 和 "客户管理" 可以是两个不同的 Bounded Context。每个上下文都有自己的模型和逻辑，"订单管理" 可能关注订单的创建、更新和状态，而 "客户管理" 则关注客户的信息和验证。