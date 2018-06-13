### http client 介绍

http 包提供了 HTTP client和server的相关功能实现
比如：
```golang
resp, err := http.Get("http://example.com/")
resp, err := http.Post("http://example.com/upload", "image/jpeg", &buf)
resp, err := http.PostForm("http://example.com/form",
	url.Values{"key": {"Value"}, "id": {"123"}})
```

#### 应用http客户端的几种方式：
- 方式一：
http.get/http.post...

尝试一些更复杂的请求：
- 方式二：
生成client，之后用client.get/post...
场景：如果需要设置HTTP请求头部信息、重定向策略、超时设置等，比如：
```go
client := &http.Client{
    CheckRedirect: redirectPolicyFunc,
}
resp, err := client.Get("http://example.com")
// ...
```

- 方式三：
使用http.Newrequest， 先生成http.client -> 再生成 http.request -> 之后client.Do(request)提交请求
```go
client := &http.Client{
    CheckRedirect: redirectPolicyFunc,
}
req, err := http.NewRequest("GET", "http://example.com", nil)
// ...
req.Header.Add("If-None-Match", `W/"wyzzy"`)
resp, err := client.Do(req)
// ...
```

如果客户端读取服务器端响应体后，必须执行关闭响应体操作，比如：
```go
resp, err := http.Get("http://example.com/")
if err != nil {
	// handle error
}
defer resp.Body.Close()
body, err := ioutil.ReadAll(resp.Body)
// ...
```

练习域名有效性：

#### http服务端创建
通过 http.ListenAndServe 创建一个简单的http服务器:
```go

```

如果想设置一些控制信息，实现复杂一点的服务器，可以应用如下方式：
```golang
s := &http.Server{
    Addr:           ":8080",
	Handler:        myHandler,
	ReadTimeout:    10 * time.Second,
	WriteTimeout:   10 * time.Second,
	MaxHeaderBytes: 1 << 10,
}
s.ListenAndServe()
```

#### 快速实现一个文件服务器
```go
func newFileServer() {
	http.Handle("/static", http.StripPrefix("/static", http.FileServer(http.Dir("./"))))
	panic(http.ListenAndServe(":9000", nil))
}
```

作业：
验证证书有效性
主机监控插件
