# 测试

```
1. 单元测试
2. 性能测试
3. 代码覆盖率
4. 性能监控
```

#### 单元测试
单元测试是开发人员的一项基本工作，Go语言提供的单元测试机制不仅可以测试代码的逻辑算法是否准确，
并且可以监控代码的质量。在任何时候都可以使用简单的命令验证全部功能，找出未完成的任务和因为中途代码修改导致的错误，它与性能测试以及
代码覆盖率等一起保障了代码总是在可控范围内。单元测试伴随在整个项目开发过程中，所以说单元测试代码同主工程一样重要。

Go语言工具链和标准库提供了单元测试框架，给开发人员带来很大的便利条件，但也要按照如下语法规范编写测试代码，比如：
```
- 测试代码要以*_test.go文件的形式放在当前包
- 测试函数要以*Test形式命名
- go test xxx_test.go 测试命令将忽略以"\_"和 "."开头的测试文件
- go install/build 正常编译操作会忽略测试文件
```

```go
import "testing"
func sum(x,y int)int{
    return x+y 
}
func TestSum(t *testing.T){                                                               
    if sum(1,2) !=3 {
        t.FailNow()
    }   
}
测试：
# go test -v  //测试当前包和所有子包，用go test -v ./...
```

testing.T 提供了控制测试结果和行为的方法，常用的如下，比如：

```
Fail: 失败后继续执行当前测试函数
FailNow: 失败后立即终止执行当前测试函数
SkipNow: 跳过执行当前测试函数
Log: 报告错误信息，仅当失败或 -v 时输出
Parallel: 与有同样设置的测试函数并行执行
Error: 报告错误信息后继续，相当于Log + Fail
Fatal: 报告错误信息后停止，相当于FailNow + Log
```
冒泡排序
```go
func bubbleSort(items []int) {
	var (
		n       = len(items)
		swapped = true
	)
	for swapped {
		swapped = false
		for i := 0; i < n-1; i++ {
			if items[i] > items[i+1] {
				items[i+1], items[i] = items[i], items[i+1]
				swapped = true
			}
		}
		n = n - 1
	}
}
func main() {
	fmt.Println("vim-go")
	items := []int{3, 44, 22, 1}
	<!-- var z []int
	for i := 0; i < 1000000; i++ {
		z = append(z, rand.Intn(100000))
	} -->
	bubbleSort(items)
	fmt.Println(items)
}

```
快速排序
```go
func recurtion(arr []int, left int, right int) {
	if left < right {
		var i, j = left, right
		var key = arr[i]
		for i < j {
			for i < j && arr[j] >= key {
				j--
			}
			arr[i] = arr[j]
			for i < j && arr[i] <= key {
				i++
			}
			arr[j] = arr[i]
		}
		arr[i] = key
		recurtion(arr, left, i-1)
		recurtion(arr, i+1, right)
	}
}
```

可以利用t.Parallel()函数并行执行多个test case，提高测试效率，比如：

```go
func TestA(t *testing.T){
    t.Parallel()
    time.Sleep(time.Second *2)
}

func TestB(t *testing.T){
    t.Parallel()
    time.Sleep(time.Second *2)
}
```

- TableDrivenTests

如果在测试过程中发现自己使用复制和粘贴来做重复的功能测试，那么可以考虑使用类似数据表的模式来批量输入
条件进行测试，每个表项是一个完整的测试用例的输入和预期结果，比如：
```go
func sum(x,y int)int{
    return x+y 
}
func TestSum(t *testing.T){
    var tests = []struct{
        x, y , expect int 
    }{  
        {1,2,3},
        {1,3,3},
        {1,4,5},
    }   
    for _, u := range tests{
        result := sum(u.x, u.y)
        if result != u.expect {
            t.Errorf("%d + %d expect %d, actual result: %d\n", u.x, u.y, u.expect, result)                 
        }   
    }   
}
```

- TestMain
有时候需要为测试用例提供初始化和清理工作，但是testing并没有提供setup/teardown的机制，所以
可以利用TestMain函数来增加测试用例的测试入口，实现控制测试用例的调用方式，比如：

```go
func sum(x,y int)int{
    return x+y 
}

func TestSum(t *testing.T){
    println("TSum")
    if sum(1,2) !=3 {
        t.FailNow()
    }
}

func TestMain(m *testing.M){
    println("setup")
    m.Run()
    println("teardown")
    os.Exit(0)
}
```

可以借助MainStart自行构建M对象，实现多测试用例的组合，比如：
```go
func MainStart(matchString func(pat, str string) (bool, error), tests []InternalTest, benchmarks []In    ternalBenchmark, examples []InternalExample) *M
func TestMain(m *testing.M){
    match := func(pat, str string)(bool,error){
        matchRe, err := regexp.Compile(pat)
        if err != nil{
            return false, err 
        }   
        return matchRe.MatchString(str), nil 
    }   

    tests := []testing.InternalTest{
        {"sum",TestSum},
        {"a",TestA},                                                                                       
        {"b",TestB},
    }   

    benchmarks := []testing.InternalBenchmark{}
    examples := []testing.InternalExample{}
    m = testing.MainStart(match, tests,benchmarks, examples)
    os.Exit(m.Run())
}
执行：
go test -v sum3_test.go -run "a"
```

常用测试参数：

| 参数 | 说明 | 示例 |
|:---:|:---:|:---:|
| -args | go test 测试命令行参数 |-args "a" |
| -v | 输出测试详细信息 | |
| -parallel | 并行测试，默认为GOMAXPROCS| -parallel 4|
| -run | 指定测试函数，支持正则表达式| -run "Sum" |
| -timeout | 指定全部测试完成超时时间，默认是 10m | -timeout 3m10s |
| -count | 指定重复测试次数| -count 100 |
| -cpu | 指定使用的CPU| -cpu 1,2,3 |

如何设计测试用例：
```
等价类划分：
将所有可能的输入数据划分成若干个子集，在每个子集中，如果任意一个输入数据对于揭露程序中潜在
错误都具有同等效果，那么这样的子集就构成了一个等价类。 等价类划分 Ø举例:输入项-考试成绩，及格线-60分，分数范围-0~100
1)有效等价类 :0~59 之间的任意整数;59~100 之间的任意整数;
2)无效等价类 :小于 0 的负数;大于 100 的整数;0~100 之间的任何浮点数;其他任意非数字字符。
边界值覆盖：
选取输入、输出的边界值进行测试
举例:输入项-考试成绩，及格线-60分，分数范围-0~100
1)边界值数据应该包括:-1，0，1，59，60，61，99，100，101
错误推测法：
基于对被测试软件系统设计的理解、过往经验以及个人直觉，推测出软件可能存在的缺陷，从而有针对 性地设计测试用例的方法。(需求 + 设计细节 + 个人经验)
弹性云错误推测举例: 1)依赖第三方API出错的处理逻辑;同步，异步，反复多次; 
2)某个输入参数为空的处理逻辑;直接后端调取API操作; 
3)数据库操作异常的处理逻辑;
```

#### mock
mock 是单元测试中常用的一种测试手法，mock 对象被定义，并能够替换掉真实的对象被测试的函数所调用。而mock对象可以被开发人员很灵活的指定传入参数，调用次数，返回值和执行动作，来满足测试的各种情景假设。
那什么情况下需要使用mock呢?一般来说分这几种情况:
```
1. 依赖的服务返回不确定的结果，如获取当前时间。
2. 依赖的服务返回状态中有的难以重建或复现，比如模拟网络错误。
3. 依赖的服务搭建环境代价高，速度慢，需要一定的成本，比如数据库，web服务
4. 依赖的服务行为多变。
```
不同的编程语言根据语言特点其所采用的 mock 实现有所不同。在go语言中，mock一般通过两种方法来实现，一种是依赖注入，一种是通过interface,下面我们分别通过例子来说明这两种技术实践；
##### 依赖注入
依赖注入为一个类或者函数A，用到内部对象B，B在A的外部创建，当运行A调用B时，通过某种方式将外部创建的B的实例赋给A内的B。
这样当A调用B时，B就会按照外部定义的方式去运行。下面是一个例子:
##### gomock 
```sh
$ go install code.google.com/p/gomock/gomock
$ go install code.google.com/p/gomock/mockgen
```

#### 性能测试
性能测试函数是以Benchmark为名称前缀的函数同样是保存在*_test.go文件中，
执行性能测试时，go test 必须使用-bench选项，它通过逐步调整B.N值，反复执行
测试函数，直到获得准确的测试结果，比如：

```go
func sum(x ,y int)int{
    return x+y 
}
func BenchmarkSum(b *testing.B){
    println()
    for i:= 0; i < b.N;i++{
        sum(1,1)
    }   
}
输出：
#go test -v sum_test.go -bench .
BenchmarkSum-2
b.N:  1
b.N:  100
b.N:  10000
b.N:  1000000
b.N:  100000000
b.N:  2000000000
2000000000	         0.34 ns/op
PASS
ok  	command-line-arguments	0.732s
```

```
如果仅想进行性能测试， 可用-run NONE选项来忽略所有单元测试用例
```

在测试某些比较耗时代码场景下，默认的循环次数就会过少，这种情况取的平均值可能就不足以
准确计算性能，此时可以用 benchtime 选项来设定最小测试时间，增加循环次数，进而返回更准确的结果，比如：

```
func mysleep(){
    time.Sleep(time.Second)
}
func BenchmarkMysleep(b *testing.B){
    for i:= 0; i < b.N;i++{                                                     
        mysleep()
    }   
}
输出：
# go test -v sum5_test.go -bench .  -benchtime 10s
# go test -v  -bench='BenchmarkMysleep' -benchtime 3
BenchmarkMysleep-2   	      10	1000629167 ns/op
PASS
ok  	command-line-arguments	11.013s
```

如果在测试过程中需要执行其它一些业务逻辑，而不想因这些逻辑影响测试结果,那么可以临时阻止
计时器工作，比如：

```go
func sum(x ,y int)int{
    return x+y 
}
func BenchmarkSum(b *testing.B){
    time.Sleep(time.Second)
    b.ResetTimer()          //重置计时器
    for i:= 0; i < b.N;i++{
        if i == 100{
            b.StopTimer()    //暂停                                                   
            //do other
            time.Sleep(time.Second)
            //done
            b.StartTimer()  //恢复
        }   
        sum(1,1)
    }   
}
输出：
暂停计时器的效果：
BenchmarkSum-2   	2000000000	         0.72 ns/op
PASS
ok  	command-line-arguments	11.570s
不暂停计时器的效果：
BenchmarkSum-2   	   10000	    100053 ns/op
PASS
ok  	command-line-arguments	4.010s
```

性能测试过程中，除了关注目标代码的执行时间，还应该关注代码在堆上的内存分配，因为
内存分配与垃圾回收的相关操作也应该计入消耗成本，优化代码时也应该把此项作为的优化参考，比如：

```go
func malloc()[]int{
    return make([]int, 1024)
}
func BenchmarkMalloc(b *testing.B){ 
    //b.ReportAllocs()       //无论是否使用-benchmem选项，都会输出内存分配信息                                                    
    for i:= 0; i < b.N;i++{
        _ = malloc()
    }   
}
输出结果包含单次测试分配的内存总量和次数, -gcflags "-N -l" 禁用编译器优化和内联优化
# go test -v file_test.go -bench . -benchmem -gcflags "-N -l"
BenchmarkMalloc-2   	 1000000	      2457 ns/op	    8192 B/op	       1 allocs/op
PASS
ok  	command-line-arguments	2.488s
```

#### 代码覆盖率
除了执行单元测试和性能测试来测试代码质量之外，还应该度量测试用例自身的完整和有效性，也就是代码覆盖率。
通过代码覆盖率的值，可以分析出测试用例的编写质量，比如测试用例是否提供了足够多的测试条件，是否执行了
足够的函数、语句、分支和代码行等，以此来量化测试本身，使白盒测试真正起到项目代码质量保障作用。

```go
# cat sum.go
package sum 
import "math"
func sum(x,y int)int{
    if x < 0{
        x = int(math.Abs(float64(x)))
    }   
    if y < 0{
        y = int(math.Abs(float64(y)))                                                          
    }   
    return x+y 
}
# cat sum_test.go
package sum
import "testing"
func TestSum(t *testing.T){
	if sum(1,2) !=3 {
		t.Fatal()
	}
}
输出：
# go test -cover
PASS
coverage: 60.0% of statements
ok  	_/root/data/workspace/src/test/sum	0.003s
```
可以指定 covermode 和 coverprofile 参数,获取更详细测试信息，比如:

```
- set: 是否执行
- count: 执行次数
- atomic: 类似于count，不过支持并发模式
```

```bash
# go test -cover -covermode=count -coverprofile cover.out
```

在浏览器中查看，将鼠标放在代码上可以查看该语句执行次数。
```bash
# go tool cover -html=cover.out
```
![test](/static/testingcover.png  "test")

改造测试用例再进行测试：
```go
func TestSum(t *testing.T) {
	if sum(1, 2) != 3 {
		t.Fatal()
	}
	if sum(-1, 2) != 3 {
		t.Log("pass")
	}
	if sum(1, -2) != 3 {
		t.Log("pass")
	}
}
```

#### profiling
在计算机性能调试领域里，profiling 就是对应用的画像，这里画像就是应用使用 CPU 和内存的情况。也就是说应用使用了多少 CPU 资源？都是哪些部分在使用？每个函数使用的比例是多少？有哪些函数在等待 CPU 资源？知道了这些，我们就能对应用进行规划，也能快速定位性能瓶颈。

golang 是一个对性能特别看重的语言，因此语言中自带了 profiling 的库，这篇文章就要讲解怎么在 golang 中做 profiling。

在 go 语言中，主要关注的应用运行情况主要包括以下几种：

CPU profile：报告程序的 CPU 使用情况，按照一定频率去采集应用程序在 CPU 和寄存器上面的数据
Memory Profile（Heap Profile）：报告程序的内存使用情况
Block Profiling：报告 goroutines 不在运行状态的情况，可以用来分析和查找死锁等性能瓶颈
Goroutine Profiling：报告 goroutines 的使用情况，有哪些 goroutine，它们的调用关系是怎样的

#### 性能监控
是什么引发了性能问题呢？延迟过高、内存占用过多、意外阻塞等都是影响性能的表现，
Go语言提供了两种性能监控方式帮助开发者定位问题原因：

```
- 测试时输出并保存相关数据，进行初期评估。
- 运行时通过WEB接口获取实时性能监控数据。
```
比如：

```bash
# go test -run NONE -bench . -memprofile mem.out -cpuprofile cpu.out net/tcp
```
在cpu.out 和 mem.out文件中分别保存了CPU和内存的采样数据。
参数描述如下：
```
注意在执行性能测试时，为了有足够长的采样时间，可以手动设定benchtime参数
```

- pprof

Go语言提供了 pprof 包来做代码的性能监控，可以使用pporf静态分析上面的性能测试结果，并且也可以
进行实时性能监控，pprof 的最大的优点之一是它是的性能负载很小，可以在生产环境中使用，不会对 web 请求响应造成明显的性能消耗。
pprof在Go标准库中存在于两个路径下：

```
/src/net/http/pprof
/src/runtime/pprof
```
其中net/http/pprof是对runtime/pprof包的封装，用户可以通过HTTP协议进行实时监控访问。
针对不同的程序类型，将采用不同的性能监控方式，比如：

- Web服务

如果被监控程序是用http包启动的Web服务器，只需要引入net/http/pprof包即可，然后通过浏览器查看当前web服务器状态，比如：

```go
package main
import (
	"net/http"
	_ "net/http/pprof"
)
func hello(w http.ResponseWriter, req *http.Request) {
	w.Write([]byte("Hello"))
}
func main() {
	http.HandleFunc("/hello", hello);
	http.ListenAndServe(":8001", nil);
}
输出：
通过 http://localhost:8001/debug/pprof/ 查看结果
```

通过 goprofex 测试项目讲解说明；


#### CPU 分析（CPU profile）
再次执行 go tool pprof http://localhost:6060/debug/pprof/profile 进入交互模式，
这个 CPU profiler 默认执行30秒。它使用采样的方式来确定哪些函数花费了大多数的CPU时间。Go runtime 每10毫秒就停止执行过程并记录每一个运行中的协程的当前堆栈信息。
```
(pprof) top
63.77s of 69.02s total (92.39%)
Dropped 331 nodes (cum <= 0.35s)
Showing top 10 nodes out of 78 (cum >= 0.64s)
      flat  flat%   sum%        cum   cum%
    50.79s 73.59% 73.59%     50.92s 73.78%  syscall.Syscall
     4.66s  6.75% 80.34%      4.66s  6.75%  runtime.kevent
     2.65s  3.84% 84.18%      2.65s  3.84%  runtime.usleep
     1.88s  2.72% 86.90%      1.88s  2.72%  runtime.freedefer
     1.31s  1.90% 88.80%      1.31s  1.90%  runtime.mach_semaphore_signal
     1.10s  1.59% 90.39%      1.10s  1.59%  runtime.mach_semaphore_wait
     0.51s  0.74% 91.13%      0.61s  0.88%  log.(*Logger).formatHeader
     0.49s  0.71% 91.84%      1.06s  1.54%  runtime.mallocgc
     0.21s   0.3% 92.15%      0.56s  0.81%  runtime.concatstrings
     0.17s  0.25% 92.39%      0.64s  0.93%  fmt.(*pp).doPrintf
```

每一行表示一个函数的信息;
前两列表示函数在 CPU 上运行的时间以及百分比;
第三列是当前所有函数累加使用 CPU 的比例;
第四列和第五列代表这个函数以及子函数运行所占用的时间和比例（也被称为累加值 cumulative），应该大于等于前两列的值;
最后一列就是函数的名字;
如果应用程序有性能问题，上面这些信息应该能告诉我们时间都花费在哪些函数的执行上了。

有一个更好的方法来查看高级别的性能概况 —— web 命令，它会生成一个热点（hot spots）的 SVG 图像，可以在浏览器中打开它：

```bash
yum install graphviz
apt-get install graphviz
brew install Graphviz
```

从上图你可以看到这个应用花费了 CPU 大量的时间在 logging、测试报告（metric reporting ）上，以及部分时间在垃圾回收上。
使用 list 命令可以 inspect 每个函数的详细代码，例如 list leftpad 
```
(pprof) list leftpad
ROUTINE ======================== main.leftpad in /Users/artem/go/src/github.com/akrylysov/goprofex/leftpad.go
      20ms      490ms (flat, cum)  0.71% of Total
         .          .      3:func leftpad(s string, length int, char rune) string {
         .          .      4:   for len(s) < length {
      20ms      490ms      5:       s = string(char) + s
         .          .      6:   }
         .          .      7:   return s
         .          .      8:}
```
对无惧查看反汇编代码的人而言，可以使用 pprof 的 disasm 命令，它有助于查看实际的处理器指令：
```
(pprof) disasm chitchat
Total: 1.59mins
ROUTINE ======================== github.com/5imili/chitchat/data.Threads
      20ms      9.22s (flat, cum)  9.69% of Total
         .          .     72f290: MOVQ FS:0xfffffff8, CX                         ;thread.go:92
         .          .     72f299: LEAQ 0xffffff48(SP), AX
         .          .     72f2a1: CMPQ 0x10(CX), AX
         .          .     72f2ba: LEAQ 0x130(SP), BP
         .          .     72f2c2: MOVQ 0x2ec8df(IP), AX                   ;thread.go:93
```

#### 函数堆栈分析（Heap profile）
```bash
$ go tool pprof  http://127.0.0.1:8080/debug/pprof/heap
```
默认情况下，它显示当前正在使用的内存量：
但是我们更感兴趣的是分配的对象的数量，执行 pprof 时使用选项 -alloc_objects
```bash
$ go tool pprof -alloc_objects   http://127.0.0.1:8080/debug/pprof/heap
```
还有一些非常有用的调试内存问题的选项， -inuse_objects 可以显示正在使用的对象的数量，-alloc_space 可以显示程序启动以来分配的多少内存。
自动内存分配很便利，但世上没有免费的午餐。动态内存分配不仅比堆栈分配要慢得多，还会间接地影响性能。你在堆上分配的每一块内存都会增加 GC 的负担，并且占用更多的 CPU 资源。要使垃圾回收花费更少的时间，唯一的方法是减少内存分配。

#### 协程分析（Goroutine profile）
```bash
$ go tool pprof  http://127.0.0.1:8080/debug/pprof/gorouine
```
Goroutine profile 会转储协程的调用堆栈与运行中的协程数量

#### 阻塞分析（Block profile）
阻塞分析会显示导致阻塞的函数调用，它们使用了同步原语（synchronization primitives），如互斥锁（mutexes）和 channels 。
在执行 block contention profile 之前，你必须设置使用runtime.SetBlockProfileRate 设置 profiling rate。该标记用于控制记录Goroutine阻塞事件的时间间隔，n的单位为次，默认值为1,当传入的参数为 0 时，就相当于彻底取消记录操作。当传入的参数为 1 时，每个阻塞事件都将被记录。你可以在 main 函数或者 init 函数中添加这个调用。
```bash
go tool pprof  http://127.0.0.1:8080/debug/pprof/block
```

- 应用服务

如果被监控程序是个服务进程，使用net/http/pprof包，再额外开启一个Gorouting来运行
HTTP服务，比如：

```go
package main
import (
	"time"
	"net/http"
	_ "net/http/pprof"
)
func main() {
	go http.ListenAndServe(":6060", nil);

	for{
		time.Sleep(time.Second * 3)
	}
}
输出：
通过 http://localhost:6060/debug/pprof/ 查看结果
```
上述测试也可以通过命令行获取采样结果：

```bash
# go tool pprof http://localhost:6060/debug/pprof/profile
# go tool pprof http://localhost:6060/debug/pprof/heap
# go tool pprof http://localhost:6060/debug/pprof/block
# go tool pprof https+insecure://localhost:6060/debug/pprof/block
```

- 应用程序

如果被测试程序是应用程序，那么就不能使用net/http/pprof包了，需要使用runtime/pprof，
具体用法是：

```
pprof.StartCPUProfile 和 pprof.StopCPUProfile 监控CPU的消耗
pprof.WriteHeapProfile 监控MEM的消耗
```
```go
package main                                                                                   
import (
    "log"
    "flag"
    "os"
    "runtime/pprof"
    "sync"
    "time"
    "fmt"
)

var cpuprofile = flag.String("cpuprofile", "", "write cpu profile to file")

func sum(s []int, ch chan int, wg *sync.WaitGroup){
    defer wg.Done()
    var result int= 0
    for _, v := range s{
        result += v
    }   
    ch <- result
}

func readChan(ch chan int){
    for{
        select{
        case v,ok := <-ch:
            if ok{ 
                fmt.Println(ok,v)
            }   
        }   
    }   
}
func main() {
    flag.Parse()
    if *cpuprofile != "" {
        f, err := os.Create(*cpuprofile)
        if err != nil {
            log.Fatal(err)
        }   
        defer f.Close()
        pprof.StartCPUProfile(f)
        defer pprof.StopCPUProfile()
    }
    var wg sync.WaitGroup
    var ch = make(chan int)
    go readChan(ch)
    for i := 0;i < 100;i++{
        s := make([]int, 0)
        s = append(s,1,2,2,3,5)
        wg.Add(1)
        go sum(s,ch, &wg)
        time.Sleep(time.Second * 1)
    }   
    wg.Wait()
    close(ch)
    println("exit")
}

输出：
通过如下命令进入交互模式：
#go tool pprof [应用程序] [profile]
top命令可以看最耗时(内存)的function;各字段的含义如下：
    1. 采样点落在该函数中的次数
    2. 采样点落在该函数中的百分比
    3. 上一项的累积百分比
    4. 采样点落在该函数，以及被它调用的函数中的总次数
    5. 采样点落在该函数，以及被它调用的函数中的总次数百分比
    6. 函数名

web命令可以生成图片，更加直观分析性能
```

#### go-torch
```bash
$ go get github.com/uber/go-torch
$ git clone https://github.com/brendangregg/FlameGraph.git
cp flamegraph.pl /usr/local/bin
export PATH=$PATH:/path/to/FlameGraph
$ go-torch -u http://10.86.81.32:8080/debug/pprof/profile -t 30 -f "1.svg"
```

#### Go语言常见的profiling使用场景
```
基准测试文件：例如使用命令go test . -bench . -cpuprofile prof.cpu 生成采样文件后，再通过命令 go tool pprof [binary] prof.cpu 来进行分析。
import _ net/http/pprof：如果我们的应用是一个web服务，我们可以在http服务启动的代码文件(eg: main.go)添加 import _ net/http/pprof，这样我们的服务 便能自动开启profile功能，有助于我们直接分析采样结果。
通过在代码里面调用 runtime.StartCPUProfile 或者 runtime.WriteHeapProfile
```

改造 chitchat， 加入pprof
问题：为什么我自定义mux的http服务不能用？
比如：
```bash
# go tool pprof http://localhost:6060/debug/pprof/profile   404
```

为什么程序导入了_ "net/http/pprof"还是会404呢？
因为导入pprof的时侯只是调用了pprof包的init函数，看看 init 里面的内容,pprof.init()，
可以得知，这里注册的所有路由都是在DefaultServeMux下的， 所以当然我们自定义的mux是没有用的，
要使它有用也很简单，我们自己手动注册路由与相应的处理函数。

```go
func main() {
	// 启动一个自定义mux的http服务器
	mux := http.NewServeMux()
	mux.HandleFunc("/debug/pprof/", pprof.Index)
	mux.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
	mux.HandleFunc("/debug/pprof/profile", pprof.Profile)
	mux.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
	mux.HandleFunc("/debug/pprof/trace", pprof.Trace)
	http.ListenAndServe(":6066", mux)
}
```

#### 性能优化
Logging：
让应用运行更快，一个很好又不是经常管用的方法是，让它执行更少的工作。除了 debug 的目的之外，这行代码 log.Printf("%s request took %v", name, elapsed) 在 web service 中不需要。所有非必要的 logs 应该在生产环境中被移除代码或者关闭功能。可以使用分级日志（a leveled logger）来解决这个问题，比如这些很棒的 日志工具库（logging libraries）
关于打日志或者其他一般的 I/O 操作，另一个重要的事情是尽可能使用有缓冲的输入输出（buffered input/output），这样可以减少系统调用的次数。通常，并不是每个 logger 调用都需要立即写入文件 —— 使用 bufio package 来实现 buffered I/O 。我们可以使用 bufio.NewWriter 或者 bufio.NewWriterSize 来简单地封装 io.Writer 对象，再传递给 logger:
比如：
```go
func init() {
    log.SetOutput(bufio.NewWriterSize(f, 1024*16))
}
```
```text
    用 --inuse_space来分析程序常驻内存的占用情况;
    用 --alloc_objects来分析内存的临时分配情况，可以提高程序的运行速度。
```
leftpad：
在每一个循环中连接字符串的做法并不高效，因为每一次循环迭代都会分配一个新的字符串（反复分配内存空间）。有一种更好的方法来构建字符串，使用 bytes.Buffer，比如：
```go
func leftpad(s string, length int, char rune) string {
    buf := bytes.Buffer{}
    for i := 0; i < length-len(s); i++ {
        buf.WriteRune(char)
    }
    buf.WriteString(s)
    return buf.String()
}
```
另外，我们还可以使用 string.Repeat ，使代码更加简洁，比如：
```go
func leftpad(s string, length int, char rune) string {
    if len(s) < length {
        return strings.Repeat(string(char), length-len(s)) + s
    }
    return s
}
```
StatsD client:
接下来需要优化的代码是 StatsD.Send 函数,以下是有一些可能的值得改进的地方：
```
    1. Sprintf 对字符串格式化非常便利，它性能表现很好，除非你每秒调用它几千次。不过，它把输入参数进行字符串格式化的时候会消耗 CPU 时间，而且每次调用都会分配一个新的字符串。为了更好的性能优化，我们可以使用 bytes.Buffer + Buffer.WriteString/Buffer.WriteByte 来替换它。
    2. 不需要每一次都创建一个新的 Replacer 实例，它可以声明为全局变量，或者作为 StatsD 结构体的一部分。
    3. 用 strconv.AppendFloat 替换 strconv.FormatFloat ，并且使用堆栈上分配的 buffer 来传递变量，防止额外的堆分配。
```
示例如下：
```go
var reservedReplacer = strings.NewReplacer(":", "_", "|", "_", "@", "_")
func (s *StatsD) Send(stat string, kind string, delta float64) {
    buf := bytes.Buffer{}
    buf.WriteString(s.Namespace)
    buf.WriteByte('.')
    buf.WriteString(reservedReplacer.Replace(stat))
    buf.WriteByte(':')
    buf.Write(strconv.AppendFloat(make([]byte, 0, 24), delta, 'f', -1, 64))
    buf.WriteByte('|')
    buf.WriteString(kind)
     if s.SampleRate != 0 && s.SampleRate < 1 {
        buf.WriteString("|@")
        buf.Write(strconv.AppendFloat(make([]byte, 0, 24), s.SampleRate, 'f', -1, 64))
    }
    buf.WriteTo(ioutil.Discard)
}
```
这样做，将分配数量（number of allocations）从15减少到1个，并且使 Send 运行快了 5 倍。
- 测试优化结果

通过如下方式进行基准测试:
```bash
go test -run=NONE -bench=. -benchmem ./... > old.txt
# make changes
# git stash and git stach pop
go test -run=NONE -bench=. ./... > new.txt
benchcmp old.txt new.txt
```
strconv 基本包括四类函数：
```
    1. Append 类，例如 AppendBool(dst []byte, b bool)[]byte，将值转化后添加到[]byte的末尾
    2. Format 类，例如 FormatBool(b bool) string，将bool  float int uint 类型的转换为string，FormatInt的缩写为Itoa
    3. Parse 类，例如 ParseBool(str string)(value bool, err error) 将字符串转换为 bool float int uint类型的值，err指定是否转换成功,ParseInt 的缩写是 Atoi
    4. Quote 类，对字符串的 双引号 单引号 反单引号 的操作
```
比如：
```go
func main() {
	fmt.Println(string(strconv.AppendBool(make([]byte, 0, 24), true)))
	fmt.Println(string(strconv.FormatBool(true)))
	b, _ := strconv.ParseBool("true")
	fmt.Println(reflect.TypeOf(b), b)
	s := "Gopher 你好"
	fmt.Println(s)
	fmt.Println(string(strconv.AppendQuote(make([]byte, 0, 24), s)))
}
```
#### 优化技巧
```txt
    1. 避免不必要的 heap 内存分配。
    2. 对于不大的结构体，值传参比指针传参更好。
    3. 如果你事先知道长度，最好提前分配 maps 或者 slice 的内存。
    4. 生产环境下，非必要情况不打日志。
    5. 如果你要频繁进行连续的读写，请使用缓冲读写（buffered I/O）
    6. 如果你的应用广泛使用 JSON，请考虑使用解析器/序列化器 (parser/serializer generators) (easyjson)
    7. 在主要路径上的每一个操作都很关键（Every operation matters in a hot path）
```

#### Golang的性能调优手段
Go语言内置的CPU和Heap profiler
使用Go的profiler我们能获取以下的样本信息：
CPU profiles
Heap profiles
block profile、traces等

https://studygolang.com/articles/12685?fr=sidebar