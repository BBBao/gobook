# 数据
```
1. 字符串
2. 数组
3. 切片
4. 字典
5. 结构
```

#### 字符串

Go语言中的字符串是由一组不可变的字节(byte)序列组成，从源码文件中看出其本身是一个复合结构：

```go
string.go 
type stringStruct struct {
	str unsafe.Pointer
	len int
}
```
> 字符串中的每个字节都是以UTF-8编码存储的Unicode字符，字符串的头部指针指向字节数组的开始，但是没有NULL或'\0'结尾标志。
> 表示方式很简单，用双引号("")或者反引号(``)，它们的区别是：
>
> 1. 双引号之间的转义符会被转义，而反引号之间的转义符保持不变
> 2. 反引号支持跨行编写，而双引号则不可以

```go
{
    println("hello\tgo")    //输出hello	go
    println(`hello\tgo`)    //输出hello\tgo
}

{
    println( "hello 
        go" )                   //syntax error: unexpected semicolon or newline, expecting comma or )

    println(`hello                                                                                                                                                                                                 
        go`)
}
输出：
hello 
	go
```

在前面类型的章节中描述过字符串的默认值是""，而不是nil,比如：

```go
var s string
println( s == "" )  //true
println( s == nil ) //invalid operation: s == nil (mismatched types string and nil)
```
Go字符串支持 "+ , += , == , != , < , >" 六种运算符

Go字符串允许用索引号访问字节数组(非字符)，但不能获取元素的地址：比如

```go
{
    var a = "hello"
    println(a[0])       //输出 104
    println(&a[1])      //cannot take the address of a[1]
}
```
Go字符串允许用切片的语法返回子串(起始和结束索引号)

```go
    var a = "0123456"                                                                                                                                                                                              
    println(a[:3])      //0,1,2
    println(a[1:3])     //1,2
    println(a[3:])      //3,4,5,6
```

日常开发中，经常会有遍历字符串的场景，比如：

```
{
    var a = "Go语言"
    for i:=0;i < len(a);i++{                //以byte方式按字节遍历
        fmt.Printf("%d: [%c]\n", i, a[i])
    }
    for i, v := range a{                    //以rune方式遍历                                                                                               
        fmt.Printf("%d: [%c]\n", i, v)
    }
}
输出：    
0: [G]
1: [o]
2: [è]
3: [¯]
4: [­]
5: [è]
6: [¨]
7: []
0: [G]
1: [o]
2: [语]
5: [言]
```
在Go语言中，字符串的底层使用byte数组存的，并且是不可以改变的，所以byte方式每次得到的只有一个byte，而中文字符是占3个byte的。
rune采用计算字符串长度的方式与byte方式不同，比如：

```go
println(utf8.RuneCountInString(a)) // 结果为  4
println(len(a)) // 结果为 8
```
所以如果想要获得期待的那种结果的话，需要先将字符串a转换为rune切片，再使用内置的len函数，比如:

```go
{
    r := []rune(a)
    for i:= 0;i < len(r);i++{
        fmt.Printf("%d: [%c]\n", i, r[i])
    }
}
```
所以，在遍历或处理的字符串的情况下，如果其中存在中文，尽量使用rune方式处理。

- 转换
前面讲过不能修改原字符串，如果修改的话需要将字符串转换成[]byte或[]rune , 然后在转换回来，比如：

```go
{
    var a = "hello go"                                                                                                       
    a[1] = 'd'              //cannot assign to a[1]
}
{
    var a = "hello go"
    bs := []byte(a)
    ...                                                                                                                                                                                                            
    s2 := string(bs)

    rs := []rune(a)
    ...
    s3 := string(rs)
}
```

Go语言支持用"+"运算符进行字符串拼接，但是每次拼接都需要重新分配内存，如果频繁构造一个很长的字符串，
则性能影响就会很大，比如：

```go
func test1()string{
    var s string
    for i:= 0;i < 1000 ;i++{
        s += "a" 
    }   
    return s
}

func Benchmark_test1(b *testing.B){
    for i:= 0;i < b.N; i++{
        test1()                                                                                                                                                                                                    
    }   
}
输出：
# go test str1_b_test.go  -bench="test1" -benchmem
Benchmark_test1-2   	    5000	    227539 ns/op	  530338 B/op	     999 allocs/op
```

常用的改进方法是预分配足够的内存空间，然后使用strings.Join函数，
该函数会统计出所有参数的长度，并一次性完成内存分配操作，改进一下上面的代码：

```go
func test()string{
    s := make([]string,1000)
    for i:= 0;i < 1000 ;i++{
        s[i] = "a" 
    }   
    return strings.Join(s,"")
}
func Benchmark_test(b *testing.B){
    for i:= 0;i < b.N; i++{
        test()
    }   
}
输出：
# go test -v b_test.go  -bench="test1" -benchmem
Benchmark_test1-2   	  200000	     10765 ns/op	    2048 B/op	       2 allocs/op
```
在日常开发中，可以使用fmt.Sprintf函数来格式化和拼接较少的字符串操作，比如：

```go
{
    a := 10010
    as := fmt.Sprintf("%d",a)
    fmt.Printf("%T , %v\n",as,as)
}
```

#### 数组
数组是内置(build-in)类型，是一组存放相同类型数据的集合，数组的数据类型是由存储的元素类型和数组的长度共同决定的,
初始化之后长度是固定无法修改的，数组也支持逻辑判断运算符 ”==“， ”！=“，定义方式如下：

```go
{
    var a [10]int
    var b [20]int
    println(a == b)         //invalid operation: a == b (mismatched types [10]int and [20]int)
}
```
即使元素类型相同，但是长度不同数组，也不属于同一类型。

数组的初始化相对灵活，下标索引值从0开始，支持按索引位置初始化，对于未初始化的数组，编译器将给以默认值。

```go
{
    var a[4] int                //元素初始化为0
    b := [4] int{0,1}           //未初始化的元素将被初始化为0
    c := [4] int{0, 2: 3}       //可指定索引位置初始化
    d := [...]int{0,1,2}        //编译器根据初始化值数量来确定数组的长度
    e := [...]int{1, 3:3}       //支持索引位置初始化，但数组长度与其无关
    
    type user struct{
        name string
        age int 
    }   

    d := [...] user{            //复合数据类型数组可省略元素初始化类型标签
        {"a",1},
        {"b",2},
    }
}
```
定义多维数组时，只有数组的第一维度允许使用 "..."

```go
    x := [2]int{2,2}
    a := [2][2]int{{1,2},{2,2}}
    b := [...][2]int{{2,3},{2,2},{3,3}}
    c := [...][2][2]int{{ {2,3},{2,2} },{{3,3},{4,4}} }
```
计算数组长度时，无论使用内置的len还是cap，返回的都是第一维度的长度，比如：

```go
    fmt.Println(x, len(x), cap(x))    
    fmt.Println(a, len(a), cap(x))
    fmt.Println(b, len(b), cap(x))
    fmt.Println(c, len(c), cap(x))
输出：
[2 2] 2 2
[[1 2] [2 2]] 2 2
[[2 3] [2 2] [3 3]] 3 2
[[[2 3] [2 2]] [[3 3] [4 4]]] 2 2
```
- 数组指针&指针数组

数组除了可以存放具体类型的数据，也可以存放指针，比如：

```go
{ 
    x, y := 10, 20
    a := [...]*int{&x, &y}      //指针数组 
    p := &a                     //数组的指针
}
```
- 数组复制

Go语言数组是值(非引用)类型，所以在赋值和参数传递过程中都会复制整个数组数据，比如:

```go
func test(x [2]int){
    fmt.Printf("x:= %p,%v\n", &x, x)
}

func main(){ 
    a := [2] int{1, 2}
    test(a)                     //传参过程中完全复制
    var b [2]int
    b = a                       //赋值过程中完全复制
    fmt.Printf("a:= %p,%v\n", &a, a)
    fmt.Printf("b:= %p,%v\n", &b, b)                                                                                                                                                                               
}
输出：
x:= 0xc42000a330,[1 2]
a:= 0xc42000a320,[1 2]
b:= 0xc42000a370,[1 2]
```

#### 切片(slice)
在日常开发中，更多的场景是是需要一个可以动态更新长度的数据存储结构，切片本身并非是动态数组或数组指针，
 他内部通过指针引用底层数组，并设定相关属性将数据读写操作限定在指定区域内。

```go
/runtime/slice.go

type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```
- 切片初始化

切片有两种基本初始化方式：
> 切片可以通过内置的make函数来初始化，初始化时len=cap，一般使用时省略cap参数，默认和len参数相同，在追加元素时，如果容量cap不足时，将按len的2倍动态扩容。
> 通过数组来初始化切片，以开始和结束索引位置来确定最终所引用的数组片段

```go
//make([]T, len, cap) //T是切片的数据的类型，len表示切片的长度，cap表示capacity
{
    s := make([]int,5)      //len: 5  cap: 5
    s := make([]int,5,10)    //len: 5  cap: 10
    s := []int{1,2,3}
}                           

{
    arr := [...]int{0,1,2,3,4,5,6,7,8,9}
    s1 := arr[:]
    s2 := arr[2:5]
    s3 := arr[2:5:7]
    s4 := arr[4:]
    s5 := arr[:4]
    s6 := arr[:4:6]

    fmt.Println("s1: ",s1, len(s1),cap(s1))
    fmt.Println("s2: ",s2, len(s2),cap(s2))
    fmt.Println("s3: ",s3, len(s3),cap(s3))
    fmt.Println("s4: ",s4, len(s4),cap(s4))
    fmt.Println("s5: ",s5, len(s5),cap(s5))
    fmt.Println("s6: ",s6, len(s6),cap(s6))
}
输出：
s1:  [0 1 2 3 4 5 6 7 8 9] 10 10
s2:  [2 3 4] 3 8
s3:  [2 3 4] 3 5
s4:  [4 5 6 7 8 9] 6 6
s5:  [0 1 2 3] 4 10
s6:  [0 1 2 3] 4 6
```
cap 表示切片所引用数组片段的真实长度，len表示已经赋过值的最大下标(索引)值加1.

注意下面两种初始化方式的区别：

```go
{
    var a  []int
    b := []int{}
    fmt.Println(a==nil,b==nil)

    fmt.Printf("a: %#v\n", (*reflect.SliceHeader)(unsafe.Pointer(&a)))
    fmt.Printf("b: %#v\n", (*reflect.SliceHeader)(unsafe.Pointer(&b)))
    fmt.Printf("a size %d\n", unsafe.Sizeof(a))
    fmt.Printf("b size %d\n", unsafe.Sizeof(b))
}
输出：
true false
a: &reflect.SliceHeader{Data:0x0, Len:0, Cap:0}
b: &reflect.SliceHeader{Data:0x5168b0, Len:0, Cap:0}
a size 24
b size 24
说明：
1. 变量b的内部指针被赋值，即使该指针指向了runtime.zerobase，但它依然完成了初始化操作
2. 变量a表示一个未初始化的切片对象，切片本身依然会分配所需的内存
```

切片之间不支持逻辑运算符，仅能判断是否为nil，比如：

```go
{
    var a  []int
    b := []int{}
    fmt.Println(a==b) //invalid operation: a == b (slice can only be compared to nil)
}
```

- reslice

在原slice的基础上进行新建slice，新建的slice依旧指向原底层数组，新创建的slice不能超出原slice
的容量，但是不受其长度限制，并且如果修改新建slice的值，对所有关联的切片都有影响，比如:

```go
{
    s := []string{"a","b","c","d","e","f","g"}

    s1 := s[1:3]        //b,c
    fmt.Println(s1, len(s1),cap(s1))
    s1_1 := s1[2:5]        //c,d,e
    fmt.Println(s1_1, len(s1_1),cap(s1_1))
}
输出：
[b c] 2 6
[d e f] 3 4
```    
- append

向切片尾部追加数据，返回新的切片对象; 数据被追加到原底层数组，如果超出cap限制，则为新切片对象重新分配
数组，新分配的数组cap是原数组cap的2倍，比如：

```go
{
    s := make([]int,0,5)
    s = append(s , 1)
    s = append(s , 2,3,4,5)
    fmt.Printf("%p, %v, %d\n", s,s, cap(s))
                                                                                                                                                                                                                   
    s = append(s , 6)       //重新分配内存
    fmt.Printf("%p, %v, %d\n", s, s, cap(s))
}
输出：
0xc420010210, [1 2 3 4 5], 5
0xc4200140a0, [1 2 3 4 5 6], 10
```
如果是向nil切片追加数据，则会高频率的重新分配内存和数据复制，比如:

```go
{
    var s []int
    fmt.Printf("%p, %v, %d\n", s,s, cap(s))
    for i:= 0; i < 10;i++{
        s = append(s, i)
        fmt.Printf("%p, %v, %d\n", s, s, cap(s))
    }
}
```
所以为了避免程序运行中的频繁的资源开销，在某些场景下建议预留出足够多的空间。

- copy

两个slice之间复制数据时，允许指向同一个底层数组，并允许目标区间重叠。最终复制的长度以较短的切片长度(len)
为准，比如：

```go
{
    s1 := []int{0, 1, 2, 3, 4, 5 ,6}
    s2 := []int{7, 8 ,9}
    copy(s1,s2)
    fmt.Println(s1,len(s1),cap(s1))
    s1 = []int{0, 1, 2, 3, 4, 5 ,6}
    s2 = []int{7, 8 ,9}                                                                                                                                                                                            
    copy(s2,s1)
    fmt.Println(s2,len(s2),cap(s2))
}
```
可不可以在同一切片之间复制数据呢？

在日常开发过程中，如果slice长时间引用一个大数组中很小的片段，那么建议新建一个独立的切片，并复制出所需的
数据，以便原数组内存可以被gc及时释优化回收。

#### 字典(map)

```
初始化
基本操作
遍历
排序
```
-  字典初始化
map的声明格式如下： map[KeyType]ValueType

字典是无序键值对集合，字典要求KeyType必须是支持相等运算符(==,!=)的数据类型，比如数字、字符串、指针、数组、结构体，
以及对应的接口类型，ValueType可以是任意类型，字典也是引用类型，使用make函数或者初始化表达式语句来创建。

```go
{
    m := make(map[string]int)
    m["a"] = 1 
    m["b"] = 2 

    m2 := map[int]struct{   //匿名结构体
        x int 
    }{  
        1: {x:100},
        2: {x:200},
    }   
    fmt.Println(m,m2)
}
```
字典基本操作

```go
{
    m := make(map[string]int)
    m["route"] = 66 //添加key
    i := m["route"] //读取key
    j := m["root"]  //the value type is int, so the zero value is 0
    n := len(m) //获取长度
    //n := cap(m)   // ???
    delete(m, "route") //删除key
}
```
访问不存在的键值，不会引发错误，默认返回ValueType的零值，那能否根据零值来判断key的存在呢 ？

在读取map操作时， 可以使用双返回值来正常读取map，比如：

```go
{
    j, ok := m["root"]
}
```
ok是个bool类型变量，如果key真实存在，则ok的值为true，反之为false，如果你只是想测试key是否存在而不
获取值的话，可以使用忽略字符"_"。

在修改map值操作时，因为受内存访问安全和哈希算法等缘故，字典被设计成"not adrressable"，因此不能直接修改
value成员(结构体或数组)。

```go
{
    m := map[int] user {
        1 : {
            name:"tom",
            age:19},
    }
    m[1].age = 1        //cannot assign to struct field m[1].age in map                                                                                                                                                                             
}
```
有两种改造方式：
> 一是对整个value进行重新复制
> 二是声明map时valueType为指针类型

```go
{
    m := map[int] user {
        1 : {
            name:"tom",
            age:19},
    }
    u := m[1]
    u.age = 1
    m[1] = u
}
{
    m := map[int] *user {
        1 : {
            name:"tom",
            age:19},
    }
    m[1].age += 1
}
```

不能对nil字典进行写操作，但是可以读，比如：

```go
{
    var m map[string]int
    //p := m["a"]       //ok
    m["a"] = 1          //panic: assignment to entry in nil map
}
```

map遍历

```go
{
//  var m = make(map[string]int)
    var m = map[string]int{}
    m["route"] = 66
    m["root"] = 67
    for key,value := range m{
        fmt.Println("Key:", key, "Value:", value)
    }
}
```
因为map是无序的，如果想按照有序key输出的话，可以先把所有的key取出，然后对key进行排序，再遍历map，比如:

```go
{
    m := make(map[int]int)
    var keys []int
    for i := 0 ;i <= 5;i++{
        m[i] = i
    }
                                                                                                                                                      
    for k, v := range m{
        fmt.Println("Key:",k,"Value:",v)
    }   
    for k := range m{
        keys = append(keys, k)
    }   
    sort.Ints(keys)
    for _, k := range keys{
        fmt.Println("Key:",k,"Value:",m[k])
    }
}
```

并发
字典不是并发安全的数据结构，如果某个任务正在对字典进行写操作，那么其他任务就不能对该字典执行并发操作(读、写、删除)，
否则会导致程序崩溃，比如:

```go
{
    m := make(map[string]int)
    go func(){
        for {
            m["a"] += 1
            time.Sleep(time.Microsecond)
        }   
    }() 

    go func (){ 
        for{
            _ = m["b"]
            time.Sleep(time.Microsecond)
        }   
    }() 
    select{}
}
输出：
fatal error: concurrent map read and map write
```
go语言编译器提供了这种问题(竞争)的检测方式，比如：

```bash
# go run -race file.go
```
安全

可以使用 sync.RWMutex 实现同步，避免并发环境多goroutings同时读写操作，继续完善上面的例子，比如：

```go
{
    var lock = new(sync.RWMutex)                                                                                                                      
    m := make(map[string]int)
    go func(){
        for {
            lock.Lock()
            m["a"]++
            lock.Unlock()
            time.Sleep(time.Microsecond)
        }   
    }() 

    go func (){ 
        for{
            lock.RLock()
            _ = m["b"]
            lock.RUnlock()
            time.Sleep(time.Microsecond)
        }   
    }() 
    select{}
}
```
性能

在创建字典时预先准备足够的空间有助于提升性能，减少扩张时内存动态分配和重复哈希操作，比如：

```go
package main

import "testing"
import "fmt"

func test() map[int]int {
	m := make(map[int]int)
	for i:=0; i < 1000; i++{
		m[i] = 1
	}
	return m
}

func testCap() map[int]int{
	m := make(map[int]int,1000)
	for i:=0; i < 1000; i++{
		m[i] = 1
	}
	return m
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
# go run conmap.go
BenchmarkTest 	 10000, 160203 ns/op,98 allocs/op, 89556 B/op
BenchmarkTestCap 20000, 65478 ns/op,12 allocs/op, 41825 B/op
```

#### 结构体(struct)
结构体由一系列被称为字段的命名元素组成。每个字段由名称和类型组成，字段名称可以被显示指定，也可以是匿名的。
声明一个结构体类型时，字段名称必须唯一，可使用"_"补位，支持使用自身指针类型成员。
字段名和排列顺序都属于结构体类型组成的部分，编译器会对结构体中的字段做对齐优化，比如：

```go
type Person struct{
    name string
    age int
    _ int
} 
```
初始化

```
可按顺序初始化全部字段
使用命名方式初始化指定字段
```

```go
type Person struct{
    name string
    age int                                                                                                                                           
}

{
    p := Person{
        "Tom",
        20,
    }
    /*              //初始化排列顺序不能错乱
    p := Person{
        20,                                                                                                                                           
        "Tom",
    }   
    */                  
    /*
    p := Person{
        "Tom"}         //too few values in struct initializer
    */
    p1 := Person{
        name: "Tom",
        age: 20,
    }
    p2 := Person{
        name: "unknown",                                                                                                                              
    }
}
```
因为命名初始化不受结构体扩展和字段顺序变化的影响，在日常开发中，建议使用命名初始化。

可以在函数体中直接定义匿名结构类型变量或用作字段类型，比如：

```go
{
    u := struct{        //直接定义匿名结构体变量
        name string
        age int 
    }{  
        "Tom",
        20, 
    }
    type file struct{
        name string
        attr struct{        //定义匿名结构类型字段
            size int 
            perm int 
        }
    }
    f := file{
        name: "passwd",
        attr:{          //missing type in composite literal
            size: 1,                                                                                                                                  
            perm: 1,
        },  
    }
    f.attr.size = 1     //ok
    f.attr.perm = 1
}
```

结构体比较

只有在所有字段类型全部支持相等运算符时，才可以做相等操作，比如:

```go

type data struct{
    x int 
    //y map[int]int
    y []int
}                                                                                                                                                     

func main(){
    d1 := data{x:1}
    d2 := data{x:2}
    fmt.Println(d1 == d2) //(struct containing []int cannot be compared)
}
```

可以使用结构体指针直接操作结构体，但是不能是多级指针，比如：

```go
{
    type Person struct{
        name string
        age int
    }

    p := &Person{
        "Li",
        20,
    }
    fmt.Println(p)
    p.name = "Tom"
    p.age = 10
    fmt.Println(p)
    p2 := &p
    *p2.name = "Tom2"   //p2.name undefined (type **Person has no field or method name)
    //(*p2).name = "Tom2"
    fmt.Println(p2)
}
```
- 空结构体

空结构体(struct{}) 是指没有字段的结构体类型，无论是空结构体还是空结构体数组，其长度都为0,比如：

```go
{
    var a  struct{}
    var b  [100]struct{}                                                                                                                              
    println(unsafe.Sizeof(a), unsafe.Sizeof(b))
}
输出：
0 0
```
但这并不影响对元素的操作，切片的内置子切片、长度和容量等属性依旧可以工作比如：

```go
{
    var b  [100]struct{}
    s := b[:]           //slice
    s[0] = struct{}{}
    s[1] = struct{}{}
    s[2] = struct{}{}
    fmt.Println(s[3], len(s),cap(s))
}
输出：
{} 100 100
```
空结构体可以像其他结构体一样正常使用，空结构体也具有正常结构体的属性，但就使用来说，一般用在通道元素类型做事件通知，
比如:

```go
func hello(name string, done chan struct{}) {
    fmt.Println("hello ",name)
    done <- struct{}{}
}

func main() {
    done := make(chan struct{})
    langs := []string{"Go", "C", "C++", "Java", "Perl", "Python"}
    for _, l := range langs {
        go hello(l, done)
    }   

    for _ = range langs {
        <-done
    }   
                                                                                                                                                      
}
```

- 匿名字段

匿名字段就是只有类型没有名字的字段(anonymous field)，也称作嵌入类型或嵌入字段，比如:

```go
type attr struct{
    name string
    age int 
}
type Person struct{
    id int 
    attr            //结构体嵌入
}

func main(){
    p := Person{
        id: 20, 
        attr:attr{      //显示初始化
            name: "Tom",
            age:    20, 
        },
    }   
    p.name = "kite"     //可以直接读写匿名字段成员   
    p.attr.age = 21     //防止嵌入结构体成员与结构体成员重名                                                                                                                            
}
```
除接口指针和多级指针以外的任何命名类型都可以作为匿名字段，比如：

```go
type data struct{
    *int
    //int               //duplicate field int
    string
}

func initData(){
    x := 100
    d := data{
        int: &x,            //使用基本类型作为字段名
        string: "Tom",
    }
    fmt.Printf("%#v\n", d)                                                                                                                            
}
```
- 字段标签

Go语言结构体提供了在运行时，通过反射机制获取字段描述的元数据信息，尽管它不属于数据成员，
但却是类型的组成的部分，日常开发中，常被用做格式校验和数据库关系映射等。

```go
type attr struct{
    name string `姓名`
    age int     `年龄`
}
type Person struct{
    id int `身份证号码`
    attr    `属性`
}
func main(){
    p := Person{
        id: 20,
        attr:attr{
            name: "Tom",
            age:    20,
        },
    }
    v := reflect.ValueOf(p)
    t := v.Type()

    for i,n := 0,t.NumField(); i < n; i++{
        if v.Field(i).Kind().String() == "struct"{
            v1 := v.Field(i)
            t1 := v1.Type()
            for j, m := 0, t1.NumField(); j < m ; j++{
                fmt.Printf("%s:  %v\n",t1.Field(j).Tag, v1.Field(j))
            }
        }else{
            fmt.Printf("%s:  %v\n",t.Field(i).Tag,v.Field(i).Kind())
        }
    }
}
输出：
身份证号码:  int
姓名:  Tom
年龄:  20
```
- 内存布局

不管结构体含有多少个字段，其内存总是一次性分配的，各字段在相邻的地址空间按定义顺序排列，编译器通常
会对其做对齐处理，对齐通常以所有字段中最长基础类型宽度为准。
对于字段是引用类型、字符串和指针的，结构内存中只包含基本数据，比如：

```go
type Point struct{
    x int
    y int
}
type Value struct{
    id int
    name string
    data []byte
    next *Value
    Point
}

func main(){
    v := Value{
        id: 1,
        name: "Tom",
        data:[]byte{1,2,3,4},
        next:nil,
        Point:Point{
            x: 100,y:200,
        },
    }
    fmt.Printf("size: %d,align: %d\n", unsafe.Sizeof(v), unsafe.Alignof(v))
    fmt.Printf("field\taddress\t\toffset\tsize\n")
    fmt.Printf("id\t%p\t%d\t%d\n", &v.id, unsafe.Offsetof(v.id), unsafe.Sizeof(v.id))
    fmt.Printf("name\t%p\t%d\t%d\n", &v.name, unsafe.Offsetof(v.name), unsafe.Sizeof(v.name))
    fmt.Printf("data\t%p\t%d\t%d\n", &v.data, unsafe.Offsetof(v.data), unsafe.Sizeof(v.data))
    fmt.Printf("next\t%p\t%d\t%d\n", &v.next, unsafe.Offsetof(v.next), unsafe.Sizeof(v.next))
    fmt.Printf("x\t%p\t%d\t%d\n", &v.Point.x, unsafe.Offsetof(v.Point.x), unsafe.Sizeof(v.Point.x))
    fmt.Printf("y\t%p\t%d\t%d\n", &v.Point.y, unsafe.Offsetof(v.Point.y), unsafe.Sizeof(v.Point.y))
}
输出：
size: 72,align: 8
field	address		offset	size
id	    0xc4200140a0	0	8
name	0xc4200140a8	8	16
data	0xc4200140b8	24	24
next	0xc4200140d0	48	8
x	    0xc4200140d8	0	8
y	    0xc4200140e0	8	8

抽象布局：

    |-------16--------|------------24------------|    |------16-------| 
+---+--------+--------+--------+--------+--------+----+-------+-------+
|id |name.ptr|name.len|data.ptr|data.len|data.cap|next|point.x|point.y| 
+---+--------+--------+--------+--------+--------+----+-------+-------+
0   8        16       24       32       40       48   56      64      72
```
练习下面的列子

```go
{   
    v1 := struct{
        a byte
        s string
        b byte
    }{}
    fmt.Printf("size: %d,align: %d\n", unsafe.Sizeof(v1), unsafe.Alignof(v1))
    fmt.Printf("a\t%p\t%d\t%d\n", &v1.a, unsafe.Offsetof(v1.a), unsafe.Sizeof(v1.a))
    fmt.Printf("s\t%p\t%d\t%d\n", &v1.s, unsafe.Offsetof(v1.s), unsafe.Sizeof(v1.s))
    fmt.Printf("b\t%p\t%d\t%d\n", &v1.b, unsafe.Offsetof(v1.b), unsafe.Sizeof(v1.b))

    v2 := struct{
        a byte
        b []int
        s byte
    }{}
    fmt.Printf("size: %d,align: %d\n", unsafe.Sizeof(v2), unsafe.Alignof(v2))
    fmt.Printf("a\t%p\t%d\t%d\n", &v2.a, unsafe.Offsetof(v2.a), unsafe.Sizeof(v2.a))
    fmt.Printf("b\t%p\t%d\t%d\n", &v2.b, unsafe.Offsetof(v2.b), unsafe.Sizeof(v2.b))
    fmt.Printf("s\t%p\t%d\t%d\n", &v2.s, unsafe.Offsetof(v2.s), unsafe.Sizeof(v2.s))

    v3 := struct{
        a struct{}
        b int
        c struct{}
    }{}
    fmt.Printf("size: %d,align: %d\n", unsafe.Sizeof(v3), unsafe.Alignof(v3))
    fmt.Printf("a\t%p\t%d\t%d\n", &v3.a, unsafe.Offsetof(v3.a), unsafe.Sizeof(v3.a))
    fmt.Printf("b\t%p\t%d\t%d\n", &v3.b, unsafe.Offsetof(v3.b), unsafe.Sizeof(v3.b))
    fmt.Printf("c\t%p\t%d\t%d\n", &v3.c, unsafe.Offsetof(v3.c), unsafe.Sizeof(v3.c))

    v4 := struct{
        a struct{}
    }{}
    fmt.Printf("size: %d,align: %d\n", unsafe.Sizeof(v4), unsafe.Alignof(v4))
    fmt.Printf("a\t%p\t%d\t%d\n", &v4.a, unsafe.Offsetof(v4.a), unsafe.Sizeof(v4.a))
}
```
