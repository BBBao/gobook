# udp
用户数据报协议（User Datagram Protocol，缩写为UDP），又称用户数据报文协议，是一个简单的面向数据报(package-oriented)的传输层协议，正式规范为RFC 768。UDP只提供数据的不可靠传递，它一旦把应用程序发给网络层的数据发送出去，就不保留数据备份（所以UDP有时候也被认为是不可靠的数据报协议）。UDP在IP数据报的头部仅仅加入了复用和数据校验。

由于缺乏可靠性且属于非连接导向协议，UDP应用一般必须允许一定量的丢包、出错和复制粘贴。但有些应用，比如TFTP，如果需要则必须在应用层增加根本的可靠机制。但是绝大多数UDP应用都不需要可靠机制，甚至可能因为引入可靠机制而降低性能。流媒体（流技术）、即时多媒体游戏和IP电话（VoIP）一定就是典型的UDP应用。如果某个应用需要很高的可靠性，那么可以用传输控制协议（TCP协议）来代替UDP。

由于缺乏拥塞控制（congestion control），需要基于网络的机制来减少因失控和高速UDP流量负荷而导致的拥塞崩溃效应。换句话说，因为UDP发送者不能够检测拥塞，所以像使用包队列和丢弃技术的路由器这样的网络基本设备往往就成为降低UDP过大通信量的有效工具。数据报拥塞控制协议（DCCP）设计成通过在诸如流媒体类型的高速率UDP流中，增加主机拥塞控制，来减小这个潜在的问题。
典型网络上的众多使用UDP协议的关键应用一定程度上是相似的。这些应用包括域名系统（DNS）、简单网络管理协议（SNMP）、动态主机配置协议（DHCP）、路由信息协议（RIP）和某些影音流服务等等。

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
```
举例：
server：
```go
func main() {
	listener, err := net.ListenUDP("udp", &net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 9981})
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("Local: <%s> \n", listener.LocalAddr().String())
	data := make([]byte, 1024)
	for {
		n, remoteAddr, err := listener.ReadFromUDP(data)
		if err != nil {
			fmt.Printf("error during read: %s", err)
		}
		fmt.Printf("<%s> %s\n", remoteAddr, data[:n])
		_, err = listener.WriteToUDP([]byte("world"), remoteAddr)
		if err != nil {
			fmt.Printf(err.Error())
		}
	}
}
```
可以看到, Go UDP的处理类似TCP的处理，虽然不像TCP面向连接的方式ListenTCP和Accept的方式建立连接,但是它通过ListenUDP和ReadFromUDP可以接收各个客户端发送的数据报，并通过WriteToUDP写数据给特定的客户端.
client:
```go
func main() {
	sip := net.ParseIP("127.0.0.1")
	srcAddr := &net.UDPAddr{IP: net.IPv4zero, Port: 0}
	dstAddr := &net.UDPAddr{IP: sip, Port: 9981}
	conn, err := net.DialUDP("udp", srcAddr, dstAddr)
	if err != nil {
		fmt.Println(err)
	}
	defer conn.Close()
	conn.Write([]byte("hello"))
	fmt.Printf("<%s>\n", conn.RemoteAddr())
	<!-- data := make([]byte, 1024)
	n, remote, err := conn.ReadFromUDP(data)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("%d <%s> %s\n", n, remote, string(data)) -->
}
```
我们稍微修改一下client1.go,让它保持UDP Socket文件一直打开：
```go
b := make([]byte, 1)
os.Stdin.Read(b)
```
使用 netstat可以看到这个网络文件描述符(因为我在同一台机器上运行服务器，所以你会看到两条记录，一个是服务器打开的，一个是客户端打开的)。
```bash
# netstat -apunp | grep 9981
# lsof -i:9981
```

客户端修改为读写：
```go
func main() {
	ip := net.ParseIP("127.0.0.1")
	srcAddr := &net.UDPAddr{IP: net.IPv4zero, Port: 0}
	dstAddr := &net.UDPAddr{IP: ip, Port: 9981}
	conn, err := net.DialUDP("udp", srcAddr, dstAddr)
	if err != nil {
		fmt.Println(err)
	}
	defer conn.Close()
	conn.Write([]byte("hello"))
	data := make([]byte, 1024)
	n, err := conn.Read(data)
	fmt.Printf("read %s from <%s>\n", data[:n], conn.RemoteAddr())
}
```

### 等价的客户端和服务器
面这个是两个服务器通信的例子，互为客户端和服务器，在发送数据报的时候，我们可以将发送的一方称之为源地址，发送的目的地一方称之为目标地址。
```go
func read(conn *net.UDPConn) {
	for {
		data := make([]byte, 1024)
		n, remoteAddr, err := conn.ReadFromUDP(data)
		if err != nil {
			fmt.Printf("error during read: %s", err)
		}
		fmt.Printf("receive %s from <%s>\n", data[:n], remoteAddr)
	}
}

func main() {
	addr1 := &net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 9981}
	addr2 := &net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 9982}
	go func() {
		listener1, err := net.ListenUDP("udp", addr1)
		if err != nil {
			fmt.Println(err)
			return
		}
		go read(listener1)
		time.Sleep(5 * time.Second)
		listener1.WriteToUDP([]byte("ping to #2: "+addr2.String()), addr2)
	}()
	go func() {
		listener1, err := net.ListenUDP("udp", addr2)
		if err != nil {
			fmt.Println(err)
			return
		}
		go read(listener1)
		time.Sleep(5 * time.Second)
		listener1.WriteToUDP([]byte("ping to #1: "+addr1.String()), addr1)
	}()
	b := make([]byte, 1)
	os.Stdin.Read(b)
}
```
### Read和Write方法集的比较
前面的例子中客户端有时使用DialUDP建立数据报的源对象和目标对象(地址和端口), 它会创建UDP Socket文件描述符,然后调用内部的connect为这个文件描述符设置源地址和目标地址，这时Go将它称之为connected;尽管我们知道UDP是无连接的协议，Go这种叫法我想根源来自Unix/Linux的UDP的实现。这个方法返回 *UDPConn。
有的时候也可以通过 ListenUDP 返回的 \*UDPConn 直接往某个目标地址发送数据报，而不是通过DialUDP方式发送，区别在于两者返回的 *UDPConn 是不同的。前者是 connected，后者是 unconnected。

所以，必须清楚知道你的UDP是连接的(connected)还是未连接(unconnected)的，这样你才能正确的选择的读写方法，比如：
```go
如果*UDPConn是connected,读写方法是Read和Write。
如果*UDPConn是unconnected,读写方法是ReadFromUDP和WriteToUDP（以及ReadFrom和WriteTo)。
ReadFrom 和 WriteTo 是为了实现 PacketConn 接口而实现的方法，它们的实现基本上和ReadFromUDP和WriteToUDP一样，只不过地址换成了更通用的Addr,而不是具体化的UDPAddr。
```

几种注意情况：
```
因为unconnected的 *UDPConn 还没有目标地址，所以需要把目标地址当作参数传入到WriteToUDP的方法中，但是unconnected的*UDPConn可以调用Read方法吗？
答案是可以,但是在这种情况下，客户端的地址信息就被忽略了。
```
举例：
```go
func main() {
	listener, err := net.ListenUDP("udp", &net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 9981})
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("Local: <%s> \n", listener.LocalAddr().String())
	data := make([]byte, 1024)
	for {
		n, err := listener.Read(data)
		if err != nil {
			fmt.Printf("error during read: %s", err)
		}
		fmt.Printf("<%s>\n", data[:n])
	}
}
```
```
问题2
unconnected 的*UDPConn可以调用Write方法吗？
答案是不可以， 因为不知道目标地址。
```
举例：
```go
func main() {
	listener, err := net.ListenUDP("udp", &net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 9981})
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("Local: <%s> \n", listener.LocalAddr().String())
	_, err = listener.Write([]byte("hello"))
	if err != nil {
		fmt.Printf(err.Error())
	}
}
```
```
问题3
onnected 的*UDPConn 可以调用 WriteToUDP 方法吗？
答案是不可以， 因为目标地址已经设置。
即使是相同的目标地址也不可以。
```
举例：
```go
func main() {
	ip := net.ParseIP("127.0.0.1")
	srcAddr := &net.UDPAddr{IP: net.IPv4zero, Port: 0}
	dstAddr := &net.UDPAddr{IP: ip, Port: 9981}
	conn, err := net.DialUDP("udp", srcAddr, dstAddr)
	if err != nil {
		fmt.Println(err)
	}
	defer conn.Close()
	_, err = conn.WriteToUDP([]byte("hello"), dstAddr)
	if err != nil {
		fmt.Println(err)
	}
}
```
```
问题4
connected 的 *UDPConn 如果调用Closed以后可以调用WriteToUDP方法吗？
答案是不可以。
```
```go
 	conn.Close()
	_, err = conn.WriteToUDP([]byte("hello"), dstAddr)
	if err != nil {
		fmt.Println(err)
	}
```
```
问题5
connected的*UDPConn可以调用ReadFromUDP方法吗？
答案是可以,但是它的功能基本和Read一样，只能和它connected的对端通信。
```
举例：
```go
func main() {
	ip := net.ParseIP("127.0.0.1")
	srcAddr := &net.UDPAddr{IP: net.IPv4zero, Port: 0}
	dstAddr := &net.UDPAddr{IP: ip, Port: 9981}
	conn, err := net.DialUDP("udp", srcAddr, dstAddr)
	if err != nil {
		fmt.Println(err)
	}
	defer conn.Close()
	conn.Write([]byte("hello"))
	go func() {
		data := make([]byte, 1024)
		for {
			n, remoteAddr, err := conn.ReadFromUDP(data)
			if err != nil {
				fmt.Printf("error during read: %s", err)
			}
			fmt.Printf("<%s> %s\n", remoteAddr, data[:n])
		}
	}()
	b := make([]byte, 1)
	os.Stdin.Read(b)
}
```

```
问题六
*UDPConn还有一个通用的WriteMsgUDP(b, oob []byte, addr *UDPAddr)，同时支持connected和unconnected的UDPConn:
如果UDPConn还未连接，那么它会发送数据报给addr
如果UDPConn已连接，那么它会发送数据报给连接的对端，这种情况下addr应该为nil
```

#### 标准库多播编程
server:
```go
func main() {
	//如果第二参数为nil,它会使用系统指定多播接口，但是不推荐这样使用
	addr, err := net.ResolveUDPAddr("udp", "224.0.0.250:9981")
	if err != nil {
		fmt.Println(err)
	}
	listener, err := net.ListenMulticastUDP("udp", nil, addr)
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("Local: <%s> \n", listener.LocalAddr().String())
	data := make([]byte, 1024)
	for {
		n, remoteAddr, err := listener.ReadFromUDP(data)
		if err != nil {
			fmt.Printf("error during read: %s", err)
		}
		fmt.Printf("<%s> %s\n", remoteAddr, data[:n])
	}
}
```
同一个应用可以加入到多个组中，多个应用也可以加入到同一个组中。
多个UDP listener可以监听同样的端口，加入到同一个group中。
client:
```go
func main() {
	ip := net.ParseIP("224.0.0.250")
	srcAddr := &net.UDPAddr{IP: net.IPv4zero, Port: 0}
	dstAddr := &net.UDPAddr{IP: ip, Port: 9981}
	conn, err := net.DialUDP("udp", srcAddr, dstAddr)
	if err != nil {
		fmt.Println(err)
	}
	defer conn.Close()
	conn.Write([]byte("hello"))
	fmt.Printf("<%s>\n", conn.RemoteAddr())
}
```
广播的编程方式和多播的编程方式有所不同。简单说，广播意味着你吼一嗓子，局域网内的所有的机器都会收到。
```go
func main() {
	listener, err := net.ListenUDP("udp", &net.UDPAddr{IP: net.IPv4zero, Port: 9981})
	if err != nil {
		fmt.Println(err)
		return
	}
	fmt.Printf("Local: <%s> \n", listener.LocalAddr().String())
	data := make([]byte, 1024)
	for {
		n, remoteAddr, err := listener.ReadFromUDP(data)
		if err != nil {
			fmt.Printf("error during read: %s", err)
		}
		fmt.Printf("<%s> %s\n", remoteAddr, data[:n])
		_, err = listener.WriteToUDP([]byte("world"), remoteAddr)
		if err != nil {
			fmt.Printf(err.Error())
		}
	}
}
```
客户端代码有所不同，它不是通过DialUDP “连接” 广播地址，而是通过ListenUDP创建一个unconnected的 *UDPConn,然后通过WriteToUDP发送数据报，这和你脑海中的客户端不太一致：
```go
func main() {
	ip := net.ParseIP("192.168.10.255")
	srcAddr := &net.UDPAddr{IP: net.IPv4zero, Port: 0}
	dstAddr := &net.UDPAddr{IP: ip, Port: 9981}
	conn, err := net.ListenUDP("udp", srcAddr)
	if err != nil {
		fmt.Println(err)
	}
	n, err := conn.WriteToUDP([]byte("hello"), dstAddr)
	if err != nil {
		fmt.Println(err)
	}
	data := make([]byte, 1024)
	n, addr, err := conn.ReadFrom(data)
	if err != nil {
		fmt.Println(err)
	}
	fmt.Printf("read %s from <%s>\n", data[:n], addr.String())
	b := make([]byte, 1)
	os.Stdin.Read(b)
}
```
举例：
```go
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
```

说明：
```
1. 为了方便debug，定位问题，在初始化init()函数中，设置了log输出接口状态，比如文件+行号
2. 构建udp服务器时，先用 net.ResolveUDPAddr 函数将addr转换成UDPAddr，再应用net.ListenUDP
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
3. 应用 conn.Write 向服务端发送数据
4. 应用 conn.Read 接收服务端发送的数据
```


http://colobu.com/2016/10/19/Go-UDP-Programming/#%E7%AD%89%E4%BB%B7%E7%9A%84%E5%AE%A2%E6%88%B7%E7%AB%AF%E5%92%8C%E6%9C%8D%E5%8A%A1%E5%99%A8
总结：
```text
首先你得明白UDP中connect函数和bind函数作用.
在golang中的UDPConn分为connected和unconnected.
如果*UDPConn是connected,读写方法是Read和Write。
如果*UDPConn是unconnected,读写方法是ReadFromUDP和WriteToUDP（以及ReadFrom和WriteTo)。
DialUDP中的UDPConn为connected是不能调用WriteToUDP发送给某个地址.
ListenUDP中的UDPCon为unconnected,直接可以调用WriteToUDP发送给某个地址.
udp是无连接的，但是得指定接受方，dial只是为了抽象 socket，并不是建立了连接
```


改造 mytcp 加上 udpserver
改造成高并发时间服务