---
type: Python
sub-type: API
finished: "true"
---
## 1. 文件读写的基石: `open()` 与 `with` 语句

这是所有文件操作的起点. **永远**使用 `with open(...)` 语法, 它能确保文件在操作完成后被自动关闭, 即使发生错误也不例外. 

### 1.1 核心语法

```Python
with open('文件名', '模式', encoding='utf-8') as f:
    # 对文件对象 f 进行操作 
    ...
# 文件在此处自动关闭
```

### 1.2 常用模式
| 模式    | 描述              | 注意                                                     |
| ----- | --------------- | ------------------------------------------------------ |
| `'r'` | **读取 (Read)**   | 默认模式. 如果文件不存在, 会抛出 `FileNotFoundError`.                   |
| `'w'` | **写入 (Write)**  | **会覆盖**已有文件. 如果文件不存在, 则创建.                                |
| `'a'` | **追加 (Append)** | 在文件末尾追加内容. 如果文件不存在, 则创建.                                  |
| `'+'` | 读写模式            | 通常与 r, w, a 结合, 如 `'r+'`, `'w+'`.                        |
| `'b'` | **二进制模式**       | 用于处理非文本文件, 如图片、音频. 与 r, w, a 结合成 `'rb'`, `'wb'`, `'ab'`.  
⚠️ 关键: 指定编码: 处理文本文件时, 务必显式指定 encoding='utf-8'. 这是避免在不同操作系统上出现乱码问题的最重要实践. 

## 2. 核心操作: 读取与写入

### 2.1 读取文件

```Python
# a. 读取全部内容 (适合小文件)
with open('config.txt', 'r', encoding='utf-8') as f:
    content = f.read()
    # print(content)

# b. 逐行读取 (推荐, 内存效率最高, 适合所有文件)
with open('log.txt', 'r', encoding='utf-8') as f:
    for line in f:
        # line 自带换行符, 使用 strip() 去除
        print(line.strip())

# c. 读取所有行到列表中 (适合中小文件)
with open('requirements.txt', 'r', encoding='utf-8') as f:
    lines = f.readlines()
    # lines 是一个列表, 每一项是一行字符串, 包含换行符
    # print(lines)
```

### 2.2 写入文件

```Python
# a. 写入字符串
with open('output.txt', 'w', encoding='utf-8') as f:
    f.write('第一行\n')
    f.write('这是第二行\n')

# b. 写入一个字符串列表 (writelines 不会自动加换行符)
lines_to_write = ['标题\n', '内容第一段\n', '内容第二段\n']
with open('article.txt', 'w', encoding='utf-8') as f:
    f.writelines(lines_to_write)
```

## 3. 现代路径管理: `pathlib` 模块

从 Python 3.4+ 开始, `pathlib` 是处理文件路径的**首选方式**. 它用面向对象的方式替代了陈旧的 `os.path`, 代码更直观、更不易出错. 

```Python
from pathlib import Path

# 1. 创建路径对象
# Path.home() 获取用户主目录
# Path.cwd() 获取当前工作目录
config_path = Path.home() / 'configs' / 'app.conf'
data_dir = Path('./data') # 相对路径

# 2. 路径拼接 (使用 / 运算符, 非常优雅)
log_file = data_dir / 'logs' / 'latest.log'
print(f"日志文件路径: {log_file}")

# 3. 检查路径是否存在
if data_dir.exists():
    print(f"目录 {data_dir} 已存在")

# 4. 判断是文件还是目录
print(f"{log_file} 是文件吗? {log_file.is_file()}")
print(f"{data_dir} 是目录吗? {data_dir.is_dir()}")

# 5. 创建目录
# parents=True: 递归创建父目录
# exist_ok=True: 如果目录已存在, 不报错
log_dir = data_dir / 'logs'
log_dir.mkdir(parents=True, exist_ok=True)

# 6. 获取文件名、父目录、后缀
print(f"文件名: {log_file.name}")
print(f"父目录: {log_file.parent}")
print(f"文件后缀: {log_file.suffix}")

# 7. 遍历目录中的文件
# .glob('*.csv') 可以按通配符查找
for csv_file in data_dir.glob('*.csv'):
    print(f"找到CSV文件: {csv_file}")
    
# 8. 删除文件 (目录用 .rmdir())
if log_file.exists():
    log_file.unlink() # 删除文件
```

`pathlib`可以直接与`open()`结合使用:

```Python
my_file = Path('./data/my_text.txt')
with open(my_file, 'w', encoding='utf-8') as f:
    f.write('使用 pathlib 非常方便!')
```

## 4. 处理特定文件格式

### 4.1 JSON 文件

```Python
import json
from pathlib import Path

data = {
    'user': 'admin',
    'permissions': ['read', 'write'],
    'config': {'retries': 3, 'timeout': 60}
}
json_file = Path('settings.json')

# 写入 JSON (dump)
with open(json_file, 'w', encoding='utf-8') as f:
    # indent=4 使格式更美观
    json.dump(data, f, ensure_ascii=False, indent=4)

# 读取 JSON (load)
with open(json_file, 'r', encoding='utf-8') as f:
    loaded_data = json.load(f)
    print(f"读取到的超时配置: {loaded_data['config']['timeout']}")

```

### 4.2 CSV 文件

```Python
import csv
from pathlib import Path

csv_data = [
    ['ID', 'Name', 'Department'],
    ['001', '张三', '技术部'],
    ['002', '李四', '市场部']
]
csv_file = Path('report.csv')

# 写入 CSV
# newline='' 是为了防止写入空行
with open(csv_file, 'w', encoding='utf-8', newline='') as f:
    writer = csv.writer(f)
    writer.writerows(csv_data)
    
# 读取 CSV
with open(csv_file, 'r', encoding='utf-8') as f:
    reader = csv.reader(f)
    # 跳过标题行
    header = next(reader)
    for row in reader:
        # row 是一个列表
        print(f"ID: {row[0]}, 姓名: {row[1]}")
```

### 4.3 二进制文件(如图片)

关键在于使用二进制模式 `'b'`. 

```Python
from pathlib import Path

# 假设 source.jpg 存在
source_image = Path('source.jpg')
destination_image = Path('destination_copy.jpg')

# 复制图片
try:
    with open(source_image, 'rb') as f_read:
        content = f_read.read()
    
    with open(destination_image, 'wb') as f_write:
        f_write.write(content)
    print("图片复制成功!")
except FileNotFoundError:
    print(f"错误: 源文件 {source_image} 不存在!")
```

## 5. 重要技巧与“陷阱”

1. 编码, 编码, 还是编码!
    
    处理文本时, 永远不要忘记 encoding='utf-8'. 它是你代码健壮性的保证. 
    
2. 处理大文件
    
    不要对一个几 GB 大的文件使用 f.read() 或 f.readlines(), 这会耗尽你的内存. 坚持使用 for line in f: 的逐行读取方式. 
    
3. 文件不存在与权限问题
    
    文件操作随时可能因为文件不存在 (FileNotFoundError) 或权限不足 (PermissionError) 而失败. 在生产代码中, 使用 try...except 块来捕获这些异常, 或使用 Path.exists() 提前检查. 