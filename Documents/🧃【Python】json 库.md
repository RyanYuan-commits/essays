---
sticker: emoji//1f685
---
在 Python 中，`json` 库是一个内置的标准库，用于处理 JSON（JavaScript Object Notation）数据。JSON 是一种轻量级的数据交换格式，易于人类阅读和编写，同时也易于机器解析和生成，在 Web 应用、API 数据传输等场景中广泛使用。
## 主要功能
`json` 库主要提供了两种核心功能：
- **序列化（Serialization）**：将 Python 对象（如字典、列表等）转换为 JSON 格式的字符串，这个过程也被称为 “编码”。
- **反序列化（Deserialization）**：将 JSON 格式的字符串转换为 Python 对象，这个过程也被称为 “解码”。

## 常用方法
### 序列化方法
#### json.dumps()
==方法作用==：用于将 Python 对象序列化为 JSON 格式的字符串
```python
json.dumps(obj, *, skipkeys=False, ensure_ascii=True, check_circular=True, allow_nan=True, cls=None, indent=None, separators=None, default=None, sort_keys=False, **kw)
```
- `obj`：要进行序列化的 Python 对象。
- `indent`：可选参数，用于指定缩进的空格数，让生成的 JSON 字符串更易读。
- `sort_keys`：可选参数，若为 `True`，会按照键的字典序对字典进行排序。

```python
import json
source_obj = {
	'name': 'ryan',
	'age': 20
}
print(json.dumps(source_obj, indent=2)) # {"name": "ryan", "age": 20}
```
输出结果
```
{
  "name": "ryan",
  "age": 20
}
```
#### json.dump()
将 Python 对象序列化成 JSON 格式的字符串，**并把它写入文件**。
```python
json.dump(obj, fp, *, skipkeys=False, ensure_ascii=True, check_circular=True, allow_nan=True, cls=None, indent=None, separators=None, default=None, sort_keys=False, **kw)
```
- `obj`：要序列化的 Python 对象。
- `fp`：文件对象，用于写入序列化后的 JSON 数据。
### 反序列化方法
#### json.loads()
将 JSON 格式的字符串反序列化成 Python 对象
```python
json.loads(s, *, cls=None, object_hook=None, parse_float=None, parse_int=None, parse_constant=None, object_pairs_hook=None, **kw)
```
- `s`：要反序列化的 JSON 格式的字符串。
#### json.load()
该方法用于从文件中读取 JSON 格式的数据，并将其反序列化为 Python 对象。
```python
json.load(fp, *, cls=None, object_hook=None, parse_float=None, parse_int=None, parse_constant=None, object_pairs_hook=None, **kw)
```
- `fp`：文件对象，用于读取 JSON 数据。