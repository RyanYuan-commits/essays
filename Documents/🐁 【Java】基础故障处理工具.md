## 梗概
在 JDK 的 bin 目录下，我们运行 `tree`  命令，可以看到这些命令（不同的版本可能会有所变化）：
```
ls
appletviewer	javadoc		jfr		jsadebugd	pack200		servertool
extcheck	javah		jhat		jstack		policytool	tnameserv
idlj		javap		jinfo		jstat		rmic		unpack200
jar		jcmd		jjs		jstatd		rmid		wsgen
jarsigner	jconsole	jmap		keytool		rmiregistry	wsimport
java		jdb		jps		native2ascii	schemagen	xjc
javac		jdeps		jrunscript	orbd		serialver
```
除了常用的 `java`、`javac` 命令工具以外，还有一些其他的小的工具，随着JDK版本的更迭，这些小工具的数量和功能也在不知不觉地增加与增强。除了编译和运行 Java 程序外，打包、部署、签名、调试、监控、运维等各种场景都可能会用到它们，
## jps：虚拟机进程状况工具
jps（JVM Process Status Tool），名字非常像 Unix 命令的 ps，实际上，它的功能也确实是这样：
- 列出正在运行的虚拟机进程，并显示虚拟机执行主类（Main Class，main()函数所在的类）名称以及这些进程的本地虚拟机唯一ID（LVMID，Local Virtual Machine Identifier）。
正如 Docker 运行其他命令需要容器的 ID，我们使用 JDK 提供的其他工具，为了锁定运行的是哪个 Java 程序，业务要这样一个 ID，因此，这个命令的功能虽然比较单一，却是使用最频繁的命令之一。

**jps 命令格式**
```java
jps [OPTIONS] [HOSTID]
```
运行示例：
```bash
ryan@192 bin % jps -l
10017 com.intellij.idea.Main
32089 jdk.jcmd/sun.tools.jps.Jps
```
jps还可以通过RMI协议查询开启了RMI服务的远程虚拟机进程状态，参数hostid为RMI注册表中注册的主机名。

| 选项  | 作用                         |
| --- | -------------------------- |
| -q  | 只输出 LVMID，省略主类的名称          |
| -m  | 输出虚拟机启动的时候传给主类 main 函数的参数  |
| -l  | 输出主类的全名，如果是 JAR，则输出 JAR 路径 |
| -v  | 输出启动时的虚拟机参数                |
👆其他 OPTIONS
## jstat：虚拟机统计信息监视工具
jstat（JVM Statistics Monitoring Tool）是用于监视虚拟机各种运行状态信息的命令行工具。它可以显示本地或者远程虚拟机进程中的类加载、内存、垃圾收集、即时编译等运行时数据，在没有GUI图形界面、只提供了纯文本控制台环境的服务器上，它将是运行期定位虚拟机性能问题的常用工具。
jstat 命令格式：
```java
jstat [ OPTIONS ] [ VMID ] [ INTERVAL(s|ms) ] [ COUNT ]
```
其中的 间隔（INTERVAL）、次数（COUNT）都是可选参数，如果省略这两个参数，代表的含义就是只查询一次。
假设需要每250毫秒查询一次进程2764垃圾收集状况，一共查询20次，那命令应当是：
```java
jstat -gc 2764 250 20
```
默认的 INTERVAL 是 ms，如果要执行 s 的命令，格式为：
```java
jstat -gc 2764 1s 20
```

|选项|作用|
|----|----|
|-class|监视类加载、卸载数量、总空间以及类装载所耗费的时间|
|-gc|监视 Java 堆状况，包括 Eden 区、2 个 Survivor 区、老年代、永久代等的容量，已用空间，垃圾收集时间合计等信息|
|-gccapacity|监视内容与 -gc 基本相同，但输出主要关注 Java 堆各个区域使用到的最大、最小空间|
|-gcutil|监视内容与 -gc 基本相同，但输出主要关注已使用空间占总空间的百分比|
|-gccause|与 -gcutil 功能一样，但是会额外输出导致上一次垃圾收集产生的原因|
|-gcnew|监视新生代垃圾收集状况|
|-gcnewcapacity|监视内容与 -gcnew 基本相同，输出主要关注使用到的最大、最小空间|
|-gcold|监视老年代垃圾收集状况|
|-gcoldcapacity|监视内容与 -gcold 基本相同，输出主要关注使用到的最大、最小空间|
|-gcpermcapacity|输出永久代使用到的最大、最小空间|
|-compiler|输出即时编译器编译过的方法、耗时等信息|
|-printcompilation|输出已经被即时编译的方法|
## jinfo：Java 配置信息工具
jinfo（Configuration Info for Java）的作用是实时查看和调整虚拟机各项参数。使用 jps 命令的 -v 参数可以查看虚拟机启动时 **显式指定** 的参数列表；
但如果想知道未被显式指定的参数的系统默认值，除了去找资料外，就只能使用jinfo的-flag选项进行查询了（如果只限于JDK 6或以上版本的话，使用 `java-XX：+PrintFlagsFinal` 查看参数默认值也是一个很好的选择）。
`jinfo` 还可以使用 `-sysprops` 选项把虚拟机进程的 `System.getProperties()` 的内容打印出来。
这个命令在 JDK 5 时期已经随着Linux版的JDK发布，当时只提供了信息查询的功能，JDK 6 之后，jinfo 在 Windows 和 Linux 台都有提供，并且加入了在运行期修改部分参数值的能力（可以使用 -flagname 或者 -flag name=value 在运行期修改一部分运行期可写的虚拟机参数值）。在 JDK 6 中，`jinfo` 对于 `Windows` 平台功能仍然有较大限制，只提供了最基本的 `-flag` 选项。

**jinfo 命令格式**
```java
jinfo [OPTIONS] PID
```
执行样例：查询 `CMSInitiatingOccupancyFraction` 参数值

| 选项                     | 描述                                                                         |
| ---------------------- | -------------------------------------------------------------------------- |
| `-flag <name>`         | 显示指定的[JVM](https://docs.oracle.com/javase/tutorial/deployment/jar/)选项的当前值。 |
| `-flag [+-]<name>`     | 启用或禁用布尔型JVM选项。                                                             |
| `-flag <name>=<value>` | 将指定的JVM选项设置为给定的值。                                                          |
| `-flags`               | 显示所有JVM选项及其当前值。                                                            |
| `-sysprops`            | 显示系统属性。                                                                    |


