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

#### tcp 粘包问题：
例如我们和客户端约定数据交互格式是一个json格式的字符串：
{"Id":1,"Name":"golang","Message":"message"}
当客户端发送数据给服务端的时候，如果服务端没有及时接收，客户端又发送了一条数据上来，这时候服务端才进行接收的话就会收到两个连续的字符串，形如：
{"Id":1,"Name":"golang","Message":"message"}{"Id":1,"Name":"golang","Message":"message"}
如果接收缓冲区满了的话，那么也有可能接收到半截的json字符串，这样的话还怎么用json解码呢？比如：
server端：
```go
func handleConnection(conn net.Conn) {
	buffer := make([]byte, 1024)
	for {
		n, err := conn.Read(buffer)
		if err != nil {
			Log(conn.RemoteAddr().String(), " connection error: ", err)
			return
		}
		Log(conn.RemoteAddr().String(), "receive data length:", n)
		Log(conn.RemoteAddr().String(), "receive data:", buffer[:n])
		Log(conn.RemoteAddr().String(), "receive data string:", string(buffer[:n]))
	}
}
```
客户端：
```go
func sender(conn net.Conn) {
	for i := 0; i < 100; i++ {
		words := "{\"Id\":1,\"Name\":\"golang\",\"Message\":\"message\"}"
		conn.Write([]byte(words))
	}
}
go sender(conn)
```
#### 粘包处理
处理方法：
```
短连接
自定义协议处理封包和解包（发送数据长度）
```

```go
//通讯协议处理，主要处理封包和解包的过程
package protocol

import (
	"bytes"
	"encoding/binary"
)

const (
	// ConstHeader         = "www.51reboot.com"
	// ConstHeaderLength   = 16
	ConstSaveDataLength = 4
)

//封包
func Packet(message []byte) []byte {
	s := make([]byte, 0)
	//return append(append([]byte(ConstHeader), IntToBytes(len(message))...), message...)
	return append(append(s, IntToBytes(len(message))...), message...)
}

//解包
func Unpack(buffer []byte, readerChannel chan []byte) []byte {
	length := len(buffer)

	var i int
	for i = 0; i < length; i = i + 1 {
		//if length < i+ConstHeaderLength+ConstSaveDataLength {
		if length < i+ConstSaveDataLength {
			break
		}
		//if string(buffer[i:i+ConstHeaderLength]) == ConstHeader {
		messageLength := BytesToInt(buffer[i : i+ConstSaveDataLength])
		if length < i+ConstSaveDataLength+messageLength {
			break
		}
		data := buffer[i+ConstSaveDataLength : i+ConstSaveDataLength+messageLength]
		readerChannel <- data

		i += ConstSaveDataLength + messageLength - 1
		//}
		// if string(buffer[i:i+ConstHeaderLength]) == ConstHeader {
		// 	messageLength := BytesToInt(buffer[i+ConstHeaderLength : i+ConstHeaderLength+ConstSaveDataLength])
		// 	if length < i+ConstHeaderLength+ConstSaveDataLength+messageLength {
		// 		break
		// 	}
		// 	data := buffer[i+ConstHeaderLength+ConstSaveDataLength : i+ConstHeaderLength+ConstSaveDataLength+messageLength]
		// 	readerChannel <- data

		// 	i += ConstHeaderLength + ConstSaveDataLength + messageLength - 1
		// }
	}

	if i == length {
		return make([]byte, 0)
	}
	return buffer[i:]
}

//整形转换成字节
func IntToBytes(n int) []byte {
	x := int32(n)

	bytesBuffer := bytes.NewBuffer([]byte{})
	binary.Write(bytesBuffer, binary.BigEndian, x)
	return bytesBuffer.Bytes()
}

//字节转换成整形
func BytesToInt(b []byte) int {
	bytesBuffer := bytes.NewBuffer(b)

	var x int32
	binary.Read(bytesBuffer, binary.BigEndian, &x)

	return int(x)
}
```

server端：
```go
func handleConnection(conn net.Conn) {
	//声明一个临时缓冲区，用来存储被截断的数据
	tmpBuffer := make([]byte, 0)

	//声明一个管道用于接收解包的数据
	readerChannel := make(chan []byte, 16)
	go reader(readerChannel)

	buffer := make([]byte, 1024)
	for {
		n, err := conn.Read(buffer)
		if err != nil {
			Log(conn.RemoteAddr().String(), " connection error: ", err)
			return
		}

		tmpBuffer = protocol.Unpack(append(tmpBuffer, buffer[:n]...), readerChannel)
	}
}

func reader(readerChannel chan []byte) {
	for {
		select {
		case data := <-readerChannel:
			Log(string(data))
		}
	}
}
```
client端代码：
```go
func sender(conn net.Conn) {
	for i := 0; i < 1000; i++ {
		words := "{\"Id\":1,\"Name\":\"golang\",\"Message\":\"message\"}"
		conn.Write(protocol.Packet([]byte(words)))
	}
	fmt.Println("send over")
}
```

server 改造成队列模式，缓解爆发链接
```go
const MAX_QUEUE_SIZE = 5

type job struct {
	conn net.Conn
}

func (j *job) Do() {
	defer j.conn.Close()
	buffer := make([]byte, 1024)
	for {
		n, err := j.conn.Read(buffer)
		if err != nil {
			fmt.Println(j.conn.RemoteAddr().String(), " connection error: ", err)
			return
		}
		fmt.Println(j.conn.RemoteAddr().String(), "receive data string:", string(buffer[:n]))
	}
}

var queue = make(chan job, MAX_QUEUE_SIZE)
var quit = make(chan bool)

func main() {
	netListen, err := net.Listen("tcp", ":9988")
	if err != nil {
		panic(err)
	}
	go func() {
		for {
			select {
			case job := <-queue:
				time.Sleep(5 * time.Second)
				job.Do()
			case <-quit:
				return
			}
		}
	}()
	defer netListen.Close()
	fmt.Println("Waiting for clients")
	for {
		conn, err := netListen.Accept()
		if err != nil {
			continue
		}
		fmt.Println(conn.RemoteAddr().String(), " tcp connect success")
		j := job{
			conn: conn,
		}
		fmt.Println(cap(queue), "------", len(queue))
		if cap(queue) == len(queue) {
			fmt.Println("full ....")
		}
		queue <- j
	}
}

```
client：
```go

func sender3(conn net.Conn, i int) {
	defer conn.Close()
	words := fmt.Sprintf("{\"Id\":%d,\"Name\":\"golang\",\"Message\":\"message\"}", i)
	conn.Write([]byte(words))
}

func main() {
	server := "127.0.0.1:9988"
	tcpAddr, err := net.ResolveTCPAddr("tcp4", server)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
		os.Exit(1)
	}

	fmt.Println("connect success")
	for i := 0; i < 30; i++ {
		fmt.Println(i)
		conn, err := net.DialTCP("tcp", nil, tcpAddr)
		if err != nil {
			fmt.Fprintf(os.Stderr, "Fatal error: %s", err.Error())
			os.Exit(1)
		}
		go sender3(conn, i)
	}

	for {
		time.Sleep(10 * time.Second)
	}
}
```

server改造成二级队列模式：
```go
var (
	MaxWorker = 10
	MaxQueue  = 100
)

const MAX_QUEUE_SIZE = 5

type Job struct {
	conn net.Conn
}

var JobQueue = make(chan Job, MaxQueue)

var quit = make(chan bool)

// Worker represents the worker that executes the job
type Worker struct {
	ID         int
	WorkerPool chan chan Job
	JobChannel chan Job
	quit       chan bool
}

func NewWorker(workerPool chan chan Job, id int) Worker {
	return Worker{
		ID:         id,
		WorkerPool: workerPool,
		JobChannel: make(chan Job),
		quit:       make(chan bool)}
}

// Start method starts the run loop for the worker, listening for a quit channel in
// case we need to stop it
func (w Worker) Start() {
	go func() {
		for {
			// register the current worker into the worker queue.
			w.WorkerPool <- w.JobChannel
			fmt.Println("w.WorkerPool <- w.JobChannel", w.ID)
			select {
			case job := <-w.JobChannel:
				// we have received a work request.
				job.Do()
			case <-w.quit:
				// we have received a signal to stop
				return
			}
		}
	}()
}

// Stop signals the worker to stop listening for work requests.
func (w Worker) Stop() {
	go func() {
		w.quit <- true
	}()
}

//Do xxx
func (j *Job) Do() {
	defer j.conn.Close()
	buffer := make([]byte, 1024)
	for {
		n, err := j.conn.Read(buffer)
		if err != nil {
			fmt.Println(j.conn.RemoteAddr().String(), " connection error: ", err)
			return
		}
		fmt.Println(j.conn.RemoteAddr().String(), "receive data string:", string(buffer[:n]))
	}
}

type Dispatcher struct {
	// A pool of workers channels that are registered with the dispatcher
	WorkerPool chan chan Job
}

func NewDispatcher(maxWorkers int) *Dispatcher {
	pool := make(chan chan Job, maxWorkers)
	return &Dispatcher{WorkerPool: pool}
}

func (d *Dispatcher) Run() {
	// starting n number of workers
	for i := 0; i < MaxWorker; i++ {
		worker := NewWorker(d.WorkerPool, i)
		worker.Start()
	}

	go d.dispatch()
}

func (d *Dispatcher) dispatch() {
	for {
		select {
		case job := <-JobQueue:
			// a job request has been received
			go func(job Job) {
				// try to obtain a worker job channel that is available.
				// this will block until a worker is idle
				jobChannel := <-d.WorkerPool
				fmt.Println("jobChannel := <-d.WorkerPool")
				// dispatch the job to the worker job channel
				jobChannel <- job
			}(job)
		}
	}
}

func main() {
	netListen, err := net.Listen("tcp", ":9988")
	if err != nil {
		panic(err)
	}
	d := NewDispatcher(10)
	d.Run()
	defer netListen.Close()
	fmt.Println("Waiting for clients")
	for {
		conn, err := netListen.Accept()
		if err != nil {
			continue
		}
		fmt.Println(conn.RemoteAddr().String(), " tcp connect success")
		j := Job{
			conn: conn,
		}

		JobQueue <- j
	}
}
```
