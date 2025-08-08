![[Flask 基本概念.png|800]]
## 路由 Routing
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return 'Welcome to the Home Page!'

@app.route('/about')
def about():
    return 'This is the About Page.'
```
- `@app.route('/')`：将根 URL `/` 映射到 `home` 函数。
- `@app.route('/about')`：将 `/about` URL 映射到 `about` 函数。
## 视图函数 View Functions
视图函数是处理请求并返回响应的 Python 函数。它们通常接收请求对象作为参数，并返回响应对象，或者直接返回字符串、HTML 等内容。
```python
from flask import request

@app.route('/greet/<name>')
def greet(name):
    return f'Hello, {name}!'
```
greet 函数接收 URL 中的 name 参数，并返回一个字符串响应。
## 请求对象 Request Object
求对象包含了客户端发送的请求信息，包括请求方法、URL、请求头、表单数据等。Flask 提供了 request 对象来访问这些信息。
```python
from flask import request

@app.route('/submit', methods=['POST'])
def submit():
    username = request.form.get('username')
    return f'Hello, {username}!'
```
request.form.get('username')：获取 POST 请求中表单数据的 username 字段。
## 响应对象 Response Object
响应对象包含了发送给客户端的响应信息，包括状态码、响应头和响应体。Flask 默认会将字符串、HTML 直接作为响应体。
```python
from flask import make_response

@app.route('/custom_response')
def custom_response():
    response = make_response('This is a custom response!')
    response.headers['X-Custom-Header'] = 'Value'
    return response
```
make_response：创建一个自定义响应对象，并设置响应头 X-Custom-Header。
## 模板 Template
Flask 使用 Jinja2 模板引擎来渲染 HTML 模板。模板允许你将 Python 代码嵌入到 HTML 中，从而动态生成网页内容。
```python
from flask import render_template

@app.route('/hello/<name>')
def hello(name):
    return render_template('hello.html', name=name)
```
模板文件：
```html
<!DOCTYPE html>
<html>
<head>
    <title>Hello</title>
</head>
<body>
    <h1>Hello, {{ name }}!</h1>
</body>
</html>
```
### 应用工厂
应用工厂是一个 Python 函数，用于创建和配置 Flask 应用实例。这种方法允许你创建多个应用实例，或者在不同配置下初始化应用。
```python
from flask import Flask

def create_app(config_name):
    app = Flask(__name__)
    app.config.from_object(config_name)

    from . import routes
    app.register_blueprint(routes.bp)

    return app
```
create_app 函数创建一个 Flask 应用实例，并从配置对象中加载配置。
## 配置对象 Configuration Objects
配置对象用于设置应用的各种配置选项，如数据库连接字符串、调试模式等。可以通过直接设置或加载配置文件来配置 Flask 应用。
```python
class Config:
    DEBUG = True
    SECRET_KEY = 'mysecretkey'
    SQLALCHEMY_DATABASE_URI = 'sqlite:///mydatabase.db'
```
app.config.from_object(Config)：将 Config 类中的配置项加载到应用配置中。
## 蓝图 Blueprints
蓝图是 Flask 中的组织代码的方式。它允许你将相关的视图函数、模板和静态文件组织在一起，并且可以在多个应用中重用。
```python
from flask import Blueprint

bp = Blueprint('main', __name__)

@bp.route('/')
def home():
    return 'Home Page'
```
注册蓝图 (`app/__init__.py`)：
```python
from flask import Flask
from .routes import bp as main_bp

def create_app():
    app = Flask(__name__)
    app.register_blueprint(main_bp)
    return app
```
## 静态文件 Static Files
静态文件是不会被服务器端执行的文件，如 CSS、JavaScript 和图片文件。Flask 提供了一个简单的方法来服务这些文件。
访问静态文件示例：
```python
<link rel="stylesheet" type="text/css" href="{{ url_for('static', filename='style.css') }}">
```
静态文件目录：将静态文件放在 static 文件中，Flask 会自动提供服务
## 扩展 Extensions
Flask 有许多扩展，可以添加额外的功能，比如数据库集成、表单验证、用户认证等。这些拓展提供了更高级的功能和第三方集成。
```python
from flask_sqlalchemy import SQLAlchemy

app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///mydatabase.db'
db = SQLAlchemy(app)
```
SQLAlchemy：用于数据库集成的扩展。
## 会话 Session
Flask 使用客户端会话来存储用户信息，以便在用户浏览应用的时候记住他们的状态。会话数据存储在客户端的 cookie 中，并在服务器端进行签名和加密。
```python
from flask import session

# 自动生成的密钥
app.secret_key = 'your_secret_key_here'

@app.route('/set_session/<username>')
def set_session(username):
    session['username'] = username
    return f'Session set for {username}'

@app.route('/get_session')
def get_session():
    username = session.get('username')
    return f'Hello, {username}!' if username else 'No session data'
```
session 对象用于存取会话数据。
## 错误处理 Error Handing
Flask 允许你定义错误处理函数，当特定的错误发生时，这些函数会被调用。可以自定义错误页面或处理逻辑。
```python
@app.errorhandler(404)
def page_not_found(e):
    return 'Page not found', 404

@app.errorhandler(500)
def internal_server_error(e):
    return 'Internal server error', 500
```