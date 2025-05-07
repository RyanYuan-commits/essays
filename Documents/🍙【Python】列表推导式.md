列表推导式是 Python 中一种简洁的创建列表的方式，它可以根据已有的可迭代对象（如列表、元组、集合、字典等）快速生成一个新的列表。
## 写法
```python
[expression for item in iterable if condition]
```
- `expression`：对 `item` 进行的操作，将结果添加到新列表中。
- `item`：可迭代对象中的元素。
- `iterable`：可迭代对象，如列表、元组、集合、字典等。
- `if condition`：可选，对 `item` 的筛选条件，只有满足条件的 `item` 才会执行 `expression` 操作。
## 案例
```python
# 生成一个包含 0 到 9 的平方的列表 
squares = [x ** 2 for x in range(10)]
print(squares)
```
- `x` 从 `range(10)` 中取值，即 0 到 9。
- `x ** 2` 计算 `x` 的平方。