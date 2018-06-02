
#### 变量初始化
如下代码是否有问题？为什么？
```go
type gopher struct{
	name string
	age int
}
func main() {
	var g = gopher{}
	g.name, age := "test", 19
	fmt.Println(g)
}
```

#### while
如下代码是否有问题？ 如何修正?
```go
func main() {
	while true {
		println("hello gopher")
	}
}
```

#### rune 
如下代码输出是？为什么? 
```go
func main(){
	s := "Gopher你好"
	for i := 0; i < len(s); i++ {
		fmt.Println(string(s[i]))
	}
}
```


#### Itoa
下列代码中a,b,c,d,e,f的值分别是多少？为什么？
```go
const (
	a = iota
	b
	_
	c
	_
	d
)
```
```golang
const (
	a, b = iota + 1, iota + 2
	c, d
	e, f
)
```
```go
const (
	a = iota
	b = 5
	c
	d = iota
	e
)
```
最佳示例
```go
type ByteSize float64
const (
    _           = iota                   // ignore first value by assigning to blank identifier
    KB ByteSize = 1 << (10 * iota) // 1 << (10*1)
    MB                                   // 1 << (10*2)
    GB                                   // 1 << (10*3)
    TB                                   // 1 << (10*4)
    PB                                   // 1 << (10*5)
    EB                                   // 1 << (10*6)
    ZB                                   // 1 << (10*7)
    YB                                   // 1 << (10*8)
)
```

#### string
下面代码是否正确?为什么?
```golang
func main() {
	s := "hello"
	s[0] = 'c'
	fmt.Printf("%v\n", s)
}
```

#### slice
如下代码的输出是什么？
```go
func testSlice() {
	s := make([]int, 3)
	s = append(s, 1, 2, 3)
	fmt.Println(s)
}
```

#### foreach
如何把gopher结构体数组存入到Map中,并正确输出?
```golang
type gopher struct {
	Name string
	Age  int
}
func main(){
	var goMap = make(map[string]*gopher)
	var gophers = []gopher{
		{Name: "wang", Age: 12},
		{Name: "li", Age: 10},
		{Name: "zhao", Age: 11},
	}
	... //???
}
```

#### break、continue、goto与lable
如何快速跳出外部循环1 ?
```go
func loopTest(){
	for i := 0; i < 10; i++ {      // 1
		for j := 0; j < 10; j++ {	// 2
			for z := 0; z < 10; z++ {	// 3
				if z == 1 {
					break 
				}
				fmt.Println(i, j, z)
			}
			/*...*/
		}
		/*...*/
	}
	out := "byebye"
	fmt.Println(out)
}
```
练习移动Lable的位置，总结出什么经验？
下面代码输出什么？
```go
i := 0
LOOP:
	print(i)
	i++
	if i == 5 {
		return
	}
	goto LOOP
```
练习下面代码
```go
for i := 0; i < 10; i++ {
		for j := 0; j < 10; j++ {
			for z := 0; z < 10; z++ {
				if z == 1 {
					goto END
				}
				fmt.Println(i, j, z)
			}
		}
	}
	out := "byebye"
END:
	fmt.Println(out)
```
能否把continue换成goto？
```go
func lableTest() {
LABLE:
	for i := 0; i < 10; i++ {
		fmt.Println(i)
		continue LABLE
	}
}
```
goto传统的用法
```go
func gototest(){
	 if (!doA())
        goto exit;
    if (!doB())
        goto cleanupA;
    if (!doC())
        goto cleanupB;

    /* everything has succeeded */
    return;	
cleanupB:
    undoB();
cleanupA:
    undoA();
exit:
    return;
}
```
总结：
```
	- 三个语法都可以配合标签使用
	- 标签名区分大小写，若不使用会造成编译错误
	- Break与continue配合标签可用于多层循环的跳出,并且标签只能放置在for循环前面
	- Goto是调整执行位置，与其它2个语句配合标签的结果并不相同
```

#### select
如下代码会不会引发异常?为什么?
```go
func testpanic() {
	chanInt := make(chan int, 1)
	chanString := make(chan string, 1)
	chanInt <- 1
	chanString <- "hello"
	select {
	case val := <-chanInt:
		println(val)
	case val := <-chanString:
		panic(val)
	}
}
```
#### defer、panic、recover
下面代码的输出结果是什么?
```go
func main() {
	defer_call()
}
func defer_call() {
	defer func() { fmt.Println("1") }()
	defer func() { fmt.Println("2") }()
	defer func() { fmt.Println("3") }()
	panic("触发异常")
}
```
如何使用recover? 能否输出当前栈信息？
```golang
func recover_call() {
	defer func() {
		if r := recover(); r != nil {
			fmt.Println("panic: %v", r)
		}
	}()
	defer_call()
}
```
练习： 递归循环调用一个函数到第N次，通过panic异常退出函数,并捕获该异常
总结:
```
	- 执行方式类似其它语言中的析构函数，在函数体执行结束后
	- 按照调用顺序的相反顺序逐个执行
	- 即使函数发生严重错误也会执行
	- 支持匿名函数的调用
	- 常用于资源清理、文件关闭、解锁以及记录时间等操作
	- 通过与匿名函数配合可在return之后修改函数计算结果
	- 如果函数体内某个变量作为defer时匿名函数的参数，则在定义defer时即已经获得了拷贝，否则则是引用某个变量的地址
	- Go 没有异常机制，但有 panic/recover 模式来处理错误
	- Panic 可以在任何地方引发，但recover只有在defer调用的函数中有效
```

#### Go语言变量堆栈存储方式 

下面两个函数中的变量a，是堆存储还是栈存储?
```go
func test1(){
	var a [1]int
	c := a[:]
	fmt.Println(c)
}
func test2(){
	var a [1]int
	c := a[:]
	println(c)
}
``` 
下面的代码中x,y的变量是如何存储的?
```go
var global *int
func f() {
    var x int
    x = 1
    global = &x
}

func g() {
    y := new(int)
    *y = 1
}
```

#### function
下列代码能否编译通过?
```go
func testFunc(x,y int)(sum int,error){
	return x+y, nil
}
```

#### new
下面代码能否编译通过?
```go
func main() {
	list := new([]int)
	list = append(list, 1)
	fmt.Println(list)
}
```

#### const
下面代码能否编译通过?
```go
const cl = 100
var c2 = 123
func main() {
	println(&bl, bl)
	println(&cl, cl)
}
```

#### map
map 是线程安全的么？
```go
type gopher struct {
	stus map[string]int
}

func (g *gopher) set(name string, age int) {
	g.stus[name] = age
}

func (g *gopher) get(name string) int {
	return g.stus[name]
}
func main() {
	var g = &gopher{
		stus: make(map[string]int),
	}
	go g.set("smile", 3)
	go g.set("smile1", 3)
	go g.set("smile2", 3)
	go g.get("smile2")
	go g.get("smile2")
	time.Sleep(3 * time.Second)
}
```
