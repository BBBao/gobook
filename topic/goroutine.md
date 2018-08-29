### 并发与并行
什么是操作系统的进程(process)和线程(thread)？
比如当运行一个应用程序(IDE等)，操作系统会为这个应用程序启动一个进程，可以将这个进程看做
是一个包含了应用程序在运行中需要用到和维护各种资源的容器。
下图展示了一个包含可能分配的常用资源的进程，这些资源包括但是不限于内存地址空间、文件和设备句柄以及线程；一个线程是一个执行空间，这个空间会被操作系统调度来运行函数中所写的代码，每个进程
至少包括一个线程，每个进程的初始线程被称作主线程，因为执行这个线程的空间是应用程序的本身的空间，所以当主线程终止时，应用程序也会终止。操作系统将线程调度到某个处理器上运行，这个处理器并
不一定是进程所在的处理器，不通的操作系统的线程调度算法一般也不一样，但是这种不同会被操作系统
屏蔽，而不会展示给用户。

![test](/topic/static/process.png  "test")

操作系统会在物理处理器上调度线程来运行，而Go语言的运行时会在逻辑处理器上调度goroutine来
运行，每个逻辑处理器都分别绑定到单个操作系统线程。1.5版本以后，Go语言的运行时默认会为每个
可用的物理处理器分配一个逻辑处理器，在1.5版本之前，默认给整个应用程序只分配一个逻辑处理器。这些逻辑处理器会用于执行所有被创建的goroutine。即便只有一个逻辑处理器，Go也可以高效率的并发调度无数个goroutine。

操作系统线程、逻辑处理器和本地运行队列之间的关系。如果创建一个goroutine并准备运行，这个goroutine就会被放到调度器的全局运行队列中，之后，调度器就会将这些队列中的goroutine分配
给一个逻辑处理器，并放到这个逻辑处理器对应的本地运行队列中，本地运行队列中的goroutine会
一直等待，直到自己被分配的逻辑处理器执行。
![test](/topic/static/goroutine.png  "test")

有时，正在运行的 goroutine 需要执行一个阻塞的系统调用，如打开一个文件。当这类调用 发生时，线程和 goroutine 会从逻辑处理器上分离，该线程会继续阻塞，等待系统调用的返回。
与此同时，这个逻辑处理器就失去了用来运行的线程。所以，调度器会创建一个新线程，并将其 绑定到该逻辑处理器上。之后，调度器会从本地运行队列里选择另一个 goroutine 来运行。一旦 被阻塞的系统调用执行完成并返回，对应的 goroutine 会放回到本地运行队列，而之前的线程会 保存好，以便之后可以继续使用。

并发(concurrency)不是并行(parallelism)。并行是让不同的代码片段同时在不同的物理处 理器上执行。并行的关键是同时做很多事情，而并发是指同时管理很多事情，这些事情可能只做 了一半就被暂停去做别的事情了。在很多情况下，并发的效果比并行好，因为操作系统和硬件的 总资源一般很少，但能支持系统同时做很多事情。这种“使用较少的资源做更多的事情”的哲学， 也是指导 Go 语言设计的哲学。

如果希望让 goroutine 并行，必须使用多于一个逻辑处理器。当有多个逻辑处理器时，调度器会将 goroutine 平等分配到每个逻辑处理器上。会让 goroutine 在不同的线程上运行。不过要想真的实现并行的效果，用户需要让自己的程序运行在有多个物理处理器的机器上。否则，哪怕 Go 语言运行时使用多个线程，goroutine 依然会在同一个物理处理器上并发运行，达不到并行的效果。

下图展示了在一个逻辑处理器上并发运行 goroutine 和在两个逻辑处理器上并行运行两个并 发的 goroutine 之间的区别。调度器包含一些聪明的算法，这些算法会随着 Go 语言的发布被更新 和改进，所以不推荐盲目修改语言运行时对逻辑处理器的默认设置。如果真的认为修改逻辑处理 器的数量可以改进性能，也可以对语言运行时的参数进行细微调整。后面会介绍如何做这种修改。
![test](/topic/static/diff.png  "test")

让我们再深入了解一下调度器的行为，以及调度器是如何创建 goroutine 并管理其寿命的。 我们会先通过在一个逻辑处理器上运行的例子来讲解，再来讨论如何让 goroutine 并行运行。
通过创建两个go协程，分别输出字母表，看输出顺序是否满足并行
```go
func main() {
	runtime.GOMAXPROCS(1)   //// 分配一个逻辑处理器给调度器使用 runtime.GOMAXPROCS(1)
	var wg sync.WaitGroup
	wg.Add(2)
	fmt.Println("Start Goroutines")
	go func() {     //匿名函数
		defer wg.Done()
		for count := 0; count < 3; count++ {
			for char := 'a'; char < 'a'+26; char++ {
				fmt.Printf("%c ", char)
			}
		}
	}()
	go func() {
		defer wg.Done()
		for count := 0; count < 3; count++ {
			for char := 'A'; char < 'A'+26; char++ {
				fmt.Printf("%c ", char)
			}
		}
	}()
	fmt.Println("Waiting To Finish")
	wg.Wait()
	fmt.Println("Finished")
}
```
WaitGroup 是一个计数信号量，可以用来记录并维护运行的 goroutine。如果 WaitGroup 的值大于 0，Wait 方法就会阻塞。在第 18 行，创建了一个 WaitGroup 类型的变量，之后在 第 19 行，将这个 WaitGroup 的值设置为 2，表示有两个正在运行的 goroutine。为了减小 WaitGroup 的值并最终释放 main 函数，要在第 26 和 39 行，使用 defer 声明在函数退出时 调用 Done 方法。
关键字 defer 会修改函数调用时机，在正在执行的函数返回时才真正调用 defer 声明的函 数。对这里的示例程序来说，我们使用关键字 defer 保证，每个 goroutine 一旦完成其工作就调 用 Done 方法。

基于调度器的内部算法，一个正运行的 goroutine 在工作结束前，可以被停止并重新调度。
调度器这样做的目的是防止某个 goroutine 长时间占用逻辑处理器。当 goroutine 占用时间过长时， 调度器会停止当前正运行的 goroutine，并给其他可运行的 goroutine 运行的机会。

下图从逻辑处理器的角度展示了这一场景。在第 1 步，调度器开始运行 goroutine A，而 goroutine B 在运行队列里等待调度。之后，在第 2 步，调度器交换了 goroutine A 和 goroutine B。 由于 goroutine A 并没有完成工作，因此被放回到运行队列。之后，在第 3 步，goroutine B 完成 了它的工作并被系统销毁。这也让 goroutine A 继续之前的工作

![test](/topic/static/exchange.png  "test")

展示goroutine调度器是如何在单个线程上切分时间片的
```go
var wg sync.WaitGroup

func main() {
	runtime.GOMAXPROCS(1)
	fmt.Println("vim-go")
	wg.Add(2)
	go printPrime("A")
	go printPrime("B")
	wg.Wait()
	fmt.Println("Terminating Program")
}

func printPrime(prefix string) {
	defer wg.Done()
next:
	for out := 2; out < 5000; out++ {
		for in := 2; in < out; in++ {
			if out%in == 0 {
				continue next
			}
		}
		fmt.Printf("%s:%d\n", prefix, out)
	}
	fmt.Println("completed", prefix)
}
```

goroutine B 先显示素数。一旦 goroutine B 打印到素数 4591，调度器就会将正运行的 goroutine 切换为 goroutine A。之后 goroutine A 在线程上执行了一段时间，再次切换为 goroutine B。这次 goroutine B 完成了所有的工作。一旦 goroutine B 返回，就会看到线程再次切换到 goroutine A 并 完成所有的工作。每次运行这个程序，调度器切换的时间点都会稍微有些不同。
代码清单 6-1 和代码清单 6-4 中的示例程序展示了调度器如何在一个逻辑处理器上并发运行 多个 goroutine。像之前提到的，Go 标准库的 runtime 包里有一个名为 GOMAXPROCS 的函数， 通过它可以指定调度器可用的逻辑处理器的数量。用这个函数，可以给每个可用的物理处理器在 运行的时候分配一个逻辑处理器,让 goroutine 并行运行。
比如：runtime.GOMAXPROCS(runtime.NumCPU())
包 runtime 提供了修改 Go 语言运行时配置参数的能力。在代码清单 6-6 里，我们使用两 个 runtime 包的函数来修改调度器使用的逻辑处理器的数量。函数 NumCPU 返回可以使用的物 理处理器的数量。因此，调用 GOMAXPROCS 函数就为每个可用的物理处理器创建一个逻辑处理 器。需要强调的是，使用多个逻辑处理器并不意味着性能更好。在修改任何语言运行时配置参数 的时候，都需要配合基准测试来评估程序的运行效果。
如果给调度器分配多个逻辑处理器，我们会看到之前的示例程序的输出行为会有些不同。让 我们把逻辑处理器的数量改为 2，并再次运行第一个打印英文字母表的示例程序

如果仔细查看代码清单 6-8 中的输出，会看到 goroutine 是并行运行的。两个 goroutine 几乎 是同时开始运行的，大小写字母是混合在一起显示的。这是在一台 8 核的电脑上运行程序的输出， 所以每个 goroutine 独自运行在自己的核上。记住，只有在有多个逻辑处理器且可以同时让每个 goroutine 运行在一个可用的物理处理器上的时候，goroutine 才会并行运行。
现在知道了如何创建 goroutine，并了解这背后发生的事情了。下面需要了解一下写并发程序 时的潜在危险，以及需要注意的事情。

### 竞争状态
如果两个或者多个 goroutine 在没有互相同步的情况下，访问某个共享的资源，并试图同时 读和写这个资源，就处于相互竞争的状态，这种情况被称作竞争状态(race candition)。竞争状态 的存在是让并发程序变得复杂的地方，十分容易引起潜在问题。对一个共享资源的读和写操作必 须是原子化的，换句话说，同一时刻只能有一个 goroutine 对共享资源进行读和写操作。如下中给出的是包含竞争状态的示例程序。
```go
var (
	counter int
	wg      sync.WaitGroup
)

func main() {
	wg.Add(2)
	go incCounter(1)
	go incCounter(2)
	wg.Wait()
	fmt.Println(counter)
}

func incCounter(id int) {
	defer wg.Done()
	for i := 0; i < 2; i++ {
		value := counter
		//time.Sleep(2 * time.Second)
		runtime.Gosched()
		value++
		counter = value
	}
}
```
变量 counter 会进行 4 次读和写操作，每个 goroutine 执行两次。但是，程序终止时，counter 变量的值为 2。
![test](/topic/static/counter.png  "test")
每个 goroutine 都会覆盖另一个 goroutine 的工作。这种覆盖发生在 goroutine 切换的时候。每 个 goroutine 创造了一个 counter 变量的副本，之后就切换到另一个 goroutine。当这个 goroutine 再次运行的时候，counter 变量的值已经改变了，但是 goroutine 并没有更新自己的那个副本的值，而是继续使用这个副本的值，用这个值递增，并存回 counter 变量，结果覆盖了另一个 goroutine 完成的工作。


调用了 runtime 包的 Gosched 函数，用于将 goroutine 从当前线程退出， 给其他 goroutine 运行的机会。在两次操作中间这样做的目的是强制调度器切换两个 goroutine， 以便让竞争状态的效果变得更明显，Go 语言有一个特别的工具，可以在代码里检测竞争状态。
```sh
$ go build -race // 用竞争检测器标志来编译程序
```
展示了竞争检测器查到的哪个 goroutine 引发了数据竞争，以及哪两行代码有 冲突。毫不奇怪，这几行代码分别是对 counter 变量的读和写操作。一种修正代码、消除竞争状态的办法是，使用 Go 语言提供的锁机制，来锁住共享资源，从 而保证 goroutine 的同步状态。
#### 锁住共享资源
Go 语言提供了传统的同步 goroutine 的机制，就是对共享资源加锁。如果需要顺序访问一个 整型变量或者一段代码，atomic 和 sync 包里的函数提供了很好的解决方案。下面我们了解一 下 atomic 包里的几个函数以及 sync 包里的 mutex 类型。
##### 原子操作函数
原子函数能够以很底层的加锁机制来同步访问整型变量和指针。我们可以用原子函数来修正上面创建的竞争状态。
```go
func incCounter(id int) {
	defer wg.Done()
	for i := 0; i < 2; i++ {
		atomic.AddInt32(&counter, 1)
		//value := counter
		//time.Sleep(2 * time.Second)
		runtime.Gosched()
		//value++
		//counter = value
	}
}
```
atomic.AddInt32 函数会同步整型值的加法， 方法是强制同一时刻只能有一个 goroutine 运行并完成这个加法操作。当 goroutine 试图去调用任 何原子函数时，这些 goroutine 都会自动根据所引用的变量做同步处理。现在我们得到了正确的值 4。

另外两个有用的原子函数是 LoadInt64 和 StoreInt64。这两个函数提供了一种安全地读和写一个整型值的方式。
练习通过 LoadInt64 和 StoreInt64 如何安全的通知其它goroutine更新退出状态？
```go
var (
	running int64
	wg      sync.WaitGroup
)

func dowork(name string) {
	defer wg.Done()
	for {
		fmt.Printf("Doing %s Work\n", name)
		time.Sleep(200 * time.Millisecond)
		if atomic.LoadInt64(&running) == 1 {
			fmt.Printf("shutdown %s Work\n", name)
			break
		}
	}
}

func monitor() {
	defer wg.Done()
	time.Sleep(2 * time.Second)
	atomic.StoreInt64(&running, 1)
}

func main() {
	fmt.Println("vim-go")
	wg.Add(3)
	go dowork("A")
	go dowork("B")
	go monitor()
	wg.Wait()
}
```
### 互斥锁
另一种同步访问共享资源的方式是使用互斥锁(mutex)。互斥锁这个名字来自互斥(mutual exclusion)的概念。互斥锁用于在代码上创建一个临界区，保证同一时间只有一个 goroutine 可以 执行这个临界区代码,重新实现counter例子。
```go
func incCounter(id int) {
	defer wg.Done()
	for i := 0; i < 2; i++ {
		mutex.Lock()
		//atomic.AddInt32(&counter, 1)
		value := counter
		//time.Sleep(2 * time.Second)
		runtime.Gosched()
		value++
		counter = value
		mutex.Unlock()
	}
}
```

### 通道
在 Go 语言里，你不仅可以使用原子函数和互斥锁来保证对共享资源的安全访 问以及消除竞争状态，还可以使用通道，通过发送和接收需要共享的资源，在 goroutine 之间做同步。
当一个资源需要在 goroutine 之间共享时，通道在 goroutine 之间架起了一个管道，并提供了 确保同步交换数据的机制。声明通道时，需要指定将要被共享的数据的类型。可以通过通道共享 内置类型、命名类型、结构类型和引用类型的值或者指针。例如：
```go
// 无缓冲的整型通道
unbuffered := make(chan int)
// 有缓冲的字符串通道
buffered := make(chan string, 10)
```

通道是否带有缓冲，其行为会有一些不同。理解这个差异对决定到底应该使用还是不使用缓 冲很有帮助。下面我们分别介绍一下这两种类型。

### 无缓冲通道
无缓冲的通道(unbuffered channel)是指在接收前没有能力保存任何值的通道。这种类型的通道要求发送 goroutine 和接收 goroutine 同时准备好，才能完成发送和接收操作。如果两个 goroutine 没有同时准备好，通道会导致先执行发送或接收操作的 goroutine 阻塞等待。这种对通道进行发送和接收的交互行为本身就是同步的。其中任意一个操作都无法离开另一个操作单独存在。
下图展示两个 goroutine 如何利用无缓冲的通道来共享一个值：
![test](/topic/static/unbuf.png  "test")
在第 1 步，两个 goroutine 都到达通道，但哪个都没有开始执行发送或者接收。在第 2 步，左侧 的 goroutine 将它的手伸进了通道，这模拟了向通道发送数据的行为。这时，这个 goroutine 会在 通道中被锁住，直到交换完成。在第 3 步，右侧的 goroutine 将它的手放入通道，这模拟了从通 道里接收数据。这个 goroutine 一样也会在通道中被锁住，直到交换完成。在第 4 步和第 5 步， 进行交换，并最终，在第 6 步，两个 goroutine 都将它们的手从通道里拿出来，这模拟了被锁住 的 goroutine 得到释放。两个 goroutine 现在都可以去做别的事情了。

为了讲得更清楚，让我们来看两个完整的例子。这两个例子都会使用无缓冲的通道在两个 goroutine 之间同步交换数据。
在网球比赛中，两位选手会把球在两个人之间来回传递。选手总是处在以下两种状态之一: 要么在等待接球，要么将球打向对方。可以使用两个 goroutine 来模拟网球比赛，并使用无缓冲的通道来模拟球的来回;

```go
var (
	wg sync.WaitGroup
)

func player(name string, court chan int) {
	defer wg.Done()
	for {
		ball, ok := <-court
		if !ok {
			fmt.Printf("%s Won!!!\n", name)
			break
		}
		n := rand.Intn(100)
		if n%13 == 0 {
			fmt.Printf("%s Missed!!! rand number is %d\n", name, n)
			close(court)
			break
		}
		fmt.Printf("player %s hit the ball %d,rand number is %d\n", name, ball, n)
		ball++
		court <- ball
	}
}
func init(){
		rand.Seed(time.Now().UnixNano())
}
func main() {
	court := make(chan int)
	wg.Add(2)
	go player("black", court)
	go player("white", court)
	court <- 1
	wg.Wait()
}
```
在 main 函数的第 22 行，创建了一个 int 类型的无缓冲的通道，让两个 goroutine 在击球时 能够互相同步。之后在第 28 行和第 29 行，创建了参与比赛的两个 goroutine。在这个时候，两个 goroutine 都阻塞住等待击球。在第 32 行，将球发到通道里，程序开始执行这个比赛，直到某个 goroutine 输掉比赛。
在 player 函数里，在第 43 行可以找到一个无限循环的 for 语句。在这个循环里，是玩游 戏的过程。在第 45 行，goroutine 从通道接收数据，用来表示等待接球。这个接收动作会锁住goroutine。
直到有数据发送到通道里。通道的接收动作返回时，第 46 行会检测 ok 标志是否为 false。如果这个值是 false，表示通道已经被关闭，游戏结束。在第 53 行到第 60 行，会产 生一个随机数，用来决定 goroutine 是否击中了球。如果击中了球，在第 64 行 ball 的值会递增 1，并在第 67 行，将 ball 作为球重新放入通道，发送给另一位选手。在这个时刻，两个 goroutine 都会被锁住，直到交换完成。最终，某个 goroutine 没有打中球，在第 58 行关闭通道。之后两个 goroutine 都会返回，通过 defer 声明的 Done 会被执行，程序终止。

例2 用不同的模式，使用无缓冲的通道，在 goroutine 之间同步数据，来模拟接力 比赛。在接力比赛里，4 个跑步者围绕赛道轮流跑(如代码清单。第二个、第三个和 第四个跑步者要接到前一位跑步者的接力棒后才能起跑。比赛中最重要的部分是要传递接力棒， 要求同步传递。在同步接力棒的时候，参与接力的两个跑步者必须在同一时刻准备好交接。
```go
var (
	wg sync.WaitGroup
)

func main() {
	baton := make(chan int)
	wg.Add(1)
	go Runner(baton)
	baton <- 1
	wg.Wait()
}
func Runner(baton chan int) {
	var newRunner int
	runner := <-baton
	fmt.Printf("Runner %d running with the Baton\n", runner)
	if runner != 4 {
		newRunner = runner + 1
		fmt.Printf("Runner %d to the line\n", runner)
		go Runner(baton)
	}
	//模拟跑步
	time.Sleep(100 * time.Millisecond)
	if runner == 4 {
		fmt.Printf("Runner %d Finished,Race Over\n", runner)
		wg.Done()
		return
	}
	fmt.Printf("Runner %d Exchange With Runner %d\n", runner, newRunner)
	baton <- newRunner
}
```
在 main 函数的创建了一个无缓冲的 int 类型的通道 baton，用来同步传递接力棒，接下来我们给 WaitGroup 加 1，这样 main 函数就会等最后一位跑步者跑步结束。创建一个 goroutine用来表示第一位跑步者来到跑道。之后将接力棒交给这个跑步者， 比赛开始。最终，main 函数阻塞在 WaitGroup，等候最后一位跑步者完成比赛。在 Runner goroutine 里，可以看到接力棒 baton 是如何在跑步者之间传递的。goroutine 对 baton 通道执行接收操作，表示等候接力棒。一旦接力棒传了进来，就会
创建一位新跑步者，准备接力下一棒，直到 goroutine 是第四个跑步者。跑步者围绕 跑道跑 100 ms。如果第四个跑步者完成了比赛，就调用 Done，将 WaitGroup 减 1， 之后 goroutine 返回。如果这个 goroutine 不是第四个跑步者，接力棒会交到下一个已经在等待的跑步者手上。在这个时候，goroutine 会被锁住，直到交接完成。
在这两个例子里，我们使用无缓冲的通道同步 goroutine，模拟了网球和接力赛。代码的流程 与这两个活动在真实世界中的流程完全一样，这样的代码很容易读懂。现在知道了无缓冲的通道 是如何工作的，接下来我们会学习有缓冲的通道的工作方法。
### 有缓冲通道
有缓冲的通道(buffered channel)是一种在被接收前能存储一个或者多个值的通道。这种类 型的通道并不强制要求 goroutine 之间必须同时完成发送和接收。通道会阻塞发送和接收动作的 条件也会不同。只有在通道中没有要接收的值时，接收动作才会阻塞。只有在通道没有可用缓冲 区容纳被发送的值时，发送动作才会阻塞。这导致有缓冲的通道和无缓冲的通道之间的一个很大 的不同:无缓冲的通道保证进行发送和接收的 goroutine 会在同一时间进行数据交换;有缓冲的 通道没有这种保证。
可以看到两个 goroutine 分别向有缓冲的通道里增加一个值和从有缓冲的通道里移 除一个值。在第 1 步，右侧的 goroutine 正在从通道接收一个值。在第 2 步，右侧的这个 goroutine 独立完成了接收值的动作，而左侧的 goroutine 正在发送一个新值到通道里。在第 3 步，左侧的 goroutine 还在向通道发送新值，而右侧的 goroutine 正在从通道接收另外一个值。这个步骤里的 两个操作既不是同步的，也不会互相阻塞。最后，在第 4 步，所有的发送和接收都完成，而通道 里还有几个值，也有一些空间可以存更多的值。
![test](/topic/static/buff-chan.png  "test")
让我们看一个使用有缓冲的通道的例子，这个例子管理一组 goroutine 来接收并完成工作。 有缓冲的通道提供了一种清晰而直观的方式来实现这个功能:
```go
var (
	wg sync.WaitGroup
)

const (
	numberGoroutines = 4  //goroutines
	taskLoad         = 10 //to do
)

func worker(tasks chan string, gr int) {
	defer wg.Done()
	for {
		task, ok := <-tasks
		if !ok {
			fmt.Println("closed...")
			return
		}
		fmt.Printf("worker %d start to work %s\n", gr, task)
		sleep := rand.Intn(100)
		time.Sleep(time.Duration(sleep) * time.Millisecond)
		fmt.Printf("Worker: %d : Completed %s\n", gr, task)
	}
}

func init() {
	rand.Seed(time.Now().UnixNano())
}
func main() {
	var tasks = make(chan string, taskLoad)
	wg.Add(numberGoroutines)
	for gr := 1; gr <= numberGoroutines; gr++ {
		go worker(tasks, gr)
	}
	for post := 1; post <= taskLoad; post++ {
		tasks <- fmt.Sprintf("Task : %d", post)
	}
	close(tasks)
	wg.Wait()
}
```
由于程序和 Go 语言的调度器带有随机成分，这个程序每次执行得到的输出会不一样。不过， 通过有缓冲的通道，使用所有 4 个 goroutine 来完成工作，这个流程不会变。从输出可以看到每 个 goroutine 是如何接收从通道里分发的工作

关闭通道的代码非常重要。当通道关闭后，goroutine 依旧可以从通道接收数据， 但是不能再向通道里发送数据。能够从已经关闭的通道接收数据这一点非常重要，因为这允许通 道关闭后依旧能取出其中缓冲的全部值，而不会有数据丢失。从一个已经关闭且没有数据的通道 里获取数据，总会立刻返回，并返回一个通道类型的零值。如果在获取通道时还加入了可选的标志，就能得到通道的状态信息。

总结：
并发是指 goroutine 运行的时候是相互独立的。
使用关键字 go 创建 goroutine 来运行函数。
goroutine 在逻辑处理器上执行，而逻辑处理器具有独立的系统线程和运行队列。
竞争状态是指两个或者多个 goroutine 试图访问同一个资源。
原子函数和互斥锁提供了一种防止出现竞争状态的办法。
通道提供了一种在两个 goroutine 之间共享数据的简单方法。 
无缓冲的通道保证同时交换数据，而有缓冲的通道不做这种保证

### 读chan死锁问题分析
```go
import (
	"fmt"
	"time"
)

func main() {
	fmt.Println("vim-go")
	ch := make(chan int)
	result := make(chan int)
	for i := 0; i < 2; i++ {
		go func() {
			x := <-ch
			result <- x
		}()
	}
	ch <- 1
	ch <- 2
	for x := range result {
		fmt.Printf("read: %d\n", x)
	}
}
```
方法一：
主程序中的for x := range result代码，在处理完1和2两条数据后，还一直在等待 results channel 的内容，无法结束。这样的话，Go就会判定产生了死锁
修复办法：
```go
	go func() {
		for x := range result {
			fmt.Printf("read: %d\n", x)
		}
	}()
	time.Sleep(3 * time.Second)
```
这个方法是把 results channel 的接收处理放到一个goroutine里去做。这样的话，主程序就不会卡住不动，即不会产生死锁了。
总结：
fatal error: all goroutines are asleep - deadlock!）这种抛出 error 的死锁，是只有主程序卡住不动才会生产的吗？如果是的话，那不让主程序卡住不动，就不会发生这种事情了。所以以下的各种解决方案，都是围绕如何“不让主程序卡住”来做的。 

方法二：
最上面死锁的原因是，for result循环一直在等待输入，但实际上输入只有2个，处理完后就没有了。为了解决这个问题，把 results channel 关闭，for result循环就可以正常退出了。
具体细节是，在所有的 results channel 的输入处理之前，wg.Wait()这个goroutine会处于等待状态。当处理完后（wg.Done），wg.Wait()就会放开执行，执行后面的close(results)。
```go
func main() {
	ch := make(chan int)
	result := make(chan int)
	for i := 0; i < 2; i++ {
		wg.Add(1)
		go func() {
			defer wg.Done()
			x := <-ch
			result <- x
		}()
	}
	go func() {
		wg.Wait()
		close(result)
	}()

	ch <- 1
	ch <- 2
	for x := range result {
		fmt.Printf("read: %d\n", x)
	}
}
```

总结： channel 上如果发生了流入和流出不配对，就可能会发生死锁
```go
无缓冲
func deadLock1() {
	ch := make(chan int)
	ch <- 1
}
有缓冲
func deadLock2() {
	ch := make(chan int, 1)
	ch <- 1
	ch <- 1
}
```
#### 如何避免死锁
1. 明确保证写入和读出成对出现
2. 如果无法保证成对出现的话，在输入完成之后关闭通道
3. 明确保证成对出现的话，可以使用缓冲来解决写入的阻塞
