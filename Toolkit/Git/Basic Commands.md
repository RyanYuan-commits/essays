---
created: 2025-09-27 22:34:06
updated: 2025-09-27 22:34:06
---
![[Git Architecture.gif|800]]

## 1 初始化和配置
| 命令                                    | 描述                  | 示例                                                  |
| ------------------------------------- | ------------------- | --------------------------------------------------- |
| `git init`                            | 在当前目录初始化一个新的 Git 仓库 | `git init`                                          |
| `git clone <url>`                     | 克隆一个远程仓库到本地         | `git clone https://...`                             |
| `git config --global user.name "名称"`  | 配置全局用户名             | `git config --global user.name "RyanYuan"`          |
| `git config --global user.email "邮箱"` | 配置全局用户邮箱            | `git config --global user.email "name@example.com"` |
## 2 基本工作流
| 命令                    | 描述                                        | 示例                       |
| --------------------- | ----------------------------------------- | ------------------------ |
| `git status`          | 查看工作区和暂存区的状态（**常用**）                      | `git status`             |
| `git add <file>`      | 将**指定文件**添加到暂存区                           | `git add index.html`     |
| `git add .`           | 将**所有修改和新增文件**添加到暂存区                      | `git add .`              |
| `git commit -m "消息"`  | 将暂存区的内容提交到仓库，并附上说明                        | `git commit -m "添加登录功能"` |
| `git commit -am "消息"` | 相当于 `git add .` + `git commit -m`（对已跟踪文件） | `git commit -am "修改样式"`  |
| `git log`             | 查看当前分支的提交历史                               | `git log`                |
| `git log --oneline`   | 以简洁的一行格式查看历史                              | `git log --oneline`      |

## 3 分支管理
| 命令                      | 描述                     | 示例                          |
| ----------------------- | ---------------------- | --------------------------- |
| `git branch`            | 查看所有本地分支（当前分支前有 `*` 号） | `git branch`                |
| `git branch <分支名>`      | 创建一个新分支                | `git branch new-feature`    |
| `git checkout <分支名>`    | 切换到指定分支                | `git checkout new-feature`  |
| `git switch <分支名>`      | 更清晰的切换分支命令             | `git switch new-feature`    |
| `git checkout -b <分支名>` | **创建并立即切换**到新分支        | `git checkout -b hotfix`    |
| `git switch -c <分支名>`   | 创建并切换                  | `git switch -c hotfix`      |
| `git branch -d <分支名>`   | **删除**已合并的分支           | `git branch -d old-feature` |
| `git branch -D <分支名>`   | **强制删除**分支（包括未合并的）     | `git branch -D experiment`  |
| `git merge <分支名>`       | **将指定分支合并到当前分支**       | `git merge feature`         |
## 4 与远程仓库交互
| 命令                               | 描述                                                          | 示例                                    |
| -------------------------------- | ----------------------------------------------------------- | ------------------------------------- |
| `git remote -v`                  | 查看已关联的远程仓库地址                                                | `git remote -v`                       |
| `git fetch`                      | 从远程仓库获取最新信息（但不自动合并）                                         | `git fetch`                           |
| `git pull`                       | **拉取**远程分支最新代码并**合并**到当前分支  <br>(`git fetch` + `git merge`) | `git pull origin main`                |
| `git pull --rebase`              | 拉取远程代码并使用 rebase 方式合并（更整洁）                                  | `git pull --rebase`                   |
| `git push`                       | **推送**本地提交到远程仓库                                             | `git push`                            |
| `git push -u origin <分支名>`       | **首次推送**本地分支时，建立跟踪关系                                        | `git push -u origin new-feature`      |
| `git push origin --delete <分支名>` | 删除远程分支                                                      | `git push origin --delete old-branch` |
## 5 撤销和回退
| 命令                             | 描述                                                | 注意          |
| ------------------------------ | ------------------------------------------------- | ----------- |
| `git restore <file>`           | **丢弃工作区的修改**，恢复到最近一次 `git commit` 或 `git add` 的状态 | (Git 2.23+) |
| `git checkout -- <file>`       | 同上（旧命令）                                           |             |
| `git restore --staged <file>`  | **将文件从暂存区撤出**（unstage），但保留工作区的修改                  | (Git 2.23+) |
| `git reset HEAD <file>`        | 同上（旧命令）                                           |             |
| `git reset --hard <commit-id>` | **危险！** 强行回退到指定提交，丢弃所有之后的提交和工作区修改                 | 仅用于本地分支     |
| `git revert <commit-id>`       | **安全撤销**。创建一个新的提交来抵消指定提交的更改                       | 适用于公共分支     |

## 6 其他实用命令
| 命令                   | 描述                                 | 示例                                   |
| -------------------- | ---------------------------------- | ------------------------------------ |
| `git diff`           | 查看**工作区**和**暂存区**的差异               | `git diff`                           |
| `git diff --staged`  | 查看**暂存区**和**最后一次提交**的差异            | `git diff --staged`                  |
| `git rm <file>`      | 从 Git 和工作区中**删除文件**                | `git rm old-file.txt`                |
| `git mv <old> <new>` | **移动或重命名**文件                       | `git mv old-name.txt new-name.txt`   |
| `.gitignore`         | **（重要文件）** 在其中写入文件名/模式，Git 会自动忽略它们 | `echo "node_modules/" >> .gitignore` |
