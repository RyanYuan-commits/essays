视图函数是 Flask 应用中的核心部分，它负责处理请求并生成响应。
视图函数与路由紧密结合，通过路由将 URL 映射到具体的视图函数。
以下是对 Flask 视图函数的详细说明，包括如何定义、使用请求数据、返回响应、以及如何处理错误等。
## 定义视图函数
视图函数是一个普通的 Python 函数，它接受请求并返回相应。视图函数通常与路由配合使用，通过装饰器将 URL 视图映射到视图函数。
```python
from flask import Flask

app = Flask(__name__)

@app.route('/')
def home():
    return 'Hello, World!'
```
- `@app.route('/')`：将根 URL `/` 映射到 `home` 视图函数。
- `def home()`：视图函数，返回字符串 `'Hello, World!'` 作为响应。
## 接收请求数据
视图函数可以接受不同类型的请求数据，包括 URL 参数、表单数据、查询参数等。
### 获取 URL 参数
```python
@app.route('/greet/<name>')
def greet(name):
    return f'Hello, {name}!'
```
### 获取表单数据
```python
@app.route('/search')
def search():
    query = request.args.get('query')
    return f'Search results for: {query}'
```
## 返回响应
视图函数可以范湖哦多种类型的响应，包括字符串、HTML、JSON 或者自定义响应对象。
### 返回字符串
```python
@app.route('/message')
def message():
    return 'This is a simple message.'
```
### 返回 HTML 模板
```python
from flask import render_template

@app.route('/hello/<name>')
def hello(name):
    return render_template('hello.html', name=name)
```
render_template('hello.html', name=name)：渲染 HTML 模板 hello.html，并将 name 变量传递给模板。
### 返回 JSON 数据
```python
from flask import jsonify

@app.route('/api/data')
def api_data():
    data = {'key': 'value'}
    return jsonify(data)
```
jsonify(data)：将 Python 字典转换为 JSON 响应。
### 返回自定义响应对象
```python
from flask import Response

@app.route('/custom')
def custom_response():
    response = Response('Custom response with headers', status=200)
    response.headers['X-Custom-Header'] = 'Value'
    return response
```
Response('Custom response with headers', status=200)：创建自定义响应对象，并设置响应头。
## 处理请求和响应
视图函数可以访问请求对象，并根据请求数据生成响应。可以使用 request 对象来获取请求的信息，使用 make_response 来创建自定义响应。
```python
from flask import request

@app.route('/info')
def info():
    user_agent = request.headers.get('User-Agent')
    return f'Your user agent is {user_agent}'
```
- `request.headers.get('User-Agent')`：获取请求头中的 User-Agent 信息。
使用 make_response：
```python
from flask import make_response

@app.route('/header')
def custom_header():
    response = make_response('Response with custom header')
    response.headers['X-Custom-Header'] = 'Value'
    return response
```
- `make_response('Response with custom header')`：创建响应对象并设置自定义头信息。
`make_response` 的参数顺序规则：
![[make_response 的参数规则.png|700]]
## 处理错误
可以在视图函数中吃力异常或者错误，或者通过 Flask 提供的错误处理机制来处理应用中的错误：
### 从视图函数中处理错误
```python
@app.route('/divide/<int:x>/<int:y>')
def divide(x, y):
    try:
        result = x / y
        return f'Result: {result}'
    except ZeroDivisionError:
        return 'Error: Division by zero', 400
```
- 使用 try-except 语句处理除零错误，并自定义错误消息和状态码。
### 全局处理错误
```python
@app.errorhandler(404)
def not_found(error):
    return 'Page not found', 404
```
- @app.errorhandler(404)：定义处理 404 错误的函数。
## 视图函数的装饰器
除了 @app.route，Flask 还支持其他装饰器，用于实现更复杂的功能。
示例：
- **`@app.before_request`**：在每个请求处理之前运行的函数。
- **`@app.after_request`**：在每个请求处理之后运行的函数。
- **`@app.teardown_request`**：在请求结束后运行的函数，用于清理工作。
示例装饰器使用：
```python
@app.before_request
def before_request():
    print('Before request')

@app.after_request
def after_request(response):
    print('After request')
    return response

@app.teardown_request
def teardown_request(exception):
    print('Teardown request')
```