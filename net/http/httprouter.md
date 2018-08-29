### httpRouter

```go
func Index(w http.ResponseWriter, r *http.Request, _ httprouter.Params) {
	fmt.Fprint(w, "Welcome!\n")
}

func Hello(w http.ResponseWriter, r *http.Request, ps httprouter.Params) {
	fmt.Fprintf(w, "hello, %s!\n", ps.ByName("name"))
}

func main() {
	router := httprouter.New()
	router.GET("/", Index)
	router.GET("/hello/:name", Hello)
	/*
	server := http.Server{
		Handler : router,
	}
	log.Fatal(server.ListenAndServe(":8080", router))
	*/
	log.Fatal(http.ListenAndServe(":8080", router))
}
```
基本变化：
router := httprouter.New() 通过New创建一个多路复用器
不再使用HandleFunc绑定处理器函数，而是直接把处理器函数与给定的HTTP方法进行绑定，比如:
router.GET("/hello/:name", Hello)
并且被绑定的URL中包含了命名参数，这些命名参数会被URL中具体的值替代
处理器函数也由以前的两个参数变成了三个，其中第三个参数Params就包含了之前提到的命名参数，该命名参数可以通过 ByName 函数获取
不再使用 DefaultServeMux , 而是通过 HttpRouter 实例进行替换

#### HTTP/2
如果使用HTTPS模式启动服务器，那么服务器将默认使用HTTP/2，

如何检测服务器是否运行在HTTP/2模式下呢？ 可以使用curl工具对服务器进行检查。
```sh
$ curl -I --http2 --insecure http://localhost:8080
```