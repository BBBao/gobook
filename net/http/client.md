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
func main() {
	tr := &http.Transport{
		TLSClientConfig: &tls.Config{InsecureSkipVerify: true},
	}
	client := &http.Client{Transport: tr}
	r, err := client.Get("https://localhost/hello")
	if err != nil {
		fmt.Println(err)
	}
	defer r.Body.Close()
	data, _ := ioutil.ReadAll(r.Body)

	fmt.Println(string(data))
}
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

#### http服务端创建
通过 http.ListenAndServe 创建一个简单的http服务器:
```go
import "net/http"

func main() {
	//http.Handle("/", http.HandlerFunc(index))
    //http.HandleFunc("/text", text)
	http.Handle("/", http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		w.Write([]byte("Hello"))
	}))
	http.ListenAndServe(":8001", nil)
}

import (
    "fmt"
    "net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hi there, I love %s!", r.URL.Path[1:])
}

func main() {
    http.HandleFunc("/", handler)
    http.ListenAndServe(":8080", nil)
}
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
如何自定义实现 myHandler
```go
	mux := http.NewServeMux()
	mux.HandleFunc("/", http.HandlerFunc(func(w http.ResponseWriter, req *http.Request) {
		w.Write([]byte("Hello"))
	}))

	s := &http.Server{
		Addr:           ":8001",
		Handler:        mux,
		ReadTimeout:    10 * time.Second,
		WriteTimeout:   10 * time.Second,
		MaxHeaderBytes: 1 << 10,
	}
	s.ListenAndServe()

	mux := http.NewServeMux()
    mux.Handle("/", http.HandlerFunc(index))
    mux.HandleFunc("/text", text)
    http.ListenAndServe(":8000", mux)
```



#### 快速实现一个文件服务器
```go
func newFileServer() {
	http.Handle("/static", http.StripPrefix("/static", http.FileServer(http.Dir("./"))))
	panic(http.ListenAndServe(":9000", nil))
}
```

DefaultServeMux缺陷
不支持正则路由
只支持路径匹配，不支持按照 Method，header，host等信息匹配，所以也就没法实现RESTful架构
gorilla/mux

作业：
预习gorilla/mux
验证证书有效性
主机存活监控
http://localhost:6060/doc/articles/wiki/