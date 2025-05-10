## 1 什么是 Shell？
![[shell.png#pic_center]]
通过编写 Shell 命令，发送给 Linux 内核去执行，操作的就是计算机硬件了，Shell 命令是用户操作内核的桥梁，Shell 脚本通过 Shell 命令或者编程语言编写的 Shell 文本文件，这就是 Shell 脚本，也叫 Shell 程序。
通过 `cat /etc/shells` 命令查看 Linux 提供的 Shell 解析器，通过 `echo $SHELL` 打印输出当前系统使用的 SHELL 解析器类型，常见的 Shell 解释器有以下几种：
- `/bin/sh`：Bourne Shell，是 Linux 最初使用的 Shell；
- `/bin/bash`：Bourne again Shell，是 `/bin/sh` 的扩展，是 LinuxOS 的默认 Shell，有灵活和强大的编辑接口和有好的用户界面，交互性强；
- `/sbin/nologin`：未登录解析器，用于控制用户禁止登录系统的，有时候一些服务，比如邮件服务，并不需要登录；
- `/bin/dash`：dash（Debian A 的 Quist Shell）也是一种 Unix Shell，它比 bash 小，只需要较小的磁盘空间，但是对话功能少，交互性差；
- `/bin/csh`：C Shell 是 C 语言风格的 Shell；
- `/bin/tcsh`：是 C Shell 的扩展版本。
## 2 基本格式
Shell 脚本文件是一个文本文件，后缀名建议使用 `.sh` 结尾，首行需要设置 Shell 解析器的类型语法：
```sh
# 设置当前 Shell 脚本文件使用 bash 解析器运行脚本代码
#!/bin/bash
```
注释格式：
```sh
# 单行注释格式

:<<!
	多行注释格式
!
```
## 3 Quick Start
```sh
#!/bin/bash
echo "hello world"
```
常用的执行方式有三种：sh 执行、bash执行和直接运行脚本，sh 和 bash 执行方式是直接使用 Shell 解析器运行脚本文件，不需要可执行权限，而仅路径方式是执行脚本文件自己，需要可执行权限。
> 脚本文件自己执行需要具有可执行权限，否则无法执行
> 需要通过 `chmod a+x helloworld.sh` 赋予用户对这个脚本的执行权限。