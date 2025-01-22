## 配置全局配置文件 /etc/profile 应用场景
当前用户进入 Shell 环境初始化的时候，会加载全局配置文件 /etc/profile 里面的环境变量，提供给所有的  Shell 程序使用，以后只要是所有 Shell 程序或者命令使用的变量，就可以定义在这个问价那种。

创建环境变量的步骤：
- 编辑 /ect/profile 全局配置文件
- 重新加载配置文件，因为配置文件修改之后要立即加载里面的数据需要重载 `source /etc/profile`

```sh
vim /etc/profile
# 使用 G 在命令模式可以跳转到末尾，gg 可以跳转到开头
export VAR1=VAR

source /etc/profile
echo $VAR1
```

## 加载流程原理介绍
### Shell 工作环境
Shell 工作环境：用户进入 Linux 系统就会初始化 Shell 环境，这个环境会加载全局配置文件和用户个人配置文件中的环境变量；每个脚本文件都有自己的 Shell 环境。
#### 交互式 Shell 与非交互式 Shell
交互式 Shell：与用户进行交互，用户输入一个命令，Shell 环境立即反馈响应；
非交互式 Shell：不需要用户参与就可以执行多个命令，比如多个脚本文件执行多个命令，直接执行并给出结果。
#### 登录 Shell 环境与非登录 Shell 环境
**登录环境**指的是用户通过某些方式启动 Shell 时，需要经过身份验证登录的情况，例如：
- 通过终端登录（tty 或 ssh）；
- 显式调用 Shell 时加了 -l 参数，例如：bash -l 或 sh -l 。
登录时会加载与用户环境相关的配置文件：
- **全局配置文件**：/etc/profile
- **用户级配置文件**（按顺序加载，存在即执行）：
	1. ~/.bash_profile
	2. ~/.bash_login
	3. ~/.profile

![[Shell 环境变量加载流程.png]]

Shell  登录环江执行脚本文件语法：
```bash
sh/bash -l/--login 脚本文件
```
先加载 Shell 登录环境流程初始化环境变量，再执行脚本文件

Shell 非登录环境执行脚本文件语法 ：
```sh
bash # 加载 Shell 非登录环境
sh/bash 脚本文件
```
### Shell 环境类型识别
使用 \$0 获取 Shell 环境
```bash
echo $0
```
> 输出 -bash 就是登录环境；
> 输出 bash 是非登录环境

测试案例：
![[Shell 环境类型识别.png]]
### SHELL 环境切换方式
1. 直接远程登录到虚拟机，就是登录环境；
2. 使用 su 切换用户登录
```bash
su 用户名 --login
su 用户名 -l
```
> 切换到指定用户并加载登录环境环境变量
```bash
su 用户名
```
>切换到指定用户加载非登录环境变量
3. 使用 Bash 直接切换
