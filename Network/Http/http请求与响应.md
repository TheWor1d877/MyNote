## HTTP请求
- 请求行：
一行格式是`<方法> <请求URL> <HTTP版本>`
- 请求头
由多个 键: 值 对组成，提供关于请求的附加信息。
常见头部示例：
```http
Host: www.example.com
User-Agent: Mozilla/5.0 ...
Accept: text/html,application/xhtml+xml
Accept-Language: zh-CN,zh;q=0.9
Connection: keep-alive
Content-Type: application/json   （通常用于 POST/PUT）
```
- 请求体
可选
用于携带客户端发送给服务器的数据（如表单数据、JSON 等）。
  GET 请求通常没有请求体；POST、PUT 等方法常有。
```http
{
  "username": "alice",
  "password": "123456"
}
```

## HTTP相应
- 状态行
200成功
3开头重定向
4开头前端
5开头后端
- 相应头
同样是 键: 值 对，提供响应的元信息。
常见头部示例：
```http
Content-Type: text/html; charset=utf-8
Content-Length: 1234
Server: nginx/1.18.0
Set-Cookie: sessionid=abc123; Path=/
Cache-Control: max-age=3600
```
- 相应体
服务器返回的实际数据内容，如 HTML 页面、JSON 数据、图片等。