Flask 的蓝图（Blueprints）是一种组织代码的机制，允许你将 Flask 应用分解成多个模块。这样可以更好地组织应用逻辑，使得应用更具可维护性和可扩展性。
每个蓝图可以有自己的 **路由、视图函数、模板和静态文件**，这样可以将相关的功能分组。
通过使用蓝图，你可以将 Flask 应用拆分成多个模块，每个模块处理相关的功能，使得代码更加清晰和易于管理。
## 创建蓝图
创建蓝图涉及到一下的几个步骤：
- 定义蓝图：在一个独立的木块中定义蓝图
- 注册蓝图：在住应用中注册蓝图，使其生效
假设我们要创建一个博客应用，其中包含用户管理和博客功能，我们可以将这些功能分成两个蓝图：auth 和 blog。
```python
yourapp/
│
├── app.py
├── auth/
│   ├── __init__.py
│   └── routes.py
│
└── blog/
    ├── __init__.py
    └── routes.py
```
## 定义蓝图
```python
from flask import Blueprint, render_template, request, redirect, url_for

auth = Blueprint('auth', __name__)

@auth.route('/login')
def login():
    return render_template('login.html')

@auth.route('/logout')
def logout():
    return redirect(url_for('auth.login'))

@auth.route('/register')
def register():
    return render_template('register.html')
```
- Blueprint('auth', __name__)：创建一个名为 auth 的蓝图。
- 蓝图中定义的路由函数可以用来处理请求。
```python
from flask import Blueprint, render_template

blog = Blueprint('blog', __name__)

@blog.route('/')
def index():
    return render_template('index.html')

@blog.route('/post/<int:post_id>')
def post(post_id):
    return f'Post ID: {post_id}'
```
- Blueprint('blog', __name__)：创建一个名为 blog 的蓝图。
## 注册蓝图
```python
from flask import Flask

app = Flask(__name__)

# 导入蓝图
from auth.routes import auth
from blog.routes import blog

# 注册蓝图
app.register_blueprint(auth, url_prefix='/auth')
app.register_blueprint(blog, url_prefix='/blog')

if __name__ == '__main__':
    app.run(debug=True)
```
- `app.register_blueprint(auth, url_prefix='/auth')`：注册 `auth` 蓝图，并将所有的路由前缀设置为 `/auth`。
- `app.register_blueprint(blog, url_prefix='/blog')`：注册 `blog` 蓝图，并将所有的路由前缀设置为 `/blog`。

## 使用蓝图中的模板和静态文件
蓝图中的模板和静态文件应放在蓝图的文件夹下的 templates 和 static 子文件夹中。
```python
yourapp/
│
├── app.py
├── auth/
│   ├── __init__.py
│   ├── routes.py
│   └── templates/
│       ├── login.html
│       └── register.html
│
└── blog/
    ├── __init__.py
    ├── routes.py
    └── templates/
        ├── index.html
        └── post.html
```
## 在蓝图中使用请求钩子
```python
@auth.before_app_request
def before_request():
    # 执行在每个请求之前的操作
    pass

@auth.after_app_request
def after_request(response):
    # 执行在每个请求之后的操作
    return response
```
## 在蓝图中定义错误处理
```python
@blog.errorhandler(404)
def page_not_found(error):
    return 'Page not found', 404
```