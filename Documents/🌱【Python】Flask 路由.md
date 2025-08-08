Flask 路由是 Web 应用程序中将 URL 映射到 Python 函数的机制。
Flask 路由是 Flask 应用的核心部分，用于处理不同 URL 的请求，并将请求的处理委托给相应的视图函数。
## 定义路由
基本路由定义：
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return 'Welcome to the Home Page!'
```
- `@app.route('/)`：装饰器，用于定义路由，/ 标识根 URL。
- `def home()`：视图函数，当访问根 URL 的时候，返回 'Welcome to the Home Page!'
## 路由参数
路由可以包含动态部分，通过在路由中指定参数，可以将 URL 中的部分数据传递给视图函数。
```python
@app.route('/greet/<name>')
def greet(name):
    return f'Hello, {name}!'
```
## 路由规则
路由规则支持不同的参数和匹配规则
- **字符串（默认）：** 匹配任意字符串。
- **整数（`<int:name>`）：** 匹配整数值。
- **浮点数（`<float:value>`）：** 匹配浮点数值。
- **路径（`<path:name>`）：** 匹配任意字符，包括斜杠 `/`。
```python
@app.route('/user/<int:user_id>')
def user_profile(user_id):
    return f'User ID: {user_id}'

@app.route('/files/<path:filename>')
def serve_file(filename):
    return f'Serving file: {filename}'
```
## 请求方法
Flask 路由支持不同的 HTTP 请求方法，如 GET、POST、PUT、DELETE 等，可以通过 methods 参数指定允许的请求方法。
```python
@app.route('/submit', methods=['POST'])
def submit():
    return 'Form submitted!'
```
## 路由转换器
Flask 提供了一些内置的转换器，可以对 URL 中的参数进行特定类型的转换。
常用的转换器：int、float、path
```python
@app.route('/items/<int:item_id>/details')
def item_details(item_id):
    return f'Item details for item ID: {item_id}'
```
- `<int:item_id>`：将 URL 中的 `item_id` 转换为整数。
## 路由函数返回
视图函数可以返回多种类型的相应，字符串、HTML、JSON、Response 对象，
```python
from flask import jsonify, Response

@app.route('/json')
def json_response():
    data = {'key': 'value'}
    return jsonify(data)

@app.route('/custom')
def custom_response():
    response = Response('Custom response with headers', status=200)
    response.headers['X-Custom-Header'] = 'Value'
    return response
```
- `jsonify(data)`：将字典转换为 JSON 响应。
- `Response('Custom response with headers', status=200)`：创建自定义响应对象。
## 静态文件和模板
静态文件（如 CSS、JavaScript、图片）可以通过 static 路由访问。模板文件则通过 templates 文件夹组织，用于渲染 HTML 页面。
静态文件访问：
```html
<link rel="stylesheet" href="{{ url_for('static', filename='style.css') }}">
```
```python
from flask import render_template

@app.route('/hello/<name>')
def hello(name):
    return render_template('hello.html', name=name)
```
模板文件：
```python
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
## 路由优先级
Flask 安装定义的顺序匹配路由，第一个匹配成功的路由将被处理，**确保更具体的路由放在更一般的路由之前**。
```python
@app.route('/user/<int:user_id>')
def user_profile(user_id):
    return f'User ID: {user_id}'

@app.route('/user')
def user_list():
    return 'User List'
```