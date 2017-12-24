# tcp

我们以一个简单的Echo服务器实现为例，说明一下如何用Go语言构建TCP服务器和客户端，比如：


- TCP服务器

```go
相关包：
"io"
"net"
相关函数：
func Listen(net, laddr string) (Listener, error)
func (l *TCPListener) Accept() (Conn, error)
func (c *IPConn) RemoteAddr() Addr
func (l *TCPListener) Close() error
举例：
package main
import (
	"io"
	"log"
	"net"
)
var host = "0.0.0.0:2000"
func main() {
	l, err := net.Listen("tcp", host)
	if err != nil {
		log.Fatal(err)
	}
	defer l.Close()

	for {
		// Wait for a connection.
		conn, err := l.Accept()
		if err != nil {
			log.Fatal(err)
		}

		go func(c net.Conn) {
			log.Printf("Received message %s -> %s \n", c.RemoteAddr(), c.LocalAddr())
			io.Copy(c, c)
			c.Close()   // Shut down the connection.
		}(conn)
	}
}
说明：
1. 创建TCP服务器关键的一步就是监听本地一个端口号，通过IP地址加端口号，对外发布服务，
    Go语言应用net.Listen函数创建SOCKET并监听端口,host变量为IP地址加端口号，其中
    IP地址可以忽略":2000"，代表在所有接口IP上都监听该端口号，端口号不能省略。
2. 设置关闭Listener
3. 应用Accept函数，等待客户端的连接，如果没有客户端连接则一直阻塞
4. 如果有新客户端连接，则创建一个goroutine去处理业务逻辑，支持并发连接
5. 在goroutine内部实现业务逻辑，将客户端发送的数据直接返回给客户端
6. 关闭该客户端连接
```

- TCP客户端

```go
相关包：
"io"
"net"
相关函数：
func DialIP(netProto string, laddr, raddr *IPAddr) (*IPConn, error)
举例：
package main

import (
	"bufio"
	"net"
	"strconv"
	"sync"
	"log"
)

var host = "localhost:2000"

func tcpWrite(conn net.Conn, wg *sync.WaitGroup) {
	defer wg.Done()
	for i := 0; i < 10; i++ {
		_, e := conn.Write([]byte("echo " + strconv.Itoa(i) + "\r\n"))
		if e != nil {
			log.Println("Error to send message because of ", e.Error())
			break
		}
	}
}

func tcpRead(conn net.Conn, wg *sync.WaitGroup) {
	defer wg.Done()
	reader := bufio.NewReader(conn)
	for i := 1; i <= 10; i++ {
		line, err := reader.ReadString(byte('\n'))
		if err != nil {
			log.Print("Error to read message because of ", err)
			return
		}
		log.Print(line)
	}
}

func init() {
	log.SetFlags(log.Llongfile | log.LstdFlags)
}

func main() {
	conn, err := net.Dial("tcp", host)
	if err != nil {
		log.Fatal("Error connecting:", err)
	}
	defer conn.Close()
	log.Println("Connecting to " + host)
	var wg sync.WaitGroup
	wg.Add(2)
	go tcpWrite(conn, &wg)  //写逻辑
	go tcpRead(conn, &wg)   //读逻辑
	wg.Wait()
}

说明：
1. 为了方便debug，定位问题，在初始化init()函数中，设置了log输出接口状态，比如文件+行号
2. 应用net.Dial()函数连接服务器
3. 设置连接关闭
4. 分别构建tcp读写业务逻辑
5. 等待完成tcp读写业务逻辑，关闭连接
```
