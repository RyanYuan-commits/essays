## 模块
### 是什么？
模块是一个包含 Python 代码的文件，通常以 `.py` 为扩展名。该文件可以包含函数、类、变量和可执行代码等；
例如，创建一个名为 `my_module.py` 的文件，它就是一个模块。
### 案例
#### 定义模块
```python
# my_module.py 
def hello(): 
	print("Hello from my_module") 
class MyClass: 
	def __init__(self, name): 
		self.name = name PI = 3.1415926
```
- `def hello():`：定义一个名为 `hello` 的函数，用于打印消息。
- `class MyClass:`：定义一个名为 `MyClass` 的类，带有一个构造函数。
- `PI = 3.1415926`：定义一个变量 `PI`。
#### 使用模块
```python
import my_module 

my_module.hello() 
obj = my_module.MyClass("John") 
print(my_module.PI)
```
## 包
### 是什么？
包是一个包含 `__init__.py` 文件的目录，该目录可以包含多个模块和子包。
`__init__.py` 文件可以是一个空文件，也可以包含 Python 代码，用于初始化包或设置 `__all__` 变量。
### 案例
#### 示例结构
```
my_package/
    __init__.py
    module1.py
    module2.py
    sub_package/
        __init__.py
        sub_module1.py
```
#### 示例代码
```python
# my_package/module1.py
def func1():
    print("This is func1 from module1")


# my_package/module2.py
def func2():
    print("This is func2 from module2")


# my_package/sub_package/sub_module1.py
def sub_func1():
    print("This is sub_func1 from sub_module1")
```
- `my_package` 是一个包，包含 `module1.py`、`module2.py` 和 `sub_package` 子包。
- `sub_package` 是一个子包，包含 `sub_module1.py`。
#### 使用包
```python
import my_package.module1
import my_package.sub_package.sub_module1

my_package.module1.func1()
my_package.sub_package.sub_module1.sub_func1()
```
- `import my_package.module1`：导入 `my_package` 中的 `module1` 模块。
- `import my_package.sub_package.sub_module1`：导入 `sub_package` 中的 `sub_module1` 模块。
- `my_package.module1.func1()`：调用 `module1` 中的 `func1` 函数。
- `my_package.sub_package.sub_module1.sub_func1()`：调用 `sub_module1` 中的 `sub_func1` 函数。