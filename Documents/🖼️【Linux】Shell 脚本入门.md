## 基本理解
通过编写 Shell 命令，发送给 Linux 内核去执行，操作的就是计算机硬件了，Shell 命令是用户操作内核的桥梁。
Shell 是命令，类似于 Windows 系统 Dos 命令；
Shell 也是一门程序设计语言，里面还有变量、函数、逻辑控制语句等等；

Shell 脚本通过 Shell 命令或者编程语言编写的 Shell 文本文件，这就是 Shell 脚本，也叫 Shell 程序。

为什么要学习 Shell？
通过 Shell 命令与编程语言来提高 Linux 系统对管理工作效率

Shell 的运行过程 用户 => Shell 解析器 => 内核服务

解析器的基本类型，通过 cat /etc/shells 命令查看 Linux 提供的 Shell 解析器：
```bash
$ cat shells
	/bin/sh
	/bin/bash
	/usr/bin/sh
	/usr/bin/bash
```

通过 `echo $SHELL` 打印输出当前系统使用的 SHELL 解析器类型
> echo 用于打印输出数据到终端
> `$SHELL` 是全局共享的环境变量，所有 Shell 程序都可以读取的变量

| 解析器类型         | 介绍                                                                                          |
| ------------- | ------------------------------------------------------------------------------------------- |
| /bin/sh       | Bourne Shell，是 Linux 最初使用的 Shell                                                            |
| /bin/bash     | Bourne again Shell，是 Bourne Shell 的扩展，简称 bash，是 LinuxOS 的默认 SHell，有灵活和强大的编辑接口和油耗的用户界面，交互性很强 |
| /sbin/nologin | 未登录解析器，用于控制用户禁止登录系统的，有时候一些服务，比如邮件服务，大部分都是用来接受主机的邮件而已，并不需要登录                                 |
| /bin/dash     | dash（Debian A 里面 Quist Shell）也是一种 Unix Shell，它比 bash 小，只需要较小的磁盘空间，但是它的对话功能少，交互性差            |
| /bin/csh      | C Shell 是 C 语言风格的 Shell                                                                     |
| /bin/tcsh     | 是 C Shell 的扩展版本                                                                             |

## 编写格式与执行方式
Shell 脚本文件是一个文本文件，后缀名建议使用 `.sh` 结尾；
首行需要设置 Shell 解析器的类型语法为：
```sh
#!/bin/bash
```
> 设置当前 Shell 脚本文件使用 bash 解析器运行脚本代码

注释格式：
```sh
# 注释内容

:<<!
	多行注释格式
!
```
## HelloWorld
### 第一个脚本文件
```sh
#!/bin/bash
echo "hello world"
```
> 在控制台输出 hello world
### 脚本文件常用的三种执行方式
sh 解析器执行方式
```sh
sh [脚本文件路径]
```

bash 执行方式
```sh
bash [脚本文件路径]
```

仅路径执行方式
```sh
[脚本文件]
```
> 脚本文件自己执行需要具有可执行权限，否则无法执行
> 需要通过 `chmod a+x helloworld.sh` 赋予用户对这个脚本的执行权限。
### 三种方式的区别
sh 和 bash 执行方式是直接使用 Shell 解析器运行脚本文件，不需要可执行权限
而仅路径方式是执行脚本文件自己，需要可执行权限。
## 多命令处理
案例：在指定目录中执行脚本，实现在这个目录下创建一个 one.txt，并且在文件中增加内容 "Hello Shell"
```sh
#!/bin/bash
touch ./one.txt
echo "Hello Shell" >> ./one.txt
```