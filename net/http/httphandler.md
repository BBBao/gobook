### HTTP 处理请求
前面了解了 处理器、处理器函数以及多路复用器，在学会了如何接收请求并将请求转发给相应的处理器之后， 我们将进一步了解如何处理请求，并把响应回传给客户端。
#### 请求和响应
HTTP 请求和响应报文格式:
```
请求行或者响应行
零个或多个首部
一个空行
一个可选的报文体
```
比如一个GET 请求：
```
GET /hello HTTP/1.1
HOST: 127.0.0.1
USER-AGENT: go
(empty line)
body
```
Go 语言的net/http库提供了一系列用于表示HTTP报文的结构， 为了学习如何使用这个库处理请求和发送响应， 我们需对这些结构有所了解。

#### Request 结构
Request 表示由一个客户端发送的HTTP请求报文，包含如下重要的组成部分：
```
    URL字段
    Header字段
    Body 字段
    Form字段、PostForm字段和MultipartForm字段
```
#### 请求行
URL 字段用于表示请求行中包含的URL信息，代码如下：
```go
type URL struct {
	Scheme     string
	Opaque     string    // encoded opaque data
	User       *Userinfo // username and password information
	Host       string    // host or host:port
	Path       string
	RawPath    string // encoded path hint (Go 1.5 and later only; see EscapedPath method)
	ForceQuery bool   // append a query ('?') even if RawQuery is empty
	RawQuery   string // encoded query values, without '?'
	Fragment   string // fragment for references, without '#'
}
```
在web开发的时候，经常会让客户端通过URL查询参数向服务器传递信息， 这些信息就被记录在URL结构体得 RawQuery 字段中，举个例子：
http://www.example.com/post?id=123&pid=456 
那么 RawQuery 字段的内容就是 id=123&pid=456 ，虽然通过该字段可以解析出请求参数， 但是我们直接使用 Request 结构的Form字段来获取这些键值信息更为方便。

#### 请求首部
请求和响应的首部都是使用 Header 类型来描述的，这个类型使用一个映射来表示HTTP首部中多个键值对，比如：
```go
// A Header represents the key-value pairs in an HTTP header.
type Header map[string][]string
```
Header 类型有4种基本方法，这些方法可以根据给定的键执行添加、删除、获取和设置值等操作。
比如：
```go
func headers(w http.ResponseWriter, r *http.Request) {
	h := r.Header
	fmt.Fprintln(w, h)
}
func main() {
	http.HandleFunc("/headers", headers)
}
```
上面代码会把请求首部信息打印出来， 如果只想获取某个特定的首部字段，则可以这样：
```go
{
	h := r.Header["Accept-Encoding"]
	//r.Header.Get("Accept-Encoding")
}
```
#### 请求体
请求体是由 Request 结构体得 Body 字段表示的， 这个字段是一个io.ReadCloser 接口， 即包含了 Reader 接口，也包含了 Closer 接口， 其中Reader接口含有 Read 方法， 这个方法接收一个字节切片为输入， 并在执行之后返回被读取的内容，以及一个可选的错误。 而Closer接口拥有Close方法，举例说明：
```go
func bodyRecv(w http.ResponseWriter, r *http.Request) {
	len := r.ContentLength
	body := make([]byte, len)
	r.Body.Read(body)
	fmt.Fprintln(w, string(body))
}
```
首先通过 ContentLength 获取主题数据长度， 然后创建一个字节数组， 再通过调用Read 方法将数据写入到字节数组中；
使用curl测试;
```
curl -id "name=abc&pass=abc" 127.0.0.1:8080/body 
```

#### Form 字段
通过调用Request结构体提供的方法，用户可以将URL,BODY 或者两者以上的数据提取到该结构体的 Form、PostForm和MultipartForm 等字段中。
```go
func process(w http.ResponseWriter, r *http.Request) {
	r.ParseForm()
	fmt.Fprintln(w, r.Form)
}
func main() {
	http.HandleFunc("/process", process)
	http.ListenAndServe(":8080", nil)
}
curl  http://localhost:8080/process\?a\=b\&c\=ee
curl  http://localhost:8080/process\?a\=b\&c\=ee -d "a=smile"
```
#### PostForm 字段
针对上面的代码，如果c只会出现在URL或者表单中的键来说， 执行 r.Form["c"]来说将返回表单值或者URL值，而对于a这种同时出现在URL和表单中的话，执行r.Form["a"]将返回一个同时包含该键的表单值和URL值得切片，并且表单值总是排在URL值得前面。
这个时候时候如果用户只想获取表单中的值，可以调用 PostForm 再来试试，而且 PostForm 只支持 application/x-www-form-urlencoded作为内容类型；如果我们改客户端为mutipart/form-data作为内容类型呢？
```bash
$ curl -H "Content-Type: mutipart/form-data" http://localhost:8080/process\?a\=b\&c\=ee -d "a=smile"
```
思考一下应用MultipartForm字段;
```go
r.ParseMultipartForm(1024)
r.MultipartForm
```
除了上面提到的获取表单中键值对的方法，还有 FormValue 和 PostFormValue 方法，允许直接访问与给定键相关联的值，就像访问Form字段中的键值对一样， 区别在于， FormValue 方法在需要时，会自动调用 ParseForm 或 ParseMultipartForm 方法，PostFormValue 方法只会返回表单中的键值对信息，而不会返回URL键值对。

#### 文件
```go

func fileRecv(w http.ResponseWriter, r *http.Request) {
	r.ParseMultipartForm(1024)
	fileHeader := r.MultipartForm.File["upload"][0]
	file, err := fileHeader.Open()
	if err == nil {
		data, err := ioutil.ReadAll(file)
		if err == nil {
			fmt.Fprintln(w, string(data))
		}
	}
}
```
测试：
```bash
$ curl --form upload=@test.awk http://localhost:8080/process
```
和 FormValue、PostFormValue 一样，net/http 提供了一个FormFile 方法，可以快速获取上传的文件, FormFile 将返回给定键的第一个值， 所以在客户端只上传一个文件的时候，使用它很方便：
```go
func fileRecv(w http.ResponseWriter, r *http.Request) {
	file, _, err := r.FormFile("upload")
	if err == nil {
		data, err := ioutil.ReadAll(file)
		if err == nil {
			fmt.Fprintln(w, string(data))
		}
	}
}
```

#### 处理带有 JSON主体的POST 请求
当客户端请求时， 设置了 "Content-Type: application/json" 的时候， Go 语言的ParseForm 方法不会对其进行语法解析，它并不接受application/json这种编码，所以使用这一编码发送POST请求的用户，自然也无法通过 ParseForm 方法获取任何数据；

#### ResponseWriter
ResponseWriter 是一个接口， 通过这个接口创建 HTTP 响应， 实际上在创建响应时会用到 http.response 结构，但是该结构是非导出的，所以用户只能通过 ResponseWriter 接口来使用这个结构，而不是直接操作它；
ResponseWriter 接口提供了三个方法：
```go
type ResponseWriter interface {
    Header() Header
    Write([]byte) (int, error)
    WriteHeader(int)
}
```
Write 方法负责将数组中的内容写入HTTP响应体中，如果没有为首部设置响应的内容类型时， 那么响应的内容类型将通过检测被写入的前 512 字节决定，比如：
```go
func writeExample(w http.ResponseWriter, r *http.Request) {
	str := `
		<html>
		<head>
			<title> Go programming </title>
		</head>
			<body> Hello World </body>
		</html>
	`
	w.Write([]byte(str))
}

func main() {
	http.HandleFunc("/writer", writeExample)
	http.ListenAndServe(":8080", nil)
}
```
```
HTTP/1.1 200 OK
Date: Wed, 11 Jul 2018 23:19:59 GMT
Content-Length: 105
Content-Type: text/html; charset=utf-8
```
测试发现，内容类型被自动识别成 text/html ,状态码 设置成了 200;

WriteHeader 方法用于设置 http 响应状态码，调用这个方法之后， 用户可以继续对 ResponseWriter 进行写入，但是不能对响应首部有任何修改，如果在调用
Write 方法之前没有调用 WriteHeader方法，则状态码被默认设置成 200；
WriteHeader 在返回错误状态码时特别有用： 假如你定一个了API， 但是尚未对其编写具体的实现，那么当客户度访问这个API 时，你希望给其返回 501 Not Implemented，客户端就知道这个接口还没有实现，比如：

```go
func writeHeaderExample(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(501)
	fmt.Fprintln(w, "No such service")
}

func main() {
	http.HandleFunc("/writeHeaderExample", writeHeaderExample)
	http.ListenAndServe(":8080", nil)
}
```
测试结果如下：
```
HTTP/1.1 501 Not Implemented
Date: Thu, 12 Jul 2018 01:15:22 GMT
Content-Length: 16
Content-Type: text/plain; charset=utf-8

No such service
```

Header 方法可以取得一个由首部组成的映射
```go
 // A Header represents the key-value pairs in an HTTP header.
type Header map[string][]string
```
通过这个映射就可以修改首部， 修改后的首部随着响应一同发送至客户端。
```go
func headerExample(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Location", "/writer")
	w.WriteHeader(302)
}
func main() {
	http.HandleFunc("/writer", writeExample)
	http.HandleFunc("/headerExample", headerExample)
	http.ListenAndServe(":8080", nil)
}
```
测试结果如下：
```
HTTP/1.1 301 Moved Permanently
Location: /writer
Date: Thu, 12 Jul 2018 01:26:42 GMT
Content-Length: 0
Content-Type: text/plain; charset=utf-8
```
最后，让我们看一下如何通过 ResponseWriter 给客户端返回 json，比如：
```go
func jsonExample(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	post := struct {
		User string
		Age  string
	}{
		"wang",
		"22",
	}
	js, _ := json.Marshal(&post)
	w.Write(js)
}

func main() {
	http.HandleFunc("/jsonExample", jsonExample)
	http.ListenAndServe(":8080", nil)
}
```
测试结果如下：
```
curl  localhost:8080/jsonExample -i
HTTP/1.1 200 OK
Content-Type: application/json
Date: Thu, 12 Jul 2018 01:45:49 GMT
Content-Length: 26

{"User":"wang","Age":"22"}
```

#### Cookies
cookie 是一种存储在客户端，体积较小的信息，这些信息最初都是由服务器通过 HTTP 响应报文发送的。每当客户端向服务器发送一次请求，cookie 都会随着请求一并发送至服务器端。cookie的本意是要客服 HTTP的无状态性， 虽然cookie不是完成这一目标的唯一方法，但它是最常用也最流行的方法之一；

cookie 在Go 语言里用 Cookie 结构体表示，定义如下：
```go
type Cookie struct {
	Name  string
	Value string

	Path       string    // optional
	Domain     string    // optional
	Expires    time.Time // optional
	RawExpires string    // for reading cookies only

	// MaxAge=0 means no 'Max-Age' attribute specified.
	// MaxAge<0 means delete cookie now, equivalently 'Max-Age: 0'
	// MaxAge>0 means Max-Age attribute present and given in seconds
	MaxAge   int
	Secure   bool
	HttpOnly bool
	Raw      string
	Unparsed []string // Raw text of unparsed attribute-value pairs
}
```
Expires:
没设置 Expires 的cookie 称为会话 cookie，或者临时cookie ，这种cookie 在浏览器关闭时， 自动被移除；反之，称为持久 cookie，直到指定的过期时间来临或者手动清除cookie为止；

MaxAge: 
用于说明 cookie 被浏览器创建出来之后，能够存活多少秒
出现这两种设置方式，和浏览器各不相同的cookie实现机制有关，和Go语言本身设计没关系， 虽然HTTP 1/1废弃了 Expires， 推荐使用 MaxAge，但几乎所有浏览器都支持 Expires， 而且 IE6-8 都不支持 MaxAge， 所以， 为了让 cookie 能在浏览器中正常运作， 可以同时使用 Expires 和 MaxAge；

举例将cookie 发送至浏览器：
```go
func setCookie(w http.ResponseWriter, r *http.Request) {
	c1 := http.Cookie{
		Name:     "first_cookie",
		Value:    "Go Web",
		HttpOnly: true,
	}
	c2 := http.Cookie{
		Name:     "second_cookie",
		Value:    "reboot",
		HttpOnly: true,
	}
	http.SetCookie(w, &c1)
	http.SetCookie(w, &c2)
	// w.Header().Set("Set-Cookie", c1.String())
	// w.Header().Add("Set-Cookie", c2.String())
}

func getCookie(w http.ResponseWriter, r *http.Request) {
	c := r.Header["Cookie"]
	fmt.Fprintln(w, c)
}
func main() {
	http.HandleFunc("/setCookie", setCookie)
	http.HandleFunc("/getCookie", getCookie)
	http.ListenAndServe(":8080", nil)
}
```
也可更方便的获取指定Cookie；
```go
func getCookie(w http.ResponseWriter, r *http.Request) {
	c := r.Cookies()
	c1, err := r.Cookie("first_cookie")
	if err != nil {
		fmt.Fprintln(w, "Can not find first_cookie")
	}
	fmt.Fprintln(w, c1)
	fmt.Fprintln(w, c)
}
```

使用cookie实现闪现消息：
```go
func setMessage(w http.ResponseWriter, r *http.Request) {
	msg := []byte("hello world")
	c := http.Cookie{
		Name:  "flash",
		Value: base64.URLEncoding.EncodeToString(msg),
	}
	http.SetCookie(w, &c)
}

func showMeg(w http.ResponseWriter, r *http.Request) {
	c, err := r.Cookie("flash")
	if err != nil {
		fmt.Fprintln(w, "No message found")
	} else {
		rc := http.Cookie{
			Name:    "flash",
			MaxAge:  -1,
			Expires: time.Unix(1, 0),
		}
		http.SetCookie(w, &rc)
		value, _ := base64.URLEncoding.DecodeString(c.Value)
		fmt.Fprintln(w, string(value))
	}
}
```
