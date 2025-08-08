在 Python 中，`@xxx` 这种语法被称为 **装饰器（Decorator）**。装饰器是一种特殊的函数或类，用于修改或扩展其他函数或类的行为。
装饰器其实类似语法糖，将 `origin_func` 转化为 `decorator_func(origin_func)`
## 1 函数装饰器
```python
def my_decorator(func):
    def wrapper():
        print("Before the function is called.")
        func()
        print("After the function is called.")
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
```
输出结果：
```python
Before the function is called.
Hello!
After the function is called.
```
- `@my_decorator` 是一个装饰器的语法糖，等价于 `say_hello = my_decorator(say_hello)`
- `wrapper`：装饰器内部定义了一个新的函数 `wrapper` 用于包裹原函数，并添加额外的行为
## 2 类装饰器
装饰器不只可以用于函数，还可以用于类，类装饰器接收一个类作为参数，并返回一个新的类或修改后的类。
```python
def add_method(cls):
    def new_method(self):
        return "This is a new method"
    cls.new_method = new_method
    return cls

@add_method
class MyClass:
    pass

obj = MyClass()
print(obj.new_method())  # 输出: This is a new method
```
## 3 常见用途
### 3.1 日志记录
```python
def log_decorator(func):
    def wrapper(*args, **kwargs):
        print(f"Calling function: {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Function {func.__name__} finished")
        return result
    return wrapper

@log_decorator
def add(a, b):
    return a + b

print(add(2, 3))
```
### 3.2 权限验证
```python
def auth_required(func):
    def wrapper(*args, **kwargs):
        if check_auth():  # 假设这是一个验证函数
            return func(*args, **kwargs)
        else:
            raise PermissionError("Authentication failed")
    return wrapper

@auth_required
def sensitive_operation():
    print("Performing sensitive operation")

sensitive_operation()
```
## 4 内置装饰器
### 4.1 @staticmethod
将这个方法定义为静态方法，不需要实例化类就可以使用。
```python
class Math:
    @staticmethod
    def add(a, b):
        return a + b

print(Math.add(2, 3))  # 输出: 5
```
### 4.2 @classmethod
将这个方法定义为一个类方法，类方法的第一个参数是类本身，通常命名为 cls 或者 self，调用方法的时候会自动传入。
```python
class MyClass:
    @classmethod
    def class_method(cls):
        return f"Called from {cls.__name__}"

print(MyClass.class_method())  # 输出: Called from MyClass
```
### 4.3 @property
将方法转化为属性，使其可以像属性一样使用
```python
class Circle:
    def __init__(self, radius):
        self.radius = radius

    @property
    def area(self):
        return 3.14 * self.radius ** 2

circle = Circle(5)
print(circle.area)  # 输出: 78.5
```