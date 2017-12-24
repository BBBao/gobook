# 函数

```text
1. 定义
2. 参数
3. 返回值
4. 匿名函数
5. 延迟调用
6. 错误处理 
```

- 定义

函数是结构化编程中最小的模块单元，日常开发过程中，将复杂的算法过程分解为若干个小任务，使程序的结构性更清晰，程序可读性提升，易于后期维护和让别人读懂你的代码。
另外可以把重复性的任务抽象成一个函数，可以更好的重用你的代码。

Go语言中使用关键词func来定义一个函数，并且左花括号不能另起一行，比如：

```go
func hello(){   //左花括号不能另起一行
    println("hello")
}
```
Go语言中定义和应用函数时，有如下几点需要注意的点：

```
 函数无须前置声明
 不支持命名嵌套定义，支持匿名嵌套
 函数只能判断是否为nil，不支持其它比较操作
 支持多返回值
 支持命名返回值
 支持返回局部变量指针
 支持匿名函数和闭包
```

```go
func hello()
{               //左括号不能另起一行
}

func add(x,y int) (sum int){    //命名返回值
    sum = x + y
    return
}

func vals()(int,int){       //支持多返回值
    return 2,3
}

func a(){}
func b(){}

func add(x,y int) (*int){      //支持返回局部变量指针
    sum := x + y
    return &sum
}

func main(){
    println(a==b)       //只能判断是否为nil，不支持其它比较操作
    func hello() {      //不支持命名嵌套定义
        println("hello")
    }
}
```


具备相同签名(参数和返回值)的函数才视为同一类型函数，比如：

```go
func hello() {
    fmt.Println("hello")
}
func say(f func()){                                                                                                                                   
    f()
}

func main(){
    f := hello
    say(f)
}
```

- 参数
Go语言中给函数传参时需要注意以下几点：

```
 不支持默认参数
 不支持命名实参
 参数视作为函数的局部变量
 必须按签名顺序传递指定类型和数量的实参
 相邻的同类型参数可以合并
 支持不定长变参，实质上是slice
```

```go
func test(x,y int, s string, _ bool){   //相邻的同类型参数可以合并
    return
}

func add(x ,y int)int{  //参数视作为函数的局部变量
    x := 100            //no new variables on left side of :=
    var y int = 200     //y redeclared in this block
    return x +y
}

func main(){
    //test(1,2,"s")       //not enough arguments in call to test                                                                                                                             
    test(1,2,"s",false)
}
```
不管传递的是指针、引用还是其它类型参数，都是值拷贝传递的，区别在于拷贝的目标是目标对象还是拷贝指针而已。
在函数调用之前，编译器会为形参和返回值分配内存空间，并将实参拷贝到形参内存。比如：

```go
func test1(x *int){
    fmt.Printf("%p, %v\n",&x ,x)
}

func main(){
    a := 0x100
    p := &a
    fmt.Printf("%p, %v\n", &p, p)
    test1(p)
}
输出：
0xc42002c020, 0xc42000a320
0xc42002c030, 0xc42000a320
从结构中看出， 实参和形参指向同一目标，但是传递的指针是被赋值了的
```
如果函数参数过多，可以将参数封装成一个结构体类型，比如：

```go
type serverOption struct{
    addr    string
    port int 
    path    string
    timeout time.Duration
}

func newOption() * serverOption{
    return &serverOption{
        addr:"127.0.0.1",
        port:8080,
        path:"/var/www",
        timeout: time.Second * 5,
    }   
}
func (s *serverOption)server(){
    println("run server")
}

func main(){
    s := newOption()
    s.port = 80
    s.server()
    for{}                                                                                                                                             
}
```


- 变参
变参本质上是一个切片(slice)，只能接收一到多个同类型参数，且必须放在参数列表尾部，比如：

```go
func add(args ...int) int {
  total := 0
  for _, v := range args {
    total += v
  }
  return total
}
func main() {
  fmt.Println(add(1,2,3))
}
```

实质上是切片，那是否可以直接传个切片或数组呢？
```go
func test1(s string, a ...int){
    fmt.Printf("%T, %v\n", a, a)    //[]int, [1 2 3 4]
}
{
    a := [4]int{1,2,3,4}
    test1("s", a)     //cannot use a (type [4]int) as type int in argument to test1                                                                                                                               
    test1("s", a[:]     //cannot use a[:] (type []int) as type int in argument to test1
    test1("s", a[:]...) //切片展开
}
```

变参既然是切片，那么参数复制的是切片的本身，并不包括底层的数组，因此可以修改原数据，
也可以copy底层数据，防止原数据被修改，比如：

```go
func test1(a ...int){
    for i := range a{
        a[i] += 100
    }
}
func main(){
    a := [4]int{1,2,3,4}
    // b := make([]int,0)
    // copy(b,a[:])         
    // test1(b[:]...)
    test1(a[:]...)                                                                                                                                                 
    for i := range a{
        fmt.Println(a[i])
    }
}
```
- 匿名函数
匿名函数就是没有定义函数名称的函数。我们可以在函数内部定义匿名函数，也叫函数嵌套。
匿名函数可以直接被调用，也可以赋值给变量、作为参数或返回值。比如：

```go
func main(){
    func(s string){     //直接被调用
        println(s)
    }("hello gopher!!!")
    /*
    func(s string){     //未被调用
        println(s)
    }
    */
}

func main(){
    hi := func(s string){   //赋值给变量
        println(s)
    }                                                                                                                                                 
    hi("hello gopher!!!")
}

func test(f func(string)){
    f("hello gopher!!!")
}
func main(){
    hi := func(s string){
        println(s)
    }
    test(hi)     //作为参数                                                                                                                                     
}

func test()func(string){
    hi := func(s string){   //作为返回值
        println(s)
    }
    return hi
}

func main(){
    f := test()                                                                                                                                       
    f("hello gopher!!!")
}
```
普通函数和匿名函数都可以作为结构体的字段，比如：

```go
{
    type calc struct{
        mul func(x,y int)int
    }
    x := calc{
        mul: func(x,y int)int{                                                                                                                        
            return (x*y)
        },
    }
    println(x.mul(2,3))
}
```
也可以经通道传递，比如：

```go
{
    c := make(chan func(int, int)int, 2)
    c <- func(x,y int) int {return x + y}
    println((<-c)(2,3))
}
```

- 闭包(closure)
是指在上下文中引用了自由变量(未绑定到特定对象)的代码块(函数)，或者说是代码块(函数)与和引用环境的组合体。

```go
func intSeq()func()int{
    i := 0
    println(&i)
    return func()int{
        i += 1
        println(&i,i)
        return i
    }   
}
                                                                                                                                                      
func main(){
    nextInt := intSeq()
    fmt.Println(nextInt())
    fmt.Println(nextInt())
    fmt.Println(nextInt())

    newInt := intSeq()
    fmt.Println(newInt())
}
输出：
0xc42000a320
0xc42000a320 1
1
0xc42000a320 2
2
0xc42000a320 3
3
0xc42007a010
0xc42007a010 1
1
```
当nextInt函数返回后，通过输出指针，我们可以看出函数在main运行时，依然引用的是原环境变量指针，这种现象称作闭包。
所以说，闭包是函数和引用环境变量的组合体。正因为闭包是通过指针引用环境变量，那么就会导致该变量的生命周期
变长，甚至被分配到堆内存。如果多个匿名函数引用同一个环境变量，会让事情变得更加复杂，比如：

```go
func test()[]func(){
    var s []func()
    for i:= 0;i < 3;i++{
        s =  append(s, func(){
            println(&i , i)
        })
    }
    return s
}
func main(){
    funcSlice := test()
    for _ , f := range funcSlice{
        f()
    }
}
输出：
0xc42000a320 3
0xc42000a320 3
0xc42000a320 3
```
解决方法就是每次用不同的环境变量或参数赋值，比如修改后的test函数：

```go
func test()[]func(){
    var s []func()
    for i:= 0;i < 3;i++{
        x := i
        s =  append(s, func(){
            println(&x , x)                                                                                                                           
        })
    }
    return s
}
```
闭包在不用传递参数的情况下就可以读取和修改环境变量，当然我们是要为这种遍历付出代价的，所以日常开发中，在高并发服务
的场景下建议慎用，除非你明确你的需求必须这样做。

- 延迟调用
Go语言提供defer关键字，用于延迟调用，延迟到当函数返回前被执行，多用于资源释放、解锁以及错误处理等操作。

```go
func main() {
    f, err := createFile("defer.txt")
    if err != nil {
        fmt.Println(err.Error())
        return
    }   
    defer closeFile(f)
    writeFile(f)
}

func createFile(filePath string) (*os.File, error) {
    f, err := os.Create(filePath)
    if err != nil {
        return nil, err 
    }   
    return f, nil 
}

func writeFile(f *os.File) {
    fmt.Println("write file")
    fmt.Fprintln(f, "hello gopher!")
}

func closeFile(f *os.File) {
    fmt.Println("close file")
    f.Close()
}
```

如果一个函数内引用了多个defer，它们的执行顺序是怎么样的呢？

```go
package main

func main() {
	defer println("a")
	defer println("b")
}
输出：
b
a
```
如果函数中引入了panic函数，那么延迟调用defer会不会被执行呢？

```go
func main() {
    defer println("a")
    panic("d")                                                                                                                                        
    defer println("b")
}
```
日常开发中，一定要记住defer是在函数结束时才被调用的，如果应用不合理，可能会造成资源浪费，给gc带来压力，甚至
造成逻辑错误，比如：

```go
func main() {
    for i := 0;i < 10000;i++{
        filePath := fmt.Sprintf("/data/log/%d.log", i)
        fp, err := os.Open(filePath)
        if err != nil{
            continue
        }
        defef fp.Close()    //这是要在main函数返回时才会执行的，不是在循环结束后执行，延迟调用，导致占用资源
        //do stuff...
    }
}

修改方案是直接调用Close函数或将逻辑封装成独立函数：
func logAnalisys(p string){
    fp, err := os.Open(p)
    if err != nil{
        continue
    }
    defef fp.Close()
    //do stuff
}

func main() {
    for i := 0;i < 10000;i++{
        filePath := fmt.Sprintf("/data/log/%d.log", i)
        logAnalisys(filePath)
    }
}
```
在性能方面，延迟调用花费的代价也很大，因为这个过程包括注册、调用等操作，还有额外的内存开销。比如：

```
package main

import "testing"
import "fmt"
import "sync"

var m sync.Mutex

func test(){
	m.Lock()
	m.Unlock()
}

func testCap(){
	m.Lock()
	defer m.Unlock()
}

func BenchmarkTest(t *testing.B){
	for i:= 0;i < t.N; i++{
		test()
	}
}

func BenchmarkTestCap(t *testing.B){
	for i:= 0;i < t.N; i++{
		testCap()
	}
}

func main(){
	resTest := testing.Benchmark(BenchmarkTest)
	fmt.Printf("BenchmarkTest \t %d, %d ns/op,%d allocs/op, %d B/op\n", resTest.N, resTest.NsPerOp(), resTest.AllocsPerOp(), resTest.AllocedBytesPerOp())
	resTest = testing.Benchmark(BenchmarkTestCap)
	fmt.Printf("BenchmarkTestCap \t %d, %d ns/op,%d allocs/op, %d B/op\n", resTest.N, resTest.NsPerOp(), resTest.AllocsPerOp(), resTest.AllocedBytesPerOp())
}
输出：
BenchmarkTest 	 50000000, 27 ns/op,0 allocs/op, 0 B/op
estCap 	 20000000, 112 ns/op,0 allocs/op, 0 B/op
```
在要求高性能的高并发场景下，应避免使用延迟调用。
- 错误处理
Go语言标准库将将error定义为接口类型，以便实现自定义错误类型。比如：

```go
type error interface{
    Error() string
}
```
按照Go语言编程习惯，error总是最后一个函数返回值，并且标准库提供了创建函数，可以方便的
创建错误消息的error对象。比如：

```go
func divTest(x ,y int)(int, error){
    if y == 0{
        return 0, errors.New("division by zero")
    }   
    return x/y,nil
}
func main(){
    v, err := divTest(3,0)
    if err != nil{
        log.Fatalln(err.Error())
    }   
    println(v)                                                                                                                                        
}
```
日常开发中，我们需要根据需求自定义错误类型，可以存放更多的上下文信息，或者跟觉错误类型做出相应的错误处理。比如：

```go
type NegativeError struct{
    x, y int
}
func (NegativeError)Error()string{
    return "negative value error"
}

type MolError struct {
    x, y int
}
func (MolError)Error()string{
    return "devision by zero"
}

func molTest(x ,y int)(int, error){
    if y == 0{
        return 0, MolError{x,y}
    }
    if x < 0 || y < 0{
        return 0, NegativeError{x,y}
    }
    return x%y,nil
}
func main(){
    v, err := molTest(3,-1)
    if err != nil{
        switch e := err.(type){     //获取错误类型
            case MolError:
                println(e.x,e.y)
            case NegativeError:
                println(e.x,e.y)
            default:
                println(e)
        }
        log.Fatalln(err.Error())
    }
    println(v)
}
```

与error相比，panic/recover在应用上更类似于try/catch结构化。

```go
func panic() interface{}   
func recover() interface{}
```
panic 立即中断但前函数处理流程，执行延迟调用。recover在延迟调用中可以捕获并返回panic产生的错误对象，比如：

```go
func Myrecover(){
    if err := recover(); err != nil{
        log.Fatalln(err)
    }   
}

func main(){
    println("start...")
    defer Myrecover()
    panic("dead")                                                                                                                                     
    println("end...")
}
```
如果有连续多次调用panic的场景，只有最后一次panic会被recover捕获处理，比如：

```go
func Myrecover(){
    if err := recover(); err != nil{
        log.Fatalln(err)
    }   
}

func main(){
    defer Myrecover()
    defer func(){
        panic("a bad problem")
    }()
    panic("a problem")
}
```
recover只有在延迟调用函数中才能得到正常工作，比如：

```go
func main() {
    defer Myrecover()
    defer log.Println(recover())
    defer println(recover())
    panic("a problem") 
}
输出：
(0x0,0x0)
2016/11/12 07:07:54 <nil>
2016/11/12 07:07:54 a problem
exit status 1
```
在日常开发过程中，经常需要进行调试，可以使用函数输出完整的调用栈信息，比如：

```go
func Myrecover(){
    if err := recover(); err != nil{
        fmt.Println(err)
        debug.PrintStack()
        //log.Fatalln(err)
    }
}
func main(){
    defer Myrecover()
    panic("a problem")
}
输出：
a problem
goroutine 1 [running]:
runtime/debug.Stack(0xc42002c010, 0xc42003fe20, 0x1)
	/root/data/go/src/runtime/debug/stack.go:24 +0x79
runtime/debug.PrintStack()
	/root/data/go/src/runtime/debug/stack.go:16 +0x22
main.Myrecover()
	/root/data/gopath/test/panic.go:10 +0x85
panic(0x48a5e0, 0xc42000a320)
	/root/data/go/src/runtime/panic.go:458 +0x243
main.main()
	/root/data/gopath/test/panic.go:16 +0x8d
```
日常开发中，只有在系统发生了不可恢复性或无法正常工作的错误可以使用panic，比如端口号被占用、数据库未启动、文件系统错误等，否则不建议使用。

