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

Go语言工具链和标准库提供了单元测试框架，给开发人员带来很大遍历，但也要按照如下需求编写测试代码，比如：

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
#go test -v  //测试当前包和所有子包，用go test -v ./...
```
testing.T 提供了控制测试结果和行为的方法，常用的如下，比如：

```
Fail: 失败后继续执行当前测试函数
FailNow: 失败后立即终止执行当前测试函数
SkipNow: 跳过执行当前测试函数
Log: 报告错误信息，仅当失败或-v时输出
Parallel: 与有同样设置的测试函数并行执行
Error: 报告错误信息后继续，相当于Log + Fail
Fatal: 报告错误信息后停止，相当于FailNow + Log
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
| -count | 指定重复测试次数| -count 3m10s |


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
准确计算性能，此时可以用benchtime选项来设定最小测试时间，增加循环次数，进而返回更准确的结果，比如：

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
输出结果包含单次测试分配的内存总量和次数
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
可以指定covermode和coverprofile参数,获取更详细测试信息，比如:

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
Go语言提供了pprof包来做代码的性能监控，可以使用pporf静态分析上面的性能测试结果，并且也可以
进行实时性能监控，pprof在Go标准库中存在于两个路径下：

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
