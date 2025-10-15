---
type: Toolkit
sub-type: Git
---
`.gitignore` 文件用于告诉 Git 哪些文件和目录应该被忽略, 不纳入版本控制.

## 1	Core Syntax

 `.gitignore` 位于仓库根目下, 每一行都是一个匹配模式:
 
| **规则**            | **作用**                       | **示例**                     |
| ----------------- | ---------------------------- | -------------------------- |
| `file.txt`        | 忽略仓库根目录下所有的 `file.txt` 文件    | config.local.json          |
| `dir/`            | 忽略名为 `dir` 的目录及其所有的内容        | `target/`                  |
| `*.log`           | 忽略以 `.log` 结尾的文件             | `*.iml`                    |
| `/file.txt`       | 忽略当前目录下的 `file.txt`          | `/README.md` 只忽略根目录, 子目录保留 |
| `!file.txt`       | 取消忽略, 重新纳入追踪                 | `!src/main/`               |
| `dir/**/file.txt` | 匹配 `dir` 目录下任意深度的 `file.txt` | `logs/**/*.log`            |

