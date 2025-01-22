官网文档地址：[https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/](https://redis.io/docs/latest/operate/oss_and_stack/management/security/acl/)
使用版本：Redis@7.4.1
# 什么是 ACL？
ACL（Access Control List），权限控制列表，是 Redis 提供的一种对客户端的权限控制方式，它的主要应用场景有：
- 您希望通过限制对命令和密钥的访问来提高安全性，以便不受信任的客户端没有访问权限，而受信任的客户端仅具有执行所需工作所需的数据库的最低访问级别。例如，某些客户端可能只能执行只读命令。
- 您希望提高操作安全性，以便不允许访问 Redis 的进程或人员因软件错误或手动错误而损坏数据或配置。例如，从 Redis 获取延迟作业的工作程序无法调用 FLUSHALL 命令。
# ACL 入门
通过 ACL LIST 命令，我们来看一下 ACL 大概是个什么东西。
默认情况下，有一个用户定义的用户，称为 _default_。我们可以使用 `ACL LIST` 命令来检查当前活动的 ACL 并验证新启动的、默认配置的 Redis 实例的配置：
```bash
> ACL LIST
1) "user default on nopass ~* &* +@all"
```
输出内容的含义是这样的：
- 「`user default`」：user + 用户名
- 「`on`」：该用户是否被启用，也可能为 `false`
- 「`nopass`」：此用户登录无需密码
- 「`~*`」：~ 后面的 * 意味着允许用户访问所有键。可以指定更具体的键模式来限制访问，比如 ~prefix:* 只允许访问以 prefix: 开头的键。
- 「`&*`」：表示频道模式权限，& 后面的 * 意味着用户可以订阅所有频道（用于 Pub/Sub）。同样，你可以通过指定具体的模式来限制频道访问，比如 &news:* 只允许用户订阅以 news: 开头的频道。
- 「`+@all`」：用户可以调用所有命令
# ACL 常见命令
ACL 提供了一系列的命令去查看、新增、删除用户，以及配置用户的权限，具体来说有这些：
## 查看用户信息相关命令
### ACL GETUSER username：获取特定用户的配置
```bash
ACL GETUSER username # 获取某个用户的详细信息
```
**执行案例**
```bash
> ACL GETUSER default
flags
on
nopass
sanitize-payload
passwords
commands
+@all
keys
~*
channels
&*
selectors
```
• **flags**：
	• **on**：用户启用状态，表示该用户当前处于激活状态，可以登录和执行命令。
	• **nopass**：无需密码，表示不需要密码就可以登录该用户。
• **sanitize-payload**：用于清理用户输入的命令参数，防止数据注入或恶意内容的传递。
• **passwords**：这里为空，表示当前未设置密码（因为启用了 nopass）。
• **commands**：+@all 表示默认用户有执行所有命令的权限，+@all 是 Redis 内置的命令类别（command category），表示允许所有命令。
• **keys**：~* 表示该用户对所有键都具有访问权限。
• **channels**：&* 表示该用户可以访问所有的 Pub/Sub 频道。
• **selectors**：这是新的 ACL 规则，可以在条件匹配基础上进一步细分权限。显示为空表示没有附加选择器配置。
### ACL LIST：获取当前所有用户的配置
```bash
ACL LIST # 展示用户列表，并且显示详细权限配置
```
**执行案例**
```
> ACL LIST
user default on nopass sanitize-payload ~* &* +@all
```
### ACL USERS：只展示用户名
```bash
ACL USERS # 展示用户名列表
```
**执行案例**
```
> ACL USERS
default
```
### ACL WHOAMI：查看当前登录的用户
```
> ACL WHOAMI
default
```
### ACL DRYRUN：测试用户是否有执行命令的权限
```bash
ACL DRYRUN username command [arg [arg...]] #用户是否有执行命令的权限
```
比如：
```
> ACL DRYRUN default keys *
OK
```
## 用户创建与修改
用户的创建与修改都是通过下面这条命令来完成的，关于这个命令的使用方式放到后面去详细讲解。
```bash
ACL SETUSER username [rule[rule...]]
```
## 其他命令
### ACL GENPASS：密码生成
```bash
ACL GENPASS [bits]
```
Redis 中的 ACL 还提供了一个密码生成器，后面是比特数，来指定生成密码的安全性等级（即密码的随机性或强度）。它表示生成的密码的位数（bit count）。
如果不指定 bits，Redis 会默认生成一个 256-bit 长度的密码，这通常足够强。
例如：
```bash
> ACL GENPASS
b9a79edba8a1d049113c7205c60e3e580ca275be5e75a733e08238d901f43799
```
# ACL 用户配置
上面提到，ACL 是通过 `ACL SETUSER [rule [rule...]]` 来配置用户的，这里就来看一下有哪些 `rule`。
## 启动和停用用户
`on` ：可以以此用户身份进行身份验证。
`off`：禁止该用户：无法再对此用户进行身份验证;但是，以前经过身份验证的连接仍将有效。
## 配置用户可使用的命令
 `+<command>`：将命令添加到用户可以调用的命令列表中。如果希望达到更加精细的控制，可以与 `|` 一起使用，以仅允许子命令，例如 `+config|get`。
`-<command>`：将命令删除到用户可以调用的命令列表中，同样可以使用 `|`。
`+@<category>`： 添加该类别中所有要由用户调用的命令，有效类别为 @admin、@set、@sortedset...依此类推，通过调用 [`ACL CAT`](https://redis.io/commands/acl-cat) 命令查看完整列表。其中 `@all` 表示所有命令，包括当前存在于服务器中的命令，以及将来将通过模块加载的命令。
`-@<category>`：从客户端可以调用的命令列表中删除命令组。
`allcommands`：+@all 的别名。请注意，它意味着能够执行通过 modules 系统加载的所有未来命令。
`nocommands`：-@all 的别名。
## 控制用户可访问的键
`~<pattern>`：添加可以作为命令一部分提及的键模式。例如`，~*` 允许所有键。该模式是 glob 样式的模式，类似于 [`KEYS`](https://redis.io/commands/keys) 命令的模式。可以指定多个模式。
`allkeys`：`~*` 的别名。
`resetkeys`：刷新允许的键模式列表。例如，ACL `~foo:* ~bar:* resetkeys ~objects:*` 将仅允许客户端访问与模式 `objects：*` 匹配的键。
## 发布与订阅
- `&<pattern>`：添加用户可访问的 Pub/Sub 渠道的 glob 样式模式。可以指定多个通道模式。请注意，模式匹配仅对 [`PUBLISH`](https://redis.io/commands/publish) 和 [`SUBSCRIBE`](https://redis.io/commands/subscribe) 提到的通道进行，而 [`PSUBSCRIBE`](https://redis.io/commands/psubscribe) 要求其通道模式与用户允许的通道模式之间的文本匹配。PSUBSCRIBE 使用的权限匹配是**精确的**，即 `PSUBSCRIBE` 的订阅模式必须完全匹配到 ACL 配置中允许的频道模式。例如，如果你在 ACL 中配置了 &news.，那么用户可以使用 `PSUBSCRIBE news.sports` 或 `PSUBSCRIBE news.weather`。但是，如果用户试图订阅更通用的模式（例如 `PSUBSCRIBE news.`），Redis 会拒绝订阅请求。这种精确匹配有助于提高安全性，使管理员可以更加精细地控制用户对特定频道的访问权限，避免 PSUBSCRIBE 滥用过于宽泛的模式。
- `allchannels`：`&*` 的别名，允许用户访问所有 Pub/Sub 渠道。
- `resetchannels`：刷新允许的频道模式列表，如果用户的 Pub/Sub 客户端无法再访问各自的频道和/或频道模式，则断开这些客户端的连接。
## 配置用户登录密码
- `><password>`： 将此密码添加到用户的有效密码列表中。例如，`>mypass` 会将 “mypass” 添加到有效密码列表中。此指令清除 _nopass_ 标志）。每个用户都可以拥有**任意数量**的密码。
- `<<password>`： 从有效密码列表中删除此密码。如果您尝试删除的密码实际上未设置，则发出错误。
- `#<hash>`： 将此 SHA-256 哈希值添加到用户的有效密码列表中。此哈希值将与为 ACL 用户输入的密码的哈希值进行比较。这允许用户在 `acl.conf` 文件中存储哈希值，而不是存储明文密码。仅接受 SHA-256 哈希值，因为密码哈希必须为 64 个字符，并且仅包含小写十六进制字符。
- `！<hash>`： 从有效密码列表中删除此哈希值。当您不知道哈希值指定的密码，但想要删除用户的密码时，这非常有用。
- `nopass`：删除用户的所有设置密码，并将用户标记为不需要密码：这意味着每个密码都将对该用户起作用。如果此指令用于 default 用户，则每个新连接将立即使用 default 用户进行身份验证，而无需任何显式的 AUTH 命令。请注意，_resetpass_ 指令将清除此条件。
- `resetpass`：刷新允许的密码列表并删除 _nopass_ 状态。_在 resetpass_ 之后，用户没有关联的密码，如果不添加一些密码（或稍后将其设置为 _nopass_），则无法进行身份验证。
# ACL 配置的持久化
Redis 默认情况下不持久化 ACL 配置。除非显式地进行保存，ACL 配置不会自动写入配置文件或持久化存储。这意味着在 Redis 重启后，所有基于 ACL 的用户和权限设置会恢复为默认配置，除非事先保存。
持久化的第一种方式是保存到 `redis.conf` 文件中，通过 `CONFIG REWRITE` 指令，在 SECURITY 部分可以找到相关的配置：
![[redisconf 持久化 acl.png|600]]
还有一种方式是使用特定的文件来存储 acl 配置，在客户端执行 `ACL SAVE` 能够将当前配置的 ACL 保存到文件中，这个文件也需要在 `redis.conf` 中指定。
![[acl file 位置.png|600]]
需要注意指定的文件 redis 必须有访问的权限，否则会报错，可以通过查看 redis 日志来确定。
通过 `ACL SAVE` 可以将当前的 ACL 配置保存到文件中，而如果我们已经准备好了一个 ACL 文件，同样可以通过 `ACL LOAD` 将配置加载到 Redis 中。
这就是关于持久化最重要的两个命令：`ACL SAVE` 和 `ACL LOAD`