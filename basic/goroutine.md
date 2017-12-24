# 并发
- 并发(Concurrency) VS 并行(Parallelism)

```text
并发: 逻辑上将相互独立的执行过程综合到一起的编程技术,重点在于组合。
并行: 物理上同时执行(通常是相关的)计算任务的编程技术，重点在于执行。
```
两者意思不同，但却相关，并发的重点是组合，而并行则强调的是执行。
可以参照*[Concurrency is not Parallelism](http://www.vaikan.com/docs/Concurrency-is-not-Parallelism/#landing-slide)*加深理解

Go语言提供：

```text
并发执行(goroutines)
同步和消息传输(channels)
多路并发控制(select)
```

Go语言创建一个并发任务单元只需要在函数调用之前添加go关键字即可，每个任务单元会保存函数指针、调用参数，还会分配执行所需的
栈内存空间，相比系统默认MB级别的线程栈，goroutine自定义栈仅须2KB，所以能够创建上千万个并发任务，自定义栈采取的是按需分配
策略，在需要时进行扩容，最大能够达到GB的规模。
比如创建一个并发任务单元：

```go
package main

func main(){
    go println("hello gopher!!")
    go func(msg string) {
        fmt.Println(msg)
    }("going")
}
```
使用go创建完并发任务之后，新建的任务会被放置在任务队列中，等待调度器安排合适的系统线程获取执行权，当前流程不会阻塞，也不会等待任务
的启动，并且不能保证运行时并发任务的执行次序。

如何等待并发任务的结束？ 

```
1. 采用通道阻塞机制，然后发出退出信号
2. 等待多个任务单元时，推荐使用sync.WaitGroup
```
- channel wait

```go
func main(){
    done := make(chan bool)     //创建一个无缓冲通道，用于
    go func(s string){
        println("working")
        time.Sleep(time.Second)     //业务逻辑处理
        println(s)
        println("done")
        close(done)
        //done->true
    }("hello gopher")
    println("wait...")
    <-done
}
```
- sync wait

```go
func main(){
    var wg  sync.WaitGroup              
    for i:=0;i <10 ;i++ {
        wg.Add(1)                           //计数+1，注意：要在goroutine之外执行计数加1
        go func(s string,workerID int){
            defer wg.Done()                 //计数-1
            time.Sleep(time.Second)
            println(s , "ID:",workerID)
        }("hello gopher",i)
    }
    println("main wait...")
    wg.Wait()                               //阻塞等待，直到计数器归零
    println("all done.")                                                                       
}
```
- GOMAXPROCS
在日常开发高并发程序时，可能会创建很多协程，而协程是运行在线程基础上的，在任意执行时刻，只希望有限的几个线程参与并发任务的执行，
该数量默认与处理器核数相等，可用runtime.GOMAXPROCS函数进行修改。比如：

```go
func count(){
    x := 0
    for i:=0; i <  math.MaxUint32;i++{
        x += i
    }   
    println(x)
}

func c1(n int){
    for i:= 0;i < n;i++{
        count()
    }   
}

func c2(n int){
    var wg sync.WaitGroup
    for i:= 0;i < n;i++{
        wg.Add(1)
        go func(){
            count()
            wg.Done()
        }()
    }   
    wg.Wait()
}

func main(){
    n := runtime.GOMAXPROCS(0)      //返回CPU数目
    start := time.Now()
    c1(n)
    println("c1: ",time.Now().Sub(start).Seconds())
    start = time.Now()
    c2(n)
    println("c2: ",time.Now().Sub(start).Seconds())                                            
}
输出：
9223372030412324865
9223372030412324865
c1:  +4.143495e+000
9223372030412324865
9223372030412324865
c2:  +2.110481e+000
```
- Local Storage

    Goroutine与线程不同，goroutine无法获取ID，无法设置优先级，也没有局部存储(TLS),甚至连返回值都会被抛弃。但是除了
优先级外，其他功能都能够自己实现，比如：

```go
type gr struct{
    id  int 
    res int 
}
func main(){
    var wg sync.WaitGroup
    var grs [5]gr
    for i:=0;i<len(grs);i++{
        wg.Add(1)
        go func(id int){
            grs[id].id = id
            grs[id].res = id*10                                                                
            wg.Done()
        }(i)
    }   
    wg.Wait()
    fmt.Println(grs)
}
输出：
[{0 0} {1 10} {2 20} {3 30} {4 40}]
局部缓存也可以使用切片、字典或通道等，如果使用字典需要注意同步处理。
```
- Gosched

Gosched机制是主动释放出线程去执行其它任务单元，将该任务单元放回任务队列中，等待下一次线程调度时恢复执行。
因为运行时，调度器会主动向占用长时间(10ms)运行的任务发出抢占调度，所以该函数很少被显示应用，比如：

```go
func say(s string){
    for i:=0; i < 10;i++{
        runtime.Gosched()       //让出当前线程
        fmt.Println(s)
    }   
}

func main(){
    //runtime.GOMAXPROCS(2)                                                                    
    go say("hello")
    say("gopher")
}
```
- Goexit

Goexit机制是让goroutine自身主动结束，立即终止整个调用堆栈，并确保所有已注册的延迟函数调用都会按照FIFO规则被执行，该函数不会影响其他并发任务单元的运行，
也不会引发panic，自然不会被捕获，比如：

```go
func main(){
    done := make(chan bool)
    go func(){
        defer close(done)
        defer println("a")
        func(){
            println("b")
            defer func(){
                println("recover is nil: ",recover() == nil)                                   
            }() 
            func(){
                println("c")
                runtime.Goexit()
                println("c.exit")
            }() 
            println("b.exit")
        }() 
        println("a.exit")
    }() 
    <-done
}
输出：
b
c
recover is nil:  true
a
```
注意：os.Exit函数是终止整个进程，不会执行任何延迟调用

- 通道(channel)
Channel是用来在并发程序场景下，同步多goroutine之间进行安全通信的一种机制;
Go语言鼓励使用CSP(Communicating Sequential Process)通道代替内存共享，实现并发环境下安全通信。
通道从底层实现来看是一个(FIFO)队列，在使用通道时有两种模式：

同步模式：
发送和接收双方需要匹配出现，匹配成功后，将数据复制给对方，
如果匹配失败，则有一方被置入等待队列(阻塞)，直到另一方出现后才被唤醒。
保证取数据‘先于’放数据。同样说白了就是保证放的时候肯定有另外的goroutine在取，否则就因放不进去而阻塞。

异步模式：
该模式抢夺的则是数据缓冲区，发送方要求有空间可以写入，接收方则要求缓冲区有数据可以读取，
在不满足需求的情况下，双方同样会加入到等待队列(阻塞)，直到缓冲区腾出空间或有新数据写入。
保证往里放数据‘先于’对应的取数据。说白了就是保证在取的时候里面肯定有数据，否则就因取不到而阻塞。

使用通道前，必须使用内置函数make先显示创建通道，使用"<-"符号来操作通道，使用结束后，
同样使用内部函数close显示关闭通道，通道关闭之后，不能再向通道写入数据，比如:
在前面已经描述过利用通道作为事件通知的场景，下面进一步了解一下通道。

```go
var ChanName = make("chan" | "chan" "<-" | "<-" "chan" ElementType,Capacity)  .
...
close(ChanName)
```
同步模式：

```go
func main(){
    var ch = make(chan string,3)    //创建带3个字符串数据缓冲区的异步通道
    ch <- "hello"
    ch <- "gopher"          //还有空间，不会阻塞
    ch <- "1"
    //ch <- "2"
    println(<-ch)
    println(<-ch)                                                                                                         
}
```

缓冲区大小是通道的一个属性，而不是类型的一部分，另外通道自身也是一种引用类型，所以可以
相等操作符来判断是否为同一对象或者是否为nil，比如：

```go
func main(){
    var b = make(chan int,10)
    var a = make(chan int)
    var c chan string                                                                                                     
    println(a == b)
    println(c == nil)
}
输出：
false
true
```
通过使用cap和len函数来计算通道缓冲区的容量和当前已缓冲的数量，对于非缓冲区的通道则返回0,比如：
.
```go
func main(){
    var a = make(chan<- int)
    var b = make(chan<- int,10)
    b <- 1
    b <- 2
    b <- 3
    println(cap(a), len(a))                                                                                               
    println(cap(b), len(b))
}
输出：
0 0
10 3
```
除了使用简单的方式发送和接收通道，还可以使用range和multi-valued assignment
形式进行接收操作，比如：

```go
func main(){
    done := make(chan bool) // 创建一个channel用以同步goroutine
    c := make(chan int)
    go func() {             // 在goroutine中执行输出操作
        defer close(done)
        println("read message")
        /*
        for {
            x, ok := <-c        //阻塞5秒
            if !ok{
                break
            }
            println(x)
        }*/
        for x:= range c{        //阻塞5秒
            println(x)                                                                                                    
        }
        time.Sleep(2 * time.Second)
    }()
    time.Sleep(5 * time.Second)
    c <- 1
    c <- 2
    c <- 3
    close(c)
    println("waiting over")
    <-done                      // 等待goroutine结束
}
```
通道作为事件通知，并非只是通知结束，可以作为任何事件的通知，比如：

```go
func run(id int, wg *sync.WaitGroup,ready <-chan bool){
    defer wg.Done()
    println(id, ": Ready")
    <-ready
    println(id, ": Runing...")
    time.Sleep(time.Second *1) 
    println(id, ": Ok")
}

func main(){
    var wg sync.WaitGroup
    ready := make(chan bool)
    for i:=0 ;i < 5;i++{
        wg.Add(1)
        go run(i,&wg, ready)
    }   
    time.Sleep(time.Second *3)                                                                                            
    close(ready)
    wg.Wait()
    println("End")
}
输出：
...
```
对于closed或nil的通道，都有相应的发送和接收规则如下：

```text
向closed状态的通道发送数据会引发panic
从closed状态的通道读取数据，返回已缓存的数据或零值
无论发送和接收nil状态的通道，都会被阻塞(死锁)
```
```go
func main() {
    ch := make(chan bool, 2)
    ch <- true
    ch <- true
    close(ch)

    for i := 0; i < cap(ch) +1 ; i++ {
        v, ok := <-ch
        fmt.Println(v, ok) 
    }   
}
输出：
true true
true true
false false         
注意：
第一个false 表示的是 channel 类型的零值
第二个false 表示的是 channel 的启用状态，false表示 channel 关闭状态
重复关闭和关闭nil状态的通道都会引发panic
```
- 单向
为了使程序的操作逻辑更加严谨，日常开发中可以限制通道的收发方向，因为通道默认是双向的，并不
区分接收和发送端，比如：

```go
func main(){
    var wg sync.WaitGroup
    var c = make(chan string)
    var recv <-chan string = c      //创建接收方向的通道
    var send chan<- string = c      //创建发送方向的通道
    wg.Add(2)
    go func(){
        defer wg.Done()
        for x := range recv{
            println(x)
        }   
    }() 
    go func(){
        defer wg.Done()
        send <- "hello"
        send <- "gopher"
        close(send)             //关闭发送方向的通道
    }()                                                                                                                   
    wg.Wait()                   //等待结束
}
输出：
hello
gopher
```
是否可以在单向通道上进行逆向操作，以及关闭接收方向通道 ？

```go
func main(){
    var c = make(chan string,2)
    var recv <-chan string = c 
    var send chan<- string = c 
    recv <- "hello"
    <-send
    close(recv)
}
输出：
invalid operation: recv <- "hello" (send to receive-only type <-chan string)
invalid operation: <-send (receive from send-only type chan<- string)
invalid operation: close(recv) (cannot close receive-only channel)
```
是否可以将单向通道转回正常通道 ？

```go
func main(){
    var a,b chan string
    var c = make(chan string)                                                                                             
    var recv <-chan string = c
    var send chan<- string = c
    a = (chan string)(recv)
    b = (chan string)(send)
}
输出：
cannot convert recv (type <-chan string) to type chan string
cannot convert send (type chan<- string) to type chan string
```
- 选择(select)
在日常开发中，经常会遇到同时处理多个通道的场景，那么可以使用select语句，进行随即选择一个可用的通道做收发处理，比如：

```go
func main(){
	var wg sync.WaitGroup
	wg.Add(3)
	var a ,b = make(chan int), make(chan int)
	go func(){
		defer wg.Done()
		for{
			select{             //随即选择可读的通道
			case i,ok := <-a:
				if !ok{
					a = nil     //如果通道关闭了，则设置为nil阻塞，
					break
				}
				println("a", i)
			case i,ok := <-b:
				if !ok{
					b = nil
					break
				}
				println("b", i)
			}
			if a ==nil && b ==nil{      //通道全部关闭后，则终止接收
				return
			}
		}
	}()
	go func(){
		defer wg.Done()
		defer close(a)
		for i := 0;i < 2; i++{
			a<- i
		}
	}()
	go func(){
		defer wg.Done()
		defer close(b)
		for i := 0;i < 4; i++{
			b<- i*10
		}
	}()
	wg.Wait()
}
输出：
b 0
a 0
a 1
b 10
b 20
b 30
```
如果全部通道都不可操作时，select会执行default语句，如此可以避开阻塞，比如：

```go
func main(){
    done := make(chan bool)
    c := make(chan int)

    go func(){
        defer close(done)
        for{
            select{
                case x,ok := <-c:
                    if !ok{
                        return
                    }   
                    println(x)
                default:                //关键所在
            }   
            println(time.Now().Unix())
            time.Sleep(time.Second)     //合理处理业务逻辑
        }   
    }() 
    time.Sleep(time.Second * 5)
    c<-10
    close(c)
    <-done
}
```

日常开发中，经常会碰到设置定时器和操作超时的场景，Go标准库提供了timeout,timer和ticker通道实现，比如：

```go
func main(){
    timer := time.NewTimer(time.Second)
    ticker := time.NewTicker(time.Second * 2)
    done := make(chan bool)

    go func(){
        for{
            select{
            case <-time.After(time.Second *5): //模拟超时
                println("time out")
                done<- true
                return
            }
        }
    }()
    go func(){
        for {
            select{
            case <-timer.C:
                println("timer expired")
                timer.Reset(time.Second)  //重置定时器
            case <-ticker.C:
                println("ticker expired")
            case <-done:
                println("Done")
                timer.Stop()        //停止定时器
                return
            }                                                                                                             
        }
    }()
    <-(chan struct{})(nil)  //使用nil通道阻塞进程
}
输出：
timer expired
ticker expired
timer expired
timer expired
ticker expired
timer expired
time out
Done
```

- 信号(Signal)

信号是Linux, 类Unix和其它POSIX兼容的操作系统中用来进程间通讯的一种方式。
一个信号就是一个异步的消息通知，发送给某个进程，或者同进程的某个线程，告诉它们某个事件发生了，
收到信号后，进程会有相应的信号处理函数被执行。Go语言中的信号捕获机制是利用通道来完成的，这里不做过多信号解释，
建议参考 *<< UNIX环境高级编程 >>* 书中关于信号的部分，了解更多关于信号内容。

```go
//func Notify(c chan<- os.Signal, sig ...os.Signal)  //信号注册函数原型  
func main(){
    sigs := make(chan os.Signal，1)  //创建信号通道
    done := make(chan bool)                                                                                           
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM)  //信号注册

    go func() {
        sig := <-sigs        //等待接收信号
        fmt.Println(sig)
        done <- true
    }() 

    fmt.Println("awaiting signal")
    <-done
    fmt.Println("exiting")
}
```
日常开发中，信号处理多用在服务程序开发中，比如可以通过syscall.SIGUSR1信号通知更新配置文件，
结合logrotate与syscall.SIGUSR2信号通知处理日志文件切割，实现优雅关闭服务等，比如：

```go
func main(){
    sigs := make(chan os.Signal，1)
    signal.Notify(sigs, syscall.SIGINT, syscall.SIGTERM，syscall.SIGUSR1,syscall.SIGUSR2)  
endless:
    for {
        select {
        case s := <-sigs:              
            if s == syscall.SIGUSR1{
                ...
            }else if s == syscall.SIGUSR2{
                ...
            }else{
                fmt.Println("Signal (%v) received, stoppingn ", s)
                break endless
            }
        }   
    }   
}
```
- 性能&安全

通道为goroutines之间通信提供了便利的条件，但是在享受便利的同时也应该关注性能和安全问题，比如对于性能来说，
因为通道从实现上来说，内部依然使用锁实现同步机制，所以在高并发和对性能有要求的场景下，
可以将目标数据封装成大数据块进行传输，可以改善频繁加解锁造成的性能问题。
对于安全性来说，在使用通道时，可能引发的goroutine leak，比如当goroutines处于发送和接收阻塞状态时，导致
gorouties会在等待队列中休眠，此时垃圾回收器不会对goroutine和它引用的资源进行回收，造成资源泄露，比如：

```go
func test(){
    c := make(chan int)
    for i:=0;i<10;i++{
        go func(i int){
            println(i," start")
            c <- 1                                                                                                        
            println(i," over")
        }(i) 
    }   
}
func main(){
    test()
    for{
        time.Sleep(time.Second)
        runtime.GC()
    }   
}
输出：
# GODEBUG="gctrace=1,schedtrace=1000,scheddetail=1" ./cha13 
1  start
...
SCHED 0ms: gomaxprocs=2 idleprocs=0 threads=5 spinningthreads=0 idlethreads=0 runqueue=0 gcwaiting=0 nmidlelocked=0 stopwait=0 sysmonwait=0
  P0: status=1 schedtick=2 syscalltick=1 m=4 runqsize=0 gfreecnt=0
  P1: status=1 schedtick=10 syscalltick=0 m=2 runqsize=0 gfreecnt=0
  M4: p=0 curg=9 mallocing=0 throwing=0 preemptoff= locks=2 dying=0 helpgc=0 spinning=false blocked=false lockedg=-1
  M3: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 helpgc=0 spinning=false blocked=false lockedg=-1
  M2: p=1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=1 dying=0 helpgc=0 spinning=true blocked=false lockedg=-1
  M1: p=-1 curg=-1 mallocing=0 throwing=0 preemptoff= locks=1 dying=0 helpgc=0 spinning=false blocked=false lockedg=-1
  M0: p=-1 curg=14 mallocing=0 throwing=0 preemptoff= locks=0 dying=0 helpgc=0 spinning=false blocked=true lockedg=-1
  G1: status=4(sleep) m=-1 lockedm=-1
  G2: status=4(force gc (idle)) m=-1 lockedm=-1
  G3: status=4(GC sweep wait) m=-1 lockedm=-1
  G4: status=4(chan send) m=-1 lockedm=-1
  G5: status=4(chan send) m=-1 lockedm=-1
  G6: status=4(chan send) m=-1 lockedm=-1
  G7: status=4(chan send) m=-1 lockedm=-1
  G8: status=4(chan send) m=-1 lockedm=-1
  G9: status=2() m=4 lockedm=-1
  G10: status=4(chan send) m=-1 lockedm=-1
  G11: status=4(chan send) m=-1 lockedm=-1
  G12: status=4(chan send) m=-1 lockedm=-1
  G13: status=4(chan send) m=-1 lockedm=-1
  G14: status=3() m=0 lockedm=-1
可以看出有大量的goroutine处于chan send状态，无法结束，试想一下在高并发服务的环境下，test函数如果频繁被调用，
秒级就会产生大量阻塞状态的goroutines，会给服务带来很大的负面影响。
```
```
总结：在不同的场景下要注意采用非阻塞还是阻塞机制，以免造成不可必要的安全问题。
```

- 同步

通道解决了goroutines之间数据安全通信的问题，但是通道不能替代锁的功能，他们用在不同的场景，通道用于解决逻辑
层次的多并发处理架构，而锁则是用于解决局部范围内的数据共享访问安全问题。
Go标准库sync提供了互斥锁和读写锁来保护局部范围内的数据访问安全，另有原子操作等机制，满足了日常开发的需要。

- 互斥锁(Mutex)

使用互斥锁加锁后，多goroutine不能读也不能写，允许只有一个goroutine读或者写的场景，所以该锁也叫做全局锁，
并且不能再次对其进行加锁，直到对其解锁后，才能再次加锁．适用于读写不确定场景，即读写次数没有明显的区别，比如：

```go
func main() {
    var state = make(map[int]int)       
    var mutex = &sync.Mutex{}           

    for r := 0; r < 100; r++ {
        go func() {
            total := 0
            for {
                mutex.Lock()
                key := rand.Intn(5)
                total += state[key]
                mutex.Unlock()
            }
        }()
    }

    for w := 0; w < 10; w++ {
        go func() {
            for {
                mutex.Lock()
                key := rand.Intn(5)
                state[key] = val
                mutex.Unlock()
            }
        }()
    }
    time.Sleep(time.Second)
    mutex.Lock()
    fmt.Println("state:", state)
    mutex.Unlock()
}
```
在结构体中使用锁变量时，需要相关方法的接收者必须为指针(pointer-receiver)，否则会因为变量复制
导致锁机制失效，并且锁的粒度要控制在最小范围，及早释放，提高效率，比如：

```go
type data struct{
    state map[int]int
    l sync.Mutex        //互斥锁
}
func (d *data)read(){   //接收者一定为(d *data)，不能为(d data)
    total := 0
    for {
        key := rand.Intn(5)
        d.l.Lock()              //加互斥锁
        total += d.state[key]
        d.l.Unlock()
    }
}
func (d *data)write(){  //接收者一定为(d *data)，不能为(d data)
    for {
        key := rand.Intn(5)
        val := rand.Intn(100)
        d.l.Lock()
        d.state[key] = val
        d.l.Unlock()
    }
}
func main() {
    var d data
     d.state = make(map[int]int)
     for r := 0; r < 10; r++ {
         go d.read()
     }
    for w := 0; w < 10; w++ {
        go d.write()
    }
    time.Sleep(time.Second)
    d.l.Lock()
    fmt.Println(d.state)
    d.l.Unlock()
}
```

- 读写锁(RWMutex)

读写锁是针对读写的互斥锁，为了提高读效率，适用于读多写少的场景;读写锁基本遵循两大原则：

```text
1. 加读锁时，多goroutine可以同时读但不能写
2. 加写锁时，多goroutine不能读也不能写
```
```go
type data struct{
    state map[int]int
    l sync.RWMutex   //读写锁
}
func (d *data)read(){
    total := 0
    for {
        key := rand.Intn(5)
        d.l.RLock()         //加读锁
        total += d.state[key]
        d.l.RUnlock()
    }   
}
func (d *data)write(){
    for {
        key := rand.Intn(5)
        val := rand.Intn(100)
        d.l.Lock()
        d.state[key] = val 
        d.l.Unlock()
    }
}
func main() {
    var d data
     d.state = make(map[int]int)
     for r := 0; r < 10000; r++ {       //多读少写
         go d.read()
     }   
    for w := 0; w < 10; w++ {
        go d.write()
    }
    time.Sleep(time.Second*3)
    d.l.RLock()
    fmt.Println(d.state)
    d.l.RUnlock()
}
```

- 原子操作(atomic)
atomic是最轻量级的锁，原子操作即是进行过程中不能被中断的操作。
针对某个值的原子操作在被进行的过程中，CPU绝不会再去进行其他的针对该值的操作。为了实现这样的严谨性，
原子操作仅会由一个独立的CPU指令代表和完成。比如：

```go
type Counter int64
func (c *Counter) Add() {
    *c++
}
func (c *Counter) Value() int64 {                                                                                         
    return int64(*c)
}
func main(){
    runtime.GOMAXPROCS(runtime.NumCPU())
    counter := Counter(0)
    for i:= 0;i<10;i++{
        go func(){
            for j:= 0; j <1000 ;j++{
                counter.Add()
            }   
        }() 
    }   
    time.Sleep(time.Second *3) 
    fmt.Println(counter.Value())
}
输出：
6845
```
原子操作共有5种：增或减，比较并交换，载入，存储，交换,各原子操作的含义可以查阅对应的文档。

使用原子操作重构上面的例子，比如：

```go
type Counter int64
func (c *Counter) Add() {
    atomic.AddInt64((*int64)(c), 1)
    //*c++
}
func (c *Counter) Value() int64 {
    return atomic.LoadInt64((*int64)(c))
    //return int64(*c)
}
func main(){
    runtime.GOMAXPROCS(runtime.NumCPU())
    counter := Counter(0)

    for i:= 0;i<10;i++{                                                                                                   
        go func(){
            for j:= 0; j <1000 ;j++{
                counter.Add()
            }   
        }() 
    }   

    time.Sleep(time.Second *3) 
    fmt.Println(counter.Value())
}
输出：
10000
```
相关建议：

```text
- 对性能要求较高时， 应该避免使用延迟解锁(defer Unlock)
- 读写并发场景下，使用RWMutex提高性能
- 对单个数据读写操作时，应该使用原子操作
```

Go提供了内置的数据竞争检测工具，可以在测试、编译、运行时传入"-race"选项，go tool就可以打开竞争检查，比如：

```
编译和运行时环境：
var i int = 0
func main() {
    go func() {
        for {
            i++
            fmt.Println("subroutine: i = ", i)
            time.Sleep(1 * time.Second)
        }
    }()

    for {
        i++
        fmt.Println("mainroutine: i = ", i)
        time.Sleep(1 * time.Second)
    }                                                                                                                     
}
输出：
# go build -race file.go
# go run -race file.go

测试时环境：
package myrace

import "fmt"
import "time"
import "testing"                                                                                                          
func race() {
    var i int = 0
    go func() {
        for {
            i++
            fmt.Println("subroutine: i = ", i)
            time.Sleep(1 * time.Second)
        }
    }()

    for {
        i++
        fmt.Println("mainroutine: i = ", i)
        time.Sleep(1 * time.Second)
    }
}
func TestRace(t *testing.T){
    race()
}
输出：
# go test -v -race file_test.go 
```

