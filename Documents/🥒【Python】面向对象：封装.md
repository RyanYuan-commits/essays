## 实现方式
### 公有属性和方法
在 Python 中，默认情况下，类的属性和方法都是公开的，在类的外部可以直接访问和使用。
```python
class Person:
    def __init__(self, name, age):
        # 公有属性
        self.name = name
        self.age = age

    # 公有方法
    def introduce(self):
        print(f"Hi, my name is {self.name} and I'm {self.age} years old.")

# 创建对象
p = Person("Alice", 25)
# 直接访问公有属性
print(p.name)
# 调用公有方法
p.introduce()
```
输出结果：
```
Alice
Hi, my name is Alice and I'm 25 years old.
```
### 私有属性和方法
Python 通过在属性或者方法名前加上 `__` 来将其变为私有属性或者方法，私有属性和方法只能在类的内部访问，外部无法直接访问。
```python
class Person:
	def __init__(self, name, age):
		# 私有属性	
		self.__name = name
		self.__age = age
	# 公有方法
	def introduce(self):
		print(f"Hi, my name is {self.__name} and I'm {self.__age} years old.")

# 创建对象
p = Person("Alice", 25)
# 调用公有方法，访问私有属性
p.introduce()
# 报错，外部无法直接访问
print(p.__name)
```
输出结果：
```
Hi, my name is Alice and I'm 25 years old.
Traceback (most recent call last):
  File "./test.py", line 16, in <module>
    print(p.__name)
AttributeError: 'Person' object has no attribute '__name'
```
### 受保护的属性和方法
在 Python 中，通过在属性或者方法名前加上 `_` 来表示受保护的属性和方法，虽然受保护的属性和方法在语法上可以在类的外部访问，但按照 Python 的编程约定，它们应该只在类的内部或子类中使用。
```python
class Animal:
    def __init__(self, name):
        # 受保护的属性
        self._name = name

    # 受保护的方法
    def _make_sound(self):
        print(f"{self._name} makes a sound.")

class Dog(Animal):
    def bark(self):
        # 子类可以访问受保护的属性和方法
        self._make_sound()
        print(f"{self._name} barks.")

# 创建对象
dog = Dog("Buddy")
# 可以访问受保护的属性，但不建议这样做
print(dog._name)
# 可以调用受保护的方法，但不建议这样做
dog._make_sound()
# 子类调用受保护的方法
dog.bark()
```
输出结果：
```
Buddy
Buddy makes a sound.
Buddy makes a sound.
Buddy barks.
```

## 方法的分类
### 实例方法
实例方法是类中最常见的方法类型，它第一个参数通常是 `self`，代表类的实例对象，通过 `self` 可以访问和修改实例的属性，也能调用其他的实例方法。
```python
class Person:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    # 实例方法
    def introduce(self):
        print(f"我叫 {self.name}，今年 {self.age} 岁。")

    def have_birthday(self):
        self.age += 1
        print(f"{self.name} 过生日啦，现在 {self.age} 岁了。")


p = Person("张三", 20)
p.introduce()
p.have_birthday()
```
输出结果：
```
我叫 张三，今年 20 岁。
张三 过生日啦，现在 21 岁了。
```
### 类方法
类方法使用 `@classmethod` [[🥓【Python】装饰器]] 定义，其第一个参数通常命名为 `cls`，代表类本身。类方法可以访问和修改类的属性，也能调用其他类方法，并且可以在不创建实例的情况下调用。
```python
class Person:
    total_persons = 0

    def __init__(self, name, age):
        self.name = name
        self.age = age
        Person.total_persons += 1

    @classmethod
    def get_total_persons(cls):
        return cls.total_persons


p1 = Person("李四", 22)
p2 = Person("王五", 25)
print(Person.get_total_persons())
```
- `total_persons` 是类属性，用于记录创建的 `Person` 对象的总数。
- `get_total_persons` 是类方法，通过 `cls` 参数访问类属性 `total_persons`。
输出结果：
```
2
```
### 静态方法
静态方法使用 `@staticmethod` 装饰器定义，它没有类似 `self` 或 `cls` 这样的特殊参数。静态方法通常用于实现与类相关但不依赖于类或实例状态的工具函数。
```python
class MathUtils:
    @staticmethod
    def add(a, b):
        return a + b

result = MathUtils.add(3, 5)
print(result)
```
输出结果：
```
8
```
