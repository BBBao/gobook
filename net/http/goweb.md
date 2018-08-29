
先看一个HTTP Server的例子：
```go
package main

import (
	"fmt"
	"net/http"
)

func handler(writer http.ResponseWriter, request *http.Request) {
	fmt.Fprintf(writer, "Hello World, %s!", request.URL.Path[1:])
}

func main() {
	http.HandleFunc("/", handler)
	http.ListenAndServe(":8080", nil)
}
```

我们可以看到它使用了 net/http 包中的 HandleFunc() 函数以及 ListenAndServe() 函数，下面我们来分别解析一下这两个函数:

HandleFunc 定义在 net/http/server.go 文件中:
```go
// HandleFunc registers the handler function for the given pattern
// in the DefaultServeMux.
// The documentation for ServeMux explains how patterns are matched.
func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

可看出该函数调用了 DefaultServeMux 中的函数，接下来看看 DefaultServeMux.HandleFunc 函数的实现：
```go
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
	hosts bool // whether any patterns contain hostnames
}
// NewServeMux allocates and returns a new ServeMux.
func NewServeMux() *ServeMux { return new(ServeMux) }
// DefaultServeMux is the default ServeMux used by Serve.
var DefaultServeMux = &defaultServeMux
var defaultServeMux ServeMux
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	mux.Handle(pattern, HandlerFunc(handler))
}
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()
	...
	mux.m[pattern] = muxEntry{explicit: true, h: handler, pattern: pattern}
	...
}
```

最终通过 Handle方法进行路由注册 mux.m[pattern] = muxEntry{explicit: true, h: handler, pattern: pattern}
注册过程是个写 map 的动作，之后怎么匹配这个注册的handler呢？

Handler 处理的入口就是 serverHandler{c.server}.ServeHTTP(w, w.req)，最终到 HandleFunc的执行，往下看。

ListenAndServe定义在 net/http/server.go中，
```go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
func (srv *Server) ListenAndServe() error {
	...
	return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
func (srv *Server) Serve(l net.Listener) error {
	...
	for {
		rw, e := l.Accept()
		if e != nil {

		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve(ctx) // ?
	}
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	...
	handler.ServeHTTP(rw, req)
}
```
可看出该函数调用了 server 中的函数，接下来看看 server.ListenAndServe 函数的实现，该函数中又调用到了 srv.Serve 函数，接下来我们追溯到这个函数来看看，server 为每一个请求建立一个连接，同时进行逻辑的处理。这样就完成了 ListenAndServe 的实现和执行。

#### 总结：
http.HandleFunc: 
调用http.HandleFunc->调用DefaultServerMux.HandleFunc->调用DefaultServerMux Handle

http.ListenAndServe: 
实例化Server->调用Server.ListenAndServe()->为每一个请求建立一个连接,同时进行逻辑的处理，进行go c.serve()，读取每个请求的内容->调用handler.ServeHttp->根据request选择handler->mux.handler(r).ServeHTTP(w,r)，最终选择上面注册过的handler进行逻辑处理；

如何解析http 请求参数：
```go
    r.ParseForm()
	fmt.Println("path", r.URL.Path)
	fmt.Println("path", r.Host)
	fmt.Println("method", r.Method)
	fmt.Println(r.Form)
	fmt.Println(r.Form["id"])
	for k, v := range r.Form {
		fmt.Println("key:", k)
		fmt.Println("value:", strings.Join(v, ""))
	}
	body, _ := ioutil.ReadAll(r.Body)
	fmt.Println(string(body))
	fmt.Fprintf(writer, "Hello World, %s!", r.URL.Path[1:])
```


```go

type Retmsg struct {
	Msg string `json:"s"`
}

type Result struct {
	Succ string `json:"succ"`
	Msg  Retmsg `json:"msg"`
}

func handler(writer http.ResponseWriter, r *http.Request) {
	r.ParseForm()
	fmt.Println("path", r.URL.Path)
	fmt.Println("path", r.Host)
	fmt.Println("method", r.Method)
	fmt.Println(r.Form)
	fmt.Println(r.Form["id"])
	for k, v := range r.Form {
		fmt.Println("key:", k)
		fmt.Println("value:", strings.Join(v, ""))
	}
	body, _ := ioutil.ReadAll(r.Body)
	var result = Result{}
	fmt.Println(string(body))
	json.Unmarshal(body, &result)
	fmt.Println(result)
	fmt.Fprintf(writer, "Hello World, %s!", r.URL.Path[1:])
}

#curl -H "Content-Type: application/json" http://localhost:8080/ss -d "{\"succ\":\"yes\",\"msg\":{\"s\":\"vvv\"}}"
```