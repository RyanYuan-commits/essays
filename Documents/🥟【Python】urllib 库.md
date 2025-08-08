urllib 是 Python 标准库中的一个模块，它提供了一些用于处理 URL 的功能，主要用于打开和读取 URL 的内容。
在 Python 3 中，它被拆分成几个子模块：
- urllib.request 用于打开和读取 URL
- urllib.parse 用于解析 URL
- urllib.error 用于处理请求时的异常
- urllib.robotparser 用于解析 robots.txt 文件
## urllib.request
```python
from urllib.request import urlopen, Request
import urllib.parse

# 发送一个简单的 GET 请求
response = urlopen('https://www.example.com')
print(response.status)
print(response.read().decode())

# 发送一个带有请求头的 GET 请求
headers = {'User-Agent': 'Mozilla/5.0'}
request = Request('https://www.example.com', headers=headers)
response = urlopen(request)
print(response.read().decode())

# 发送一个 POST 请求
data = urllib.parse.urlencode({'key': 'value'}).encode()
request = Request('https://www.example.com', data=data, method='POST')
response = urlopen(request)
print(response.read().decode())
```
- `from urllib.request import urlopen, Request`：从 `urllib.request` 导入 `urlopen` 函数和 `Request` 类。
- `urlopen('https://www.example.com')`：发送一个简单的 GET 请求到 `https://www.example.com` 并获取响应。
- `response.status`：获取响应的状态码。
- `response.read().decode()`：读取响应内容并解码为字符串。
- `headers = {'User-Agent': 'Mozilla/5.0'}`：定义请求头，模拟浏览器访问。
- `Request('https://www.example.com', headers=headers)`：创建一个 `Request` 对象，包含请求头信息。
- `data = urllib.parse.urlencode({'key': 'value'}).encode()`：将字典数据编码为字节流，用于 POST 请求。
- `Request('https://www.example.com', data=data, method='POST')`：创建一个 POST 请求对象。
## urllib.parse
这个子模块用于解析和构建 URL，包括 URL 的编码、解码、拆分、合并等操作。
```python
import urllib.parse

# 编码 URL 中的参数
encoded = urllib.parse.urlencode({'key': 'value'})
print(encoded)  

# 解码 URL 编码的字符串
decoded = urllib.parse.unquote(encoded)
print(decoded)  

# 解析 URL
parsed = urllib.parse.urlparse('https://www.example.com/path?key=value')
print(parsed.scheme)  
print(parsed.netloc)  
print(parsed.path)   
print(parsed.query)  
```
- `urllib.parse.urlencode({'key': 'value'})`：将字典参数编码为 URL 编码的字符串，例如 `key=value` 会被编码为 `key%3Dvalue`。
- `urllib.parse.unquote(encoded)`：将 URL 编码的字符串解码为原始字符串。
- `urllib.parse.urlparse('https://www.example.com/path?key=value')`：解析 URL 并将其拆分为多个部分，包括协议（`scheme`）、域名（`netloc`）、路径（`path`）、查询参数（`query`）等。
## urllib.error
这个子模块包含一些异常类，用于处理 `urllib.request` 中的异常。
```python
from urllib.request import urlopen
from urllib.error import HTTPError, URLError

try:
    response = urlopen('https://www.example.com/404')
except HTTPError as e:
    print(f'HTTP Error: {e.code}')
except URLError as e:
    print(f'URL Error: {e.reason}')
```
- `from urllib.request import urlopen`：导入 `urlopen` 函数。
- `from urllib.error import HTTPError, URLError`：导入 `HTTPError` 和 `URLError` 异常类。
- `try-except` 块用于捕获可能发生的 `HTTPError`（HTTP 错误，如 404 错误）和 `URLError`（URL 错误，如网络问题）。
## urllib.robotparser
```python
import urllib.robotparser

rp = urllib.robotparser.RobotFileParser()
rp.set_url('https://www.example.com/robots.txt')
rp.read()
can_fetch = rp.can_fetch('*', 'https://www.example.com/somepage')
print(can_fetch)
```
- `import urllib.robotparser`：导入 `RobotFileParser` 类。
- `rp = urllib.robotparser.RobotFileParser()`：创建一个 `RobotFileParser` 对象。
- `rp.set_url('https://www.example.com/robots.txt')`：设置要解析的 `robots.txt` 文件的 URL。
- `rp.read()`：读取并解析 `robots.txt` 文件。
- `rp.can_fetch('*', 'https://www.example.com/somepage')`：检查是否允许使用通配符 `*` 访问 `https://www.example.com/somepage`。