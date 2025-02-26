## 基本语法
使用 `class` 关键字来定义子类，并在括号中指定父类。例如：
```python
class ParentClass:
    # 父类的属性和方法

class ChildClass(ParentClass):
    # 子类的属性和方法
```
## 方法重写
子类可以重写父类的方法，即在子类中定义与父类同名的方法。当子类对象调用该方法时，会执行子类中重写的方法，而不是父类的方法。这允许子类根据自己的需求对父类的方法进行定制化修改。
例如：
```python
class ParentClass:
    def greet(self):
        print("Hello from ParentClass.")

class ChildClass(ParentClass):
    def greet(self):
        print("Hello from ChildClass.")
```
在上面的例子中，`ChildClass` 重写了 `ParentClass` 的 `greet` 方法，当调用 `greet` 方法时，会输出 `Hello from ChildClass.`。
## 多继承
Python 支持多继承，即一个子类可以继承多个父类。子类会继承所有父类的属性和方法。
在多继承情况下，如果多个父类中有同名的方法，子类在调用该方法时需要明确指定使用哪个父类的方法，或者通过特定的继承顺序来确定方法的调用顺序。
```python
class ClassA:
    def method_a(self):
        print("Method A from ClassA.")

class ClassB:
    def method_b(self):
        print("Method B from ClassB.")

class ChildClass(ClassA, ClassB):
    pass
```
在上述代码中，`ChildClass` 继承了 `ClassA` 和 `ClassB`，可以调用它们的方法。