模板是包含占位符的 HTML 文件。Flask 使用 Jinja2 模板引擎来处理模板渲染。
模板渲染允许你将动态内容插入到 HTML 页面中，使得应用能够生成动态的网页内容。
## 创建模板
模板文件通常放在项目的 templates 文件夹中。
Flask 会自动从这个文件夹中查找模板文件。
**创建模板文件**：在项目目录下创建 templates 文件夹，并在其中创建一个 HTML 文件，如 index.html。

templates/index.html 文件代码：
```html
<!DOCTYPE html>
<html>
<head>
    <title>Welcome</title>
</head>
<body>
    <h1>{{ title }}</h1>
    <p>Hello, {{ name }}!</p>
</body>
</html>
```
{{ title }} 和 {{ name }} 是模板占位符，将在渲染时被替换成实际的值。

app.py 文件代码：
```python
from flask import Flask, render_template

app = Flask(__name__)

@app.route('/')
def home():
    return render_template('index.html', title='Welcome Page', name='John Doe')

if __name__ == '__main__':
    app.run(debug=True)
```
`render_template('index.html', title='Welcome Page', name='John Doe')`：渲染 index.html 模板，并将 title 和 name 变量传递给模板。
## 模板继承
模板集成允许你创建一个基础模板，然后再其他模板中继承和拓展这个基础模板，避免创建重复的 HTML 代码。
### 创建基础模板
在 templates 文件夹中创建一个基础模板 base.html。
```python
<!DOCTYPE html>
<html>
<head>
    <title>{% block title %}My Website{% endblock %}</title>
</head>
<body>
    <header>
        <h1>My Website</h1>
    </header>
    <main>
        {% block content %}{% endblock %}
    </main>
    <footer>
        <p>Footer content</p>
    </footer>
</body>
</html>
```
`{% block title %}{% endblock %}` 和 `{% block content %}{% endblock %}` 是定义的可替换区域。
### 创建子模板
在 templates 文件夹中创建一个子模板 index.html，继承 base.html。
```python
{% extends "base.html" %}

{% block title %}Home Page{% endblock %}

{% block content %}
<h2>Welcome to the Home Page!</h2>
<p>Content goes here.</p>
{% endblock %}
```
- `{% extends "base.html" %}`：继承基础模板。
- `{% block title %} 和 {% block content %}`：重写基础模板中的块内容。
### 控制结构
Jinja2 提供了多种控制结构，用于在模板中实现条件逻辑和循环。
### 条件语句
`{% if user %}`：检查 user 变量是否存在，如果存在，则显示欢迎消息，否则显示登录提示。
```python
{% if user %}
    <p>Welcome, {{ user }}!</p>
{% else %}
    <p>Please log in.</p>
{% endif %}
```
### 循环语句
`{% for item in items %}`：遍历 items 列表，并为每个项生成一个 `<li>` 元素。
```python
<ul>
{% for item in items %}
    <li>{{ item }}</li>
{% endfor %}
</ul>
```
### 过滤器
过滤器用于在模板中格式化和处理变量数据。
```python
<p>{{ name|capitalize }}</p>
<p>{{ price|round(2) }}</p>
```
- `{{ name|capitalize }}`：将 name 变量的值首字母大写。
- `{{ price|round(2) }}`：将 price 变量的值四舍五入到小数点后两位。
