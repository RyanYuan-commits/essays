>[!tips] pymysql 是什么？
>pymysql 是一个用于在 Python 中连接和操作 MySQL 数据库的库，它实现了 Python 数据库 API（DB-API）规范，提供了方便的接口让开发者能够在 Python 代码里与 MySQL 数据库进行交互。

## 安装
```python
pip install pymysql
```
## 基本使用流程
### 创建数据库连接
```python
import pymysql

# 建立数据库连接
conn = pymysql.connect(
    host='localhost',  # 数据库主机地址
    user='your_username',  # 数据库用户名
    password='your_password',  # 数据库密码
    database='your_database',  # 要连接的数据库名
    charset='utf8mb4',  # 字符编码
    cursorclass=pymysql.cursors.DictCursor  # 使用字典游标，查询结果以字典形式返回
)
```
- `pymysql.connect()` 函数用于创建一个数据库连接对象，通过传入主机地址、用户名、密码等参数来建立连接。
- `cursorclass=pymysql.cursors.DictCursor` 使得后续查询返回的结果以字典形式呈现，方便通过键名访问数据。
### 创建游标对象
```python
try:
    with conn.cursor() as cursor:
        # 这里的 cursor 就是创建的游标对象
        pass
finally:
    conn.close()
```
- `conn.cursor()` 方法用于创建一个游标对象，`with` 确保游标的自动关闭。
### 执行 SQL 语句
#### 插入数据
```python
import pymysql
# 获取 conn
try:
    with conn.cursor() as cursor:
        # 定义插入语句
        sql = "INSERT INTO your_table (column1, column2) VALUES (%s, %s)"
        # 执行插入操作
        cursor.execute(sql, ('value1', 'value2'))
    # 提交事务
    conn.commit()
except Exception as e:
    print(f"Error: {e}")
    # 发生错误时回滚事务
    conn.rollback()
finally:
    conn.close()
```
#### 批量插入数据
```python
columns = ', '.join(data_list[0].keys())
placeholders = ', '.join(['%s'] * len(data_list[0]))
sql = f"INSERT INTO {table} ({columns}) VALUES ({placeholders})"
# 批量插入数据
cursor.executemany(sql, [tuple(item.values()) for item in data_list])
```
- `[tuple(item.values()) for item in data_list]`：[[🍙【Python】列表推导式]]
#### 查询数据
```python
import pymysql

# 获取 conn

try:
    with conn.cursor() as cursor:
        # 定义查询语句
        sql = "SELECT * FROM your_table"
        # 执行查询操作
        cursor.execute(sql)
        # 获取查询结果
        results = cursor.fetchall()
        for row in results:
            print(row)
except Exception as e:
    print(f"Error: {e}")
finally:
    conn.close()
```
#### 更新数据
```python
import pymysql

# 获取 conn

try:
    with conn.cursor() as cursor:
        # 定义更新语句
        sql = "UPDATE your_table SET column1 = %s WHERE id = %s"
        # 执行更新操作
        cursor.execute(sql, ('new_value', 1))
    # 提交事务
    conn.commit()
except Exception as e:
    print(f"Error: {e}")
    # 发生错误时回滚事务
    conn.rollback()
finally:
    conn.close()
```
#### 删除数据
```python
import pymysql

# 获取 conn
try:
    with conn.cursor() as cursor:
        # 定义删除语句
        sql = "DELETE FROM your_table WHERE id = %s"
        # 执行删除操作
        cursor.execute(sql, (1,))
    # 提交事务
    conn.commit()
except Exception as e:
    print(f"Error: {e}")
    # 发生错误时回滚事务
    conn.rollback()
finally:
    conn.close()
```
