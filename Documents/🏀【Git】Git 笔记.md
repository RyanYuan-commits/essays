## 安装 Git
官网下载安装 https://git-scm.com/

安装完成后，配置名称和邮箱
```bash
$git config --global user.name "你的名字"
$git config --global user.email "你的邮箱"
```
在进行代码提交（`git commit`）时，配置的名字和邮箱会被记录在每一次提交信息中。当你查看提交历史（使用 `git log` 命令）时，就能看到每个提交是由谁完成的。例如：
```
commit 123456789abcdef 
Author: John Doe <johndoe@example.com> 
Date: Mon Sep 18 12:00:00 2023 +0800 
	Add new feature
```
## 仓库
本地仓库等于工作区加上版本区，工作区是磁盘上文件的集合，版本区就是 .git 文件