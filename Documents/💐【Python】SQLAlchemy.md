在 Flask 中，数据库操作是构建 Web 应用的一个重要方面。
Flask 提供了多种方式来与数据库进行交互，包括直接使用 SQL 和利用 ORM（对象关系映射）工具，如 SQLAlchemy。
## 使用 SQLAlchemy
### 安装 Flask-SQLAlchemy
```python
pip install flask-sqlalchemy
```
### 配置 SQLAlchemy
```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///example.db'  # 使用 SQLite 数据库
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)
```
## 定义模型
模型是数据库表的 Python 类，每个模型代表数据库中的一张表。
```python
class User(db.Model):
	 __tablename__ = 'users'  # 指定表名
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)

    def __repr__(self):
        return f'<User {self.username}>'
```
- `db.Model`：所有模型类需要继承自 db.Model。
- `db.Column`：定义模型的字段，指定字段的类型、是否为主键、是否唯一、是否可以为空等属性。
## 基本的 CRUD 操作
### 创建记录
```python
@app.route('/add_user')
def add_user():
    new_user = User(username='john_doe', email='john@example.com')
    db.session.add(new_user)
    db.session.commit()
    return 'User added!'
```
- `db.session.add(new_user)`：将新用户对象添加到会话中。
- `db.session.commit()`：提交事务，将更改保存到数据库。
### 读取记录
```python
@app.route('/get_users')
def get_users():
    users = User.query.all()  # 获取所有用户
    return '<br>'.join([f'{user.username} ({user.email})' for user in users])
```
### 更新记录
```python
@app.route('/update_user/<int:user_id>')
def update_user(user_id):
    user = User.query.get(user_id)
    if user:
        user.username = 'new_username'
        db.session.commit()
        return 'User updated!'
    return 'User not found!'
```
- `User.query.get(user_id)`：通过主键查询单个用户记录。
- 更新字段值并提交事务。
### 删除记录
```python
@app.route('/delete_user/<int:user_id>')
def delete_user(user_id):
    user = User.query.get(user_id)
    if user:
        db.session.delete(user)
        db.session.commit()
        return 'User deleted!'
    return 'User not found!'
```
## 查询操作
### 基本查询
```python
users = User.query.filter_by(username='john_doe').all()
```
- filter_by()：根据字段值过滤记录。
### 复杂查询
```python
from sqlalchemy import or_

users = User.query.filter(or_(User.username == 'john_doe', User.email == 'john@example.com')).all()
```
- or_()：用于执行复杂的查询条件。
### 排序和分页
```python
users = User.query.order_by(User.username).paginate(page=1, per_page=10)
```
- order_by()：按指定字段排序。
- paginate()：分页查询。
## 执行原始 SQL
```python
@app.route('/raw_sql')
def raw_sql():
    result = db.session.execute('SELECT * FROM user')
    return '<br>'.join([str(row) for row in result])
```
## 使用 SQLAlchemy 使用 MySQL
==安装必要的库==
```python
pip install flask-sqlalchemy mysqlclient
```
- Flask-SQLAlchemy 是 Flask 的一个扩展，它简化了 SQLAlchemy 的配置和操作。
- 要连接 MySQL，还需要安装 Flask-SQLAlchemy 和 MySQL 驱动。

==配置 Flask-SQLAlchemy==
```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

app = Flask(__name__)
app.config['SQLALCHEMY_DATABASE_URI'] = 'mysql://username:password@localhost/dbname'
app.config['SQLALCHEMY_TRACK_MODIFICATIONS'] = False
db = SQLAlchemy(app)
```
`SQLALCHEMY_DATABASE_URI`：设置数据库连接 URI，格式为 `mysql://username:password@localhost/dbname`。
- `username`：MySQL 用户名。
- `password`：MySQL 密码。
- `localhost`：MySQL 主机地址（本地通常为 `localhost`）。
- `dbname`：数据库名称。

==定义模型和执行基本操作==
```python
class User(db.Model):
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(80), unique=True, nullable=False)
    email = db.Column(db.String(120), unique=True, nullable=False)

    def __repr__(self):
        return f'<User {self.username}>'

@app.route('/')
def index():
    users = User.query.all()
    return '<br>'.join([f'{user.username} ({user.email})' for user in users])

if __name__ == '__main__':
    with app.app_context():
        db.create_all()  # 创建数据库表
    app.run(debug=True)
```