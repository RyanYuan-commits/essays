在 Flask 中，表单处理是构建 Web 应用时一个常见的需求。
处理表单数据涉及到接收、验证和处理用户提交的表单。Flask 提供了基本的表单处理功能，但通常结合 Flask-WTF 扩展来简化表单操作和验证。
## 基本表单处理
Flask 提供了直接处理表单数据的方式，使用 request 对象来获取提交的数据。
### 创建 HTML 表单
```python
<!DOCTYPE html>
<html>
<head>
    <title>Form Example</title>
</head>
<body>
    <form action="/submit" method="post">
        <label for="name">Name:</label>
        <input type="text" id="name" name="name">
        <br>
        <label for="email">Email:</label>
        <input type="email" id="email" name="email">
        <br>
        <input type="submit" value="Submit">
    </form>
</body>
</html>
```
- `action="/submit"`：表单数据提交到 /submit 路径。
- `method="post"`：使用 POST 方法提交数据。
### 处理表单数据
```python
from flask import Flask, render_template, request

app = Flask(__name__)

@app.route('/')
def form():
    return render_template('form.html')

@app.route('/submit', methods=['POST'])
def submit():
    name = request.form.get('name')
    email = request.form.get('email')
    return f'Name: {name}, Email: {email}'

if __name__ == '__main__':
    app.run(debug=True)
```
- `request.form.get('name') 和 request.form.get('email')`：获取提交的表单数据。
