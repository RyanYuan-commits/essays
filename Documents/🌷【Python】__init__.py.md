## Python2
### 标识包
`__init__.py` 文件是将一个目录标记为 Python 包的必要条件。
如果一个目录包含了 `__init__.py` 文件，Python 就会把这个目录当作一个包来对待。
### 初始化代码
`__init__.py` 文件中可以包含初始化包所需的代码，这些代码会在包被导入时执行。例如，在 `__init__.py` 中可以设置一些全局变量或者进行一些初始化操作。
```python
# package/__init__.py 
print("Initializing the package...") 
package_variable = 42
```
### 控制导入行为
可以通过 `__all__` 变量来控制 `from package import *` 语句导入的模块。`__all__` 是一个列表，包含了使用 `*` 导入时要导入的模块名称。
```python
# package/__init__.py 
__all__ = ['module1', 'module2']
```
当使用 `from package import *` 时，只会导入 `module1` 和 `module2`。
## Python3
在 Python 3.3 及以后的版本中，引入了隐式命名空间包的概念，`__init__.py` 文件不再是将一个目录标记为包的必要条件。
- **向后兼容性**：为了保持与 Python 2.x 代码的兼容性，或者在一些需要传统包结构的场景中，仍然可以使用 `__init__.py` 文件。
- **初始化和控制导入**：和 Python 2.x 一样，`__init__.py` 可以包含初始化代码和控制导入行为。例如，在 `__init__.py` 中可以导入包内的子模块，方便用户使用。
```python
# package/__init__.py 
from .module1 import func1 
from .module2 import func2
```
这样用户在导入包时可以直接使用 `package.func1` 和 `package.func2`：
```python
import package 
package.func1() 
package.func2()
```