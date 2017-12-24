# udp

我们以一个简单的时间服务器实现为例，说明一下如何用Go语言构建UDP服务器和客户端，比如：

- UDP服务器：

```go
相关包：
"io"
"net"
相关函数：
func ResolveUDPAddr(net, addr string) (*UDPAddr, error)
func ListenUDP(net string, laddr *UDPAddr) (*UDPConn, error)
func (c *UDPConn) WriteToUDP(b []byte, addr *UDPAddr) (int, error)
func (c *UDPConn) Close() error
举例：
package main
import (
    "log"
    "net"
    "time"
    "encoding/binary"
)
var host = ":2000"
func init(){
    log.SetFlags(log.Llongfile|log.LstdFlags)
}

func udpWorker(conn *net.UDPConn) {
    data := make([]byte, 1024)
	_, remoteAddr, err := conn.ReadFromUDP(data)
	if err != nil {
		log.Println("failed to read UDP msg because of ", err.Error())
		return
	}
	ts := time.Now().Unix()
	var b = make([]byte, 8)
	binary.BigEndian.PutUint64(b, uint64(ts))
	conn.WriteToUDP(b, remoteAddr)
}
func main() {
    addr, err := net.ResolveUDPAddr("udp", host)
    if err != nil {
        log.Fatal(err)
    }   
    conn, err := net.ListenUDP("udp", addr)
    if err != nil {
        log.Fatal(err)
    }   
    defer conn.Close()

    for {
        udpWorker(conn)
    }   
}
说明：
1. 为了方便debug，定位问题，在初始化init()函数中，设置了log输出接口状态，比如文件+行号
2. 构建udp服务器时，先用net.ResolveUDPAddr函数将addr转换成UDPAddr，再应用net.ListenUDP
    函数创建udp服务
3. 设置连接关闭
4. 实现业务逻辑udpWorker，获取当前时间戳返回给客户端
```

- UDP客户端

```go
相关包：
"net"
"log"
"time"
"encoding/binary"
相关函数：
func ResolveUDPAddr(net, addr string) (*UDPAddr, error)
func DialUDP(net string, laddr, raddr *UDPAddr) (*UDPConn, error)
func (c *UDPConn) Read(b []byte) (int, error)
func (c *UDPConn) Write(b []byte) (int, error)
func (c *UDPConn) Close() error
举例：
package main
import (
	"net"
	"log"
	"time"
	"encoding/binary"
)

var host = "localhost:2000"

func init() {
	log.SetFlags(log.Llongfile | log.LstdFlags)
}

func main() {
	addr, err := net.ResolveUDPAddr("udp", host)
	if err != nil {
		log.Fatal(err)
	}
	conn, err := net.DialUDP("udp", nil, addr)
	if err != nil {
		log.Fatal(err)
	}
	defer conn.Close()

	_, err = conn.Write([]byte(""))
	if err != nil {
		log.Fatal(err)
	}
	data := make([]byte, 8)
	_, err = conn.Read(data)
	if err != nil {
		log.Fatal(err)
	}
	ts := binary.BigEndian.Uint64(data)
	log.Println(time.Unix(int64(ts),0).String())
}
说明：
1. 为了方便debug，定位问题，在初始化init()函数中，设置了log输出接口状态，比如文件+行号
2. 应用net.ResolveUDPAddr函数将服务器端的addr转换成UDPAddr，然后再应用net.DialUDP函数
    连接UDP服务器
3. 应用conn.Write向服务端发送数据
4. 应用conn.Read接收服务端发送的数据
```
