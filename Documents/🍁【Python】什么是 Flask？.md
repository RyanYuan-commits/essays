>[!tips] 什么是 Flask？
一句话：Flask 是一个用 Python 编写的轻量级 Web **应用框架**。

## Flask 的特点
- **轻量级和简洁**：Flask 是一个微框架，提供了最基本的功能，不强制使用任何特定的工具或库。它的核心是简单而灵活的，允许开发者根据需要添加功能。
- **灵活性**：Flask 提供了基本的框架结构，但没有强制性的项目布局或组件，开发者可以根据自己的需求自定义。
- **可扩展性**：Flask 的设计允许你通过插件和扩展来添加功能。许多常见的功能，如表单处理、数据库交互和用户认证，都可以通过社区提供的扩展来实现。
- **内置开发服务器**：Flask 内置了一个开发服务器，方便在本地进行调试和测试。
- **RESTful 支持**：Flask 支持 RESTful API 的开发，适合构建现代的 Web 服务和应用程序。
## Flask 安装
```python
# 使用 pip 安装 Flask
pip install Flask
# 检查是否安装成功
pip show Flask
```
安装成功后，显示结果类似如下：
```python
Name: Flask
Version: 3.0.3
Summary: A simple framework for building complex web applications.
Home-page: 
Author: 
Author-email: 
License: 
Location: /Users/bytedance/Documents/code/py_code/flask_demo/venv/lib/python3.8/site-packages
Requires: blinker, click, importlib-metadata, itsdangerous, Jinja2, Werkzeug
Required-by: 
```
## Hello World
快速启动第一个 Flask Server：
```python
from flask import Flask

app = Flask(__name__)
@app.route('/')
def hello_world():
	return 'Hello World'
if __name__ == '__main__':
	app.run(debug=True)
```