### GO的net/http标准库

一个好的框架通常是快速构建可扩展和健壮的 Web 应用的最好的方法，但是理解那些隐藏在框架之下的底层概念和基础设施也是非常重要的，只要对底层的实现原理有了正确的认识，我们就可以清晰的了解到那些个性化的约定和模式是如何形成的，从而避免陷阱、理清思路、不再盲目使用模式；

对Go语言来说，隐藏在框架之下的通常是 net/http 和 html/template 这两个标准库，我们会分别讲解这两个标准库；

net/http 可以分为客户端服务器端两部分，库中的结构体和函数有些只支持客户端和服务端，也有一些同时支持客户端和服务端，比如：
![test](/net/static/http-req.png  "test")
##### 总结：
- Client、Response、Header、Request和Cookie 对客户端进行支持
- Server、ServerMux、Handler/HandleFunc、ResponseWriter、Header、Request和Cookie 对服务端进行支持

#### 使用Go构建服务端
通过Go构建的服务端数据处理流程：
![test](/net/static/server.png  "test")

#### Go Web服务器的构建
Go语言创建一个服务器的步骤非常简单，只要调用 http.ListenAndServe() 函数，并传入网络地址以及负责处理请求的处理器(handler)作为参数即可， 如果网络地址参数为空字符串，那么服务器默认就使用80端口，如果handler参数为nil，那么服务器将使用默认的多路复用器DefaultServeMux，比如：
```go
func handler(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "Hello World")
}
func main() {
	//http.HandleFunc("/", handler)
	http.ListenAndServe("", nil)//最简单的http server
}
```

#### HTTPS
```go
func handler(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "Hello World")
}

func main() {
	http.HandleFunc("/hello", handler)
	err := http.ListenAndServeTLS(":443", "cert.pem", "key.pem", nil)
	if err != nil {
		log.Fatal("ListenAndServe: ", err)
	}
}
```
用户除了可以通过 ListenAndServe 的参数对服务端进行配置之外， 还可以通过Server结构体对服务器进行更详细的配置，其中包括读写超时设置等；
```go
func main() {
	server := http.Server{
		Addr:    "",
		Handler: nil,
	}
	server.ListenAndServe()
}
```


#### 处理器和处理器函数
前面的内容里，我们只启动了服务，由于我们没有注册任何处理器，所以服务器端的多路复用器在收到请求之后找不到任何处理器来处理请求， 因此只能返回 404，为了让服务器能够产生实际的行为， 我们必须手动编写处理器。

什么是处理器呢？
一个处理器就是一个拥有ServeHTTP方法的接口， 这个ServeHTTP方法需要接收两个参数：
第一个是ResponseWriter接口，第二个是指向Request结构的指针；也就是说，任何接口只要拥有一个ServeHTTP方法，并且该方法带有以下声明，那么它就是一个处理器，比如：
```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```
考虑问题：
为什么ListenAndServe的默认处理器是 DefaultServeMux?
因为 DefaultServeMux 也是 ServeMux 的一个实例， 而 ServeMux 也实现了 ServeHTTP 这个接口， 所以 DefaultServeMux 不仅是一个多路复用器，而且是一个处理器，它是一个特殊的处理器， 它要做的就是根据请求的URL将请求重定向到不同的处理器。 
现在我们要自定义一个处理器，替代默认的多路复用器， 就可以让服务器正常的对客户端进行响应了，比如：
```go
type myhandler struct{}
//ServeHTTP xx
func (h *myhandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello World")
}
func main() {
	handler := myhandler{}
	server := http.Server{
		Addr:    ":8000",
		Handler: &handler,
	}
	err := server.ListenAndServe()
	fmt.Println(err)
}
```
测试发现任何URL都可以得到相同的响应，因为替代了默认的处理器后，服务器不会通过URL匹配来将请求路由至不同的处理器，而是使用同一个处理器处理所有请求。这也是我们需要在Web应用中使用多路复用器的原因， 对某些特殊用途的服务器来说， 使用一个处理器也许就够了， 但是大多数情况下，需要能够根据不通的URL请求返回不通的响应。

#### 使用多个处理器
为解决上面问题，现在使用多个处理器去处理不同的URL，为了做到这一点，我们不再用Server结构的handler字段中指定处理器， 而是让服务器使用默认的多路复用器 DefaultServeMux， 然后通过http.Handle函数将处理器绑定至 DefaultServeMux ，需要注意的是， 虽然Handle来自于 http包，但它实际上是 ServeMux 结构的方法，调用它们等同于调用 DefaultServeMux 的某个方法。比如说调用 http.Handle，就是在调用 DefaultServeMux 的Handle.

```go
type helloHandler struct{}
type worldHandler struct{}

//ServeHTTP xx
func (h *helloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello")
}

//ServeHTTP xx
func (h *worldHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "World")
}

func main() {
	helloHandler := helloHandler{}
	worldHandler := worldHandler{}
	server := http.Server{
		Addr: ":8000",
	}
	http.Handle("/hello", &helloHandler)
	http.Handle("/world", &worldHandler)
	server.ListenAndServe()
}
```
#### 处理器函数
上一节大家掌握了什么是处理器，但什么是处理器函数呢？
处理器函数实际上是与处理器拥有相同行为的函数，这些函数与ServerHTTP 方法拥有相同的声明， 比如改造上面多处理器的例子：
```go

func hello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello")
}
func world(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "World")
}

func main() {
	server := http.Server{
		Addr: ":8000",
	}
	http.HandleFunc("/hello", hello)
	http.HandleFunc("/world", world)
	server.ListenAndServe()
}
```
虽然处理器函数能够完成和处理器一样的工作， 并且使用处理器函数的代码更整洁，但是处理器函数并不能完全替代处理器。某些情况下， 代码可能已经包含了某个接口或者某种类型， 这时我们只需要为它们加ServeHTTP 方法就可以将它们转变为处理器了，并且这种转变也有助于构建出更为模块化的WEB 应用。

#### 串联多个处理器和处理器函数

```go
func log(h http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		name := runtime.FuncForPC(reflect.ValueOf(h).Pointer()).Name()
		fmt.Println("Handler function called - ", name)
		h(w, r)
	}
}
func main() {
	server := http.Server{
		Addr: ":8000",
	}
	http.HandleFunc("/hello", log(hello))
	//http.HandleFunc("/hello", hello)
	server.ListenAndServe()
}
```
log 函数接收 HandlerFunc 参数，并返回另一个 HandlerFunc ，hello 函数就是一个HandlerFunc类型的函数，所以log(hello) 实际上就是将hello函数发送至log函数内，log 函数的返回值是一个匿名函数， 因为这个匿名函数 接收ResponseWriter和Request指针作为参数，所以它实际上也是一个 HandlerFunc ，匿名函数内部， 首先会获取被传入的 HandlerFunc 名字，然后再调用这个 HandlerFunc；

就像堆积木一样，既然可以串联起两个函数，那么自然可以串联多个函数，串联多个函数可以让程序执行更多的动作，这种做法称为管道处理(pipeline processing)

```go
func protect(h http.HandlerFunc) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		name := runtime.FuncForPC(reflect.ValueOf(h).Pointer()).Name()
		fmt.Println("protect function called - ", name) //验证用户身份
		h(w, r)
	}
}
func main() {
	server := http.Server{
		Addr: ":8000",
	}
	http.HandleFunc("/hello", protect(log(hello)))
	server.ListenAndServe()
}
```
串联处理器和串联处理器函数的方式是很像的，比如：
```go
type helloHandler struct{}

func (h *helloHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello")
}

func log(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		fmt.Printf("Handler called - %T", h)
		h.ServeHTTP(w, r)
	})
}
func protect(h http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		name := runtime.FuncForPC(reflect.ValueOf(h).Pointer()).Name()
		fmt.Println("protect function called - ", name)
		h.ServeHTTP(w, r)
	})
}
func main() {
	hello := helloHandler{}
	server := http.Server{
		Addr: ":8000",
	}
	http.Handle("/hello", protect(log(&hello)))
	server.ListenAndServe()
}
```
差别如下：
protect 和 log 不再返回匿名函数， 而是返回一个Handler,程序执行也调用处理器的 ServeHTTP 方法了，还有一个差别是程序现在绑定的是处理器，而不是处理器函数； 

#### ServeMux 和 DefaultServeMux
ServeMux 是一个 HTTP 请求的多路复用器，它负责接收 HTTP 请求，并根据请求中的URL 将请求重定向到正确的处理器。
![test](/net/static/serve-mux.png  "test")

DefaultServeMux 实际上是 ServeMux 的一个实例， 并且所有引入net/http标准库的程序都可以使用这个实例，当用户没有设置Server结构指定的处理器时， 服务器就是使用 DefaultServeMux 作为ServeMux 的默认实例。

那么可以创建一个mux实例，兼顾上面的优点，比如：
```go
func index(writer http.ResponseWriter, request *http.Request) {
	fmt.Println(w, "main page")
}
func main() {
	mux := http.NewServeMux()
	// starting up the server
	mux.HandleFunc("/", index)
	server := &http.Server{
		Addr:           config.Address,
		Handler:        mux,
		ReadTimeout:    time.Duration(config.ReadTimeout * int64(time.Second)),
		WriteTimeout:   time.Duration(config.WriteTimeout * int64(time.Second)),
		MaxHeaderBytes: 1 << 20,
	}
	server.ListenAndServe()
}
```

思考： 如果浏览器访问的是/random 或者 /hello/gopher， 那么服务器又回返回什么呢？
这个问题的答案和绑定URL的方式有关，比如：
如果绑定URL为"/"， 那么匹配不成功的URL将会根据URL的层级进行下降，并最终落在根URL之上。所以当访问服务器端URL "/random"时， 服务器无法找到对应该URL的处理器， 所以会把这个URL交给根URL的处理器处理。
那么处理器是如何处理 /hello/gopher的呢？
实际原因是因为 程序绑定 helloHandler 的时候用的URL是/hello，而不是/hello/。
如果绑定的URL 不是以"/" 结尾， 那么它会与完全的URL进行匹配；
如果绑定的URL 是以"/"结尾，那么即使请求的URL只有前缀部分与绑定的URL相同， ServeMUX也会认定这两个URL是匹配的；
也就是说， 如果与helloHandler处理器绑定的URL是/hello/而不是/hello， 当浏览器请求 /hello/gopher 的时候，服务器在找不到对一个的处理器时， 就会退而求其次，开始寻找与/hello/匹配的处理器，最终找到 helloHandler 处理器.

#### 使用其他多路复用器
DefaultServeMux 缺点：
不支持正则路由
只支持路径匹配，不支持按照 Method，header，host等信息匹配，所以也就没法实现RESTful架构
举例说明:httprouter