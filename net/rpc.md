# rpc

Go语言的net/rpc包实现了最基本的RPC调用，它默认通过HTTP协议传输gob数据实现远程调用。
服务端收到客户端请求后，会反序列化客户端传来的gob数据，获取要调用的方法名称，并通过反射来
调用服务端对外提供的方法，这个方法具有两个参数一个返回值，两个参数分别为客户端的请求内容以及服务端要返回的
数据体指针; 返回值则是一个error对象。下面对RPC服务的构建，以及RPC客户端同步和异步调用方式进行代码举例说明。

- 服务器端

```go
相关包：
"log"
"net"
"net/rpc"
"net/rpc/jsonrpc"
相关函数：
func NewServer() *Server
func (server *Server) Register(rcvr interface{}) error
func (server *Server) ServeCodec(codec ServerCodec)
func NewServerCodec(conn io.ReadWriteCloser) rpc.ServerCodec
举例：
import (
    "log"
    "net"
    "net/rpc"
    "net/rpc/jsonrpc"
)

//定义一个server端的rpc处理器
type ServerHandler struct {}


//RpcObj 数据通信用的结构体
type RpcObj struct {
	Id   int `json:"id"`
	Name string `json:"name"`
}

//定义server端对外提供的rpc方法
func (serverHandler ServerHandler) GetName(id int, returnObj *RpcObj) error {
    log.Println("server\t-", "recive GetName call, id:", id)
    returnObj.Id = id
    returnObj.Name = "gopher"
    return nil
}

func startRPCServer() {
    server := rpc.NewServer()
    serverHandler := new(ServerHandler)
    server.Register(serverHandler)

    l, e := net.Listen("tcp4",":8888")
    if e != nil {
        log.Fatal("listen error:", e)
    }
    defer l.Close()

    for {
        conn, err := l.Accept()
        if err != nil {
            log.Println("Accept error:", err)
            continue
        }
        go server.ServeCodec(jsonrpc.NewServerCodec(conn))
    }
}
func init() {
    log.SetFlags(log.LstdFlags|log.Llongfile)
}

func main() {
    startRPCServer()
}
说明：
1. rpc处理器ServerHandler不需要任何字段，只要符合net/rpc server端约定的方法即可，该约定
    的方法必须具有两个参数和一个error类型的返回值
2. 定义数据通信用的结构体RpcObj
3. 定义服务端对外提供的方法调用GetName
4. 定义RPC服务startRPCServer
    4.1 定义rpc服务对象server
    4.2 应用Register函数注册rpc处理器
    4.5 启动服务，监听本地端口号，进入等待客户端连接的阻塞状态
```

- 客户端

```go
相关包：
"log"
"net"
"time"
"net/rpc/jsonrpc"
相关函数：
func DialTimeout(network, address string, timeout time.Duration) (Conn, error)
func NewClient(conn io.ReadWriteCloser) *rpc.Client
func (client *Client) Call(serviceMethod string, args interface{}, reply interface{}) error
func (client *Client) Go(serviceMethod string, args interface{}, reply interface{}, done chan *Call) *Call
举例：
import (
	"log"
	"time"
	"net"
	"net/rpc/jsonrpc"
)

type RpcObj struct {
	Id   int `json:"id"`
	Name string `json:"name"`
}
//同步调用方法
func callRpcBySynchronous() {
	client, err := net.DialTimeout("tcp", "localhost:8888", time.Second * 30) // 30秒超时时间
	if err != nil {
		log.Fatal("client\t-", err.Error())
	}
	defer client.Close()

	clientRpc := jsonrpc.NewClient(client)
	var rpcObj RpcObj
	log.Println("client\t：", "call GetName method")
	clientRpc.Call("ServerHandler.GetName", 1, &rpcObj)
	log.Println("client\t：", "recive remote return", rpcObj)
}

//异步调用方法
func callRpcByASynchronous() {
    client, err := net.DialTimeout("tcp", "localhost:8888", time.Second*30) // 30秒超时时间
    if err != nil {
        log.Fatal("client\t-", err.Error())

    }   
    defer client.Close()

    clientRpc := jsonrpc.NewClient(client)
    var wg sync.WaitGroup
    for i:= 0;i<10; i++{
        divCall := clientRpc.Go("ServerHandler.GetName", i, &RpcObj{}, nil)
        wg.Add(1)
        go func(num int) {
            defer wg.Done()
            replyCall := <-divCall.Done
            log.Println("client\t-", "recive remote return", replyCall.Reply)
        }(i)
    }   
    wg.Wait()
}

func init() {
	log.SetFlags(log.LstdFlags|log.Llongfile)
}

func main() {
	callRpcBySynchronous()
    callRpcByASynchronous()
}
说明：
1. 应用net.DialTimeout()函数与server端建立网络连接,并设置关闭
2. 定义数据通信用的结构体RpcObj，与server端定义相同
3. 应用jsonrpc.NewClient建立rpc通信通道
4. 应用clientRpc.Call调用server端对外提供的方法
5. 应用clientRpc.Go调用server端对外提供的方法
```
