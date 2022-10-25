---
title: Python 编写 Mock 服务器
date: 2022-10-25 10:45:01
tags: Web
---

## 使用 `http.server` 库编写 Web 服务器

[`http.server`](https://docs.python.org/3/library/http.server.html) 提供了基类 `BaseHTTPRequestHandler` 供用户继承并重载 `do_GET`、`do_POST` 等无参数的处理方法，在这些方法内可以通过 `self` 的 `path`、`headers`、`rfile` 等属性获取请求的信息并通过 `send_response`、`send_headers`、`end_headers`、`wfile` 等属性发送响应。一个简单的 Hello world 服务器如下：

```python
from http.server import ThreadingHTTPServer, BaseHTTPRequestHandler
import json


class HelloHandler(BaseHTTPRequestHandler):

    def do_POST(self):
        assert self.headers['Content-Type'] == 'application/json'
        content = self.rfile.read(int(self.headers['Content-Length'])).decode()
        data = json.loads(content)
        res = f'hello {data["user"]}, love from {self.path}'.encode()
        self.send_response(200)
        self.send_header('Content-Type', 'text-plain')
        self.send_header('Content-Length', len(res))
        self.end_headers()
        self.wfile.write(res)


server = ThreadingHTTPServer(('0.0.0.0', 8080), HelloHandler)
server.serve_forever()
```

先根据 `self.headers` 的 Content-Type、Content-Length 等信息从 `self.rfile` 读取并解析内容，然后通过 `self.send_response` 向 `self.wfile` 写入响应行，通过 `self.send_headers` 写入头（包括 Content-Length、Content_Type 等信息），通过 `self.end_headers` 写入空行结束头，最后将内容直接写入 `self.wfile`。用户还可以通过 `self.send_response_only` 和 `self.send_error` 写入最简单的响应。

测试如下：

```python
import json
import requests

res = requests.post('http://localhost:8080/test/',
                    data=json.dumps({'user': 'vhqr'}),
                    headers={'Content-Type': 'application/json'})

assert res.status_code == 200

print(res.text)

# => hello vhqr, love from /test/
```

## Mock 服务器

Mock 服务器的功能（from [wikipedia](https://en.wikipedia.org/wiki/MockServer)）：

> MockServer is designed to simplify integration testing, by mocking HTTP and HTTPS system such as a web service or web site, and to decouple development teams, by allowing a team to develop against a service that is not complete or is unstable.

即 Mock 服务器通过模拟后端服务器发送响应方便前端测试，避免在开发过程中依赖后端的部署。

以一个简单的登陆 api 为例：

```javascript
import axios from "axios";

const baseURL = "http://localhost:8080/";

export const login = (username, password) => {
  return axios.post(baseURL + "login/", { username, password });
};

export const users = () => {
  return axios.get(baseURL + "users/");
};
```

我们可以为该 api 写一个简单的 Mock 服务器：

```python
class MockHandler(BaseHTTPRequestHandler):

    def do_GET(self):
        res = {'status': 400, 'data': None}
        if self.path == '/users/':
            res = {'status': 200, 'data': ['vhqr1', 'vhqr2']}
        # elif self.path == '...':
        res = json.dumps(res).encode()
        self.send_response(200)
        self.send_header('Content-Type', 'application/json')
        self.send_header('Content-Length', len(res))
        self.end_headers()
        self.wfile.write(res)

    def do_POST(self):
        res = {'status': 400, 'data': None}
        try:
            assert self.headers['Content-Type'] == 'application/json'
            content = self.rfile.read(int(
                self.headers['Content-Length'])).decode()
            data = json.loads(content)
            if self.path == '/login/':
                username = data['username']
                password = data['password']
                if username == 'vhqr' and password == '123':
                    res = {'status': 200, 'data': {'username': 'vhqr'}}
                else:
                    res = {'status': 400, 'data': {'username': '???'}}
            # elif self.path == '...':
        except:
            self.send_error(400)
        else:
            res = json.dumps(res).encode()
            self.send_response(200)
            self.send_header('Content-Type', 'application/json')
            self.send_header('Content-Length', len(res))
            self.end_headers()
            self.wfile.write(res)
```

但是，上面的 Mock 服务器在实际运行过程中遇到了 CORS 问题，相关 MDN 链接：[CORS 预检请求](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/CORS#%E9%A2%84%E6%A3%80%E8%AF%B7%E6%B1%82)、[Access-Control-Allow-Headers](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Access-Control-Allow-Headers)。

简而言之，CORS 是防止 api 被滥用的安全机制。当来自网站 A 的网页请求访问网站 B 时，浏览器会先发送预检请求（方法为 OPTIONS，路径不变，添加一些特殊的头），然后通过网站 B 响应的头（Access-Control-*）判断是否允许该访问请求。

为了避免 CORS，首先让 Mock 服务器响应 OPTIONS 请求（重写方法 do_OPTIONS），并为所有响应添加头：Access-Control-Allow-Origin: *，还是不行，再添加 Access-Control-Allo-Headers: *，可以。完整代码如下：

```python
class MockHandler(BaseHTTPRequestHandler):

    def do_OPTIONS(self):
        self.send_response(200)
        self.send_header('Access-Control-Allow-Origin', '*')
        self.send_header('Access-Control-Allow-Headers', '*')
        self.end_headers()

    def do_GET(self):
        res = {'status': 400, 'data': None}
        if self.path == '/users/':
            res = {'status': 200, 'data': ['vhqr1', 'vhqr2']}
        # elif self.path == '...':
        res = json.dumps(res).encode()
        self.send_response(200)
        self.send_header('Access-Control-Allow-Origin', '*')
        self.send_header('Access-Control-Allow-Headers', '*')
        self.send_header('Content-Type', 'application/json')
        self.send_header('Content-Length', len(res))
        self.end_headers()
        self.wfile.write(res)

    def do_POST(self):
        res = {'status': 400, 'data': None}
        try:
            assert self.headers['Content-Type'] == 'application/json'
            content = self.rfile.read(int(
                self.headers['Content-Length'])).decode()
            data = json.loads(content)
            if self.path == '/login/':
                username = data['username']
                password = data['password']
                if username == 'vhqr' and password == '123':
                    res = {'status': 200, 'data': {'username': 'vhqr'}}
                else:
                    res = {'status': 400, 'data': {'username': 'vhqr'}}
            # elif self.path == '...':
        except:
            self.send_error(400)
        else:
            res = json.dumps(res).encode()
            self.send_response(200)
            self.send_header('Access-Control-Allow-Origin', '*')
            self.send_header('Access-Control-Allow-Headers', '*')
            self.send_header('Content-Type', 'application/json')
            self.send_header('Content-Length', len(res))
            self.end_headers()
            self.wfile.write(res)
```
