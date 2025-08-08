## 数字类型
### 整数
表示整数，没有小数部分。例如：`5`, `-10`, `1000`。
```python
x = 5 
print(type(x)) # <class 'int'>
```
### 浮点数
表示带有小数部分的数字。例如：`3.14`, `-0.5`, `1.0`。
```python
y = 3.14 
print(type(y)) # <class 'float'>
```
### 复数
```python
z = 2 + 3j 
print(type(z)) # <class 'complex'>
```
## 序列类型
### 列表
是一种有序、可变的序列，可存储不同类型的元素。列表使用方括号 `[]` 表示。
```python
my_list = [1, 2, 'three', 4.0]
print(my_list)  # [1, 2, 'three', 4.0]
my_list.append(5)
print(my_list)  # [1, 2, 'three', 4.0, 5]
```
### 元组
是一种有序、不可变的序列，使用圆括号 `()` 表示。
```python
my_tuple = (1, 2, 'three', 4.0) 
print(my_tuple) # (1, 2, 'three', 4.0)
```
### 字符串
```python
my_string = "Hello, World!" 
print(my_string[0]) # H 
print(len(my_string)) # 13
```
## 集合类型
### 集合
是一种无序、不重复的集合，使用花括号 `{}` 表示（空集合用 `set()` 表示，而不是 `{}`，因为 `{}` 表示空字典）。
```python
my_set = {1, 2, 3, 3, 4} 
print(my_set) # {1, 2, 3, 4} 
my_set.add(5) 
print(my_set) # {1, 2, 3, 4, 5}
```
### 冻结集合
是一种不可变的集合，一旦创建就不能修改。
```python
my_frozen_set = frozenset([1, 2, 3]) 
print(my_frozen_set) # frozenset({1, 2, 3})
```
## 映射类型
### 字典
存储键值对，使用花括号 `{}` 表示，键和值之间用冒号 `:` 分隔。
```python
my_dict = {'key1': 'value1', 'key2': 2} 
print(my_dict['key1']) # value1 
my_dict['key3'] = 3 
print(my_dict) # {'key1': 'value1', 'key2': 2, 'key3': 3}
```
字典的遍历方式：
- 遍历字典的键：`for key in my_dict.keys()`
- 遍历字典的值：`for value in my_dict.values()`
- 同时遍历字典的键值对：`for key, value in my_dict.items()`
## 布尔类型
表示逻辑值，只有 `True` 和 `False` 两个值。
```python
is_true = True 
is_false = False 
print(type(is_true)) # <class 'bool'>
```
## 数据类型转换
可以使用内置函数进行数据类型转换，例如：
```python
num_str = "123" 
num_int = int(num_str) 
print(type(num_int)) # <class 'int'>
```
